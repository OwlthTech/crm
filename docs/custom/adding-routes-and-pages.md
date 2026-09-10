# Adding a route and a page

This product has three HTTP surfaces. Pick the right one before you add a
file.

| Surface | Lives in | Serves |
| --- | --- | --- |
| App pages | `apps/app` | The signed-in UI |
| API (tRPC) | `apps/api` | Data the UI calls |
| Agent internals | `apps/agent/agent/channels/crm.ts` | Dispatch and probes the API pokes |

Do not put a vendor client in the API. Do not put a CRM write in a Next.js
route handler. The page loads data. The service writes data. The agent
decides.

Read `docs/design.md` and `docs/api.md` first.

## 1. App page (Next.js)

Pages live under `apps/app/app`. Signed-in CRM pages live under
`app/(app)/[slug]/`. The slug is cosmetic. Every query still uses
`WORKSPACE_ID`.

Copy a small existing page. Contacts is the list pattern.
`settings/currencies/page.tsx` is the settings pattern.

### Checklist

1. Add `page.tsx` under `apps/app/app/(app)/[slug]/…`.
2. Export `metadata.title`.
3. Wrap the screen in `PageShell` from `apps/app/components/page-shell.tsx`.
4. Call `requireSession()` (or `requireMailboxAccess()`) in the server
   loader, not in a client file.
5. Prefetch tRPC on the server with `getServerTrpc()` and
   `getServerQueryClient()`. Return the interactive tree inside
   `HydrateClient`.
6. Put interactive UI in a `"use client"` file next to the page. That file
   owns its prop types. It does not import `@crm/auth` or `@crm/db`.
7. Use components from `@crm/ui`. Do not invent a local variant.

### Navigation

A page with no link is a dead route. Update every place that names sections:

| What | File |
| --- | --- |
| Icon rail | `apps/app/components/app-icon-rail.tsx` `ITEMS` |
| Settings sidebar | `apps/app/app/(app)/[slug]/settings/settings-sidebar.tsx` `ITEMS` |
| Hover prefetch | `apps/app/components/crm/section-prefetch.ts` |
| Workspace slug reserved words | `packages/db/src/workspace.ts` `RESERVED_SLUGS` |
| Proxy section rewrite | `apps/app/proxy.ts` `SECTIONS` |

Add the first path segment to `RESERVED_SLUGS`. A workspace named "Reports"
must not steal `/reports`.

Add the section to `proxy.ts` `SECTIONS` when the unsigned path `/foo` must
rewrite to `/{slug}/foo` the same way `/contacts` does.

`/sign-in`, `/grant-access`, and `/eve` stay ungated. Do not add a CRM page
to that list.

### Search params

List pages that filter in the URL use nuqs. See
`apps/app/app/(app)/[slug]/contacts/contacts-search-params.ts`. Read the
`nuqs` skill before you copy that file.

## 2. tRPC route (API)

tRPC is the data surface. REST is auth, health, and internal crons only.

One module owns one alias. Copy `apps/api/src/fields/`:

```
apps/api/src/<area>/
  <area>.module.ts
  <area>.router.ts
  <area>.service.ts
  <area>.contracts.ts
```

### Checklist

1. Zod schemas in `*.contracts.ts`. Infer types with `z.infer`. Do not
   hand-write an interface beside the schema.
2. `@Router({ alias: "<area>" })` and `@UseMiddlewares(AuthMiddleware)` on
   the class. No `AuthMiddleware` means the procedure is public.
3. Keep the router thin: parse input, call the service, return the result.
4. Prisma stays in the service. Throw Nest `HttpException` types.
   `DomainErrorMiddleware` maps them.
5. List procedures take a list input and return
   `{ rows, total, facetCounts }`. Filter in Prisma.
6. Register the module in `apps/api/src/app.module.ts`.
7. List the router in the module `providers`. Runtime DI and the tRPC CLI
   fail independently. Forgetting either one hides the procedure.
8. Add `restMeta(...)` so `/openapi.json` and `/rest` stay in sync.
9. Run the generator during `dev` / `check-types`. Commit
   `apps/api/src/generated/server.ts`. The Vercel build must not regenerate
   it.

The client calls `trpc.<alias>.<method>`. After a new procedure, restart
`dev` if the app cannot see it.

Intelligence still does not live here. A new contact queues work with
`AgentTriggerService`. It does not call ClickUp, Perplexity, or Context.dev.

## 3. Agent HTTP route

The agent process is a separate deployment. The browser never talks to it
with a long-lived secret.

Internal routes live in `apps/agent/agent/channels/crm.ts`:

```
GET  /internal/crm/dispatch-health
POST /internal/crm/dispatch
POST /internal/crm/builder-dispatch
POST /internal/crm/agent-dispatch
POST /internal/crm/cancel-run
POST /internal/crm/slack/create-channel
POST /internal/crm/verify-key
```

Every route checks `AGENT_BRIDGE_SECRET` with `authorised()`. Unset secret
refuses the request. It does not open the route.

Add a route only when the API must poke the agent (dispatch, cancel, probe).
A person-facing feature belongs in tRPC plus an `AgentTask` row.

Parse the body with Zod at the handler. Return JSON errors with a real
status. Do not swallow a bad payload into a silent 202.

`GET /eve/v1/info` is eve's inventory. Use it after you add a channel route
to confirm eve loaded the file.

## 4. Nest REST controller

Use a controller only for health, auth, mailbox sync, tracking, or a cron
the platform must hit. See `apps/api/src/health/health.controller.ts`.

Register it on the module `controllers` array. Add `@AllowAnonymous()` only
when the route must work without a session, and fail closed on a missing
secret (`CRON_SECRET` for sync).

## Do not

- Import `@crm/db` or `@crm/auth` from a `"use client"` file.
- Filter a whole table in the browser.
- Add a per-package `.env`.
- Put an "Edit automation" form on a settings page.
- Call a vendor from `apps/api` "just this once".
