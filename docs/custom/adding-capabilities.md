# Adding capabilities agents can use

A capability is something the install can do when a key or a connection is
present. A missing key turns it off. It never throws.

Two layers exist. Keep them distinct.

| Layer | Meaning | Registry |
| --- | --- | --- |
| Optional source | A vendor the research agent may query | `apps/agent/agent/lib/capabilities.ts` |
| Connection / action | A workspace integration a team agent may tag or send through | Settings → Connections, then the manifest |

Read `docs/connections.md` and `docs/environment.md` first.

## Optional sources (research agent)

`capabilities()` is the single list the boot log and the session markdown
read. Today it knows:

- `PERPLEXITY_API_KEY` — web research
- `CONTEXT_DEV` / `CONTEXT_DEV_PEOPLE` — brand and LinkedIn (Settings key)
- `BLOB_READ_WRITE_TOKEN` — picture storage

`GITHUB_TOKEN` is documented in `.env.example` but is not in this list.
Add it here if you want the boot log to tell the truth.

### Checklist for an env-backed source

1. Add the variable to `.env.example` with a note.
2. Add it to root `turbo.json` `globalPassThroughEnv`.
3. Add it to `apps/agent/turbo.json` `passThroughEnv` so `eve dev` sees it.
4. If the API reads it, declare it in
   `apps/api/src/config/env.validation.ts` and `apps/api/turbo.json`.
   Most agent-only keys must not go there.
5. Add a row in `capabilitiesFrom()` with `id`, `label`, `gives`, `from`.
6. In the tool, `if (!(await enabled(id))) return unavailable(id)`.
7. Never throw when the key is missing.

The Context.dev key is not an env var. It lives in `AppSetting`. Admins
set it at `/onboarding/research` or Settings → General.
`readContextDevKey` in `@crm/db/settings` is the only reader. Copy that
pattern when a self-hoster must change a key without a redeploy.

## Connections (workspace integrations)

A connection page shows what it brings in and what it sends. It never
authors an automation.

Live connections today: Google Workspace, Slack, Microsoft 365. Stripe,
Docusign, HubSpot, Ergo, and the intake API are catalogue rows only.

### Checklist for a live connection

1. Decide brings-in and sends. Use those two words on the page.
2. Add the settings page under
   `apps/app/app/(app)/[slug]/settings/connections/<name>/`.
3. Add a row to `add-connection-dialog.tsx`.
4. Add OAuth (or a token form) in `packages/auth`. Request scopes at
   connect time. Store tokens on `Account` or a dedicated grant table.
5. Disconnect is an owner or admin decision (`canManageConnections`).
   Additive actions stay open to any member.
6. Give it a builder resource id, shaped `vendor:resource`
   (`google:gmail`, `slack:workspace`).
7. Teach `conversations.service.ts` `builderResources()` to return that
   id when the connection is live.
8. Add the id to `CAPABILITY_RESOURCE_IDS` in
   `packages/validation/src/agents.ts`.
9. Add the short name to the builder draft `integrations` enum in
   `draft-input.ts`.
10. Validate the tag in `builder-runtime.ts`. Unknown ids are an error.
11. If the runner must send through it, add a manifest action. See
    `adding-tools.md` and `adding-mcp-and-vendor-tools.md`.

Microsoft mail is connected for sync and is not a builder integration.
Do not assume a live OAuth connection is taggable.

## Team-agent actions

Actions a version may declare live in
`packages/validation/src/agent-manifest.ts`:

- `crm.activity.create`
- `run.summary`
- `slack.message.post`

Slack depends on `slack:workspace`. CRM actions depend on nothing
outside the database.

A new send path is a new action type plus a runner tool. Empty
`Sends` on the connection page is a guarantee. Do not silently add a
write.

## Triggers are not capabilities

Triggers (`MANUAL`, `SCHEDULE`, `EVENT`) already exist. New CRM events
go in `packages/db/src/crm-events.ts` and the API writes an
`agent-event` task. Do not invent a second event bus in Nest.

## Do not

- Throw at boot because a self-hoster has no ClickUp token.
- Store a second copy of a secret in a per-package `.env`.
- Put a vendor client in `apps/api`.
- Treat "Coming soon" catalogue rows as wired integrations.
- Infer workspace-wide access from an empty resource list.
