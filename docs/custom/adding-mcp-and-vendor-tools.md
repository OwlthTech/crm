# Adding vendor tools (ClickUp example) and MCP

This repo does not ship an MCP server and does not load MCP tools into
eve. `docs/plan/dynamic-fields-build.md` says to keep logic out of tools
so an MCP wrapper can come later. Until then, a vendor is a first-party
eve tool behind a capability.

MCP is a transport. The agent still needs a typed tool, a capability
gate, and — for team agents — a manifest action. Do not hand the model
a raw MCP session. The sandbox is `deny-all`. Customer text must not
leave in a third-party query.

Read `docs/agent.md`, `docs/connections.md`, `adding-tools.md`, and
`adding-capabilities.md` first.

## Choose the runtime

| When it runs | Where the tool lives |
| --- | --- |
| A rep asks, or a queued research task needs it | Root `apps/agent/agent/tools/` |
| A deployed team agent must do it every run | Runner tool + manifest action |

On-demand ClickUp from the record sheet is a root tool. "When a deal
closes, create a ClickUp task" is a team-agent action. Many vendors need
both. Build the lib function once. Point both tools at it.

## Worked example: create a ClickUp task

Replace names if the vendor is not ClickUp. The seams stay the same.

### 1. Capability and secret

`.env.example`:

```
# ClickUp — create tasks from the agent. Without it the ClickUp tools
# return unavailable and the rest of the agent keeps working.
# CLICKUP_API_TOKEN=""
# CLICKUP_LIST_ID=""
```

Add `CLICKUP_API_TOKEN` and `CLICKUP_LIST_ID` to:

- root `turbo.json` `globalPassThroughEnv`
- `apps/agent/turbo.json` `passThroughEnv`

Do not add them to `env.validation.ts` unless the API reads them. The
API must not read them.

In `apps/agent/agent/lib/capabilities.ts` add a row:

- `id`: `CLICKUP_API_TOKEN`
- `label`: `ClickUp`
- `gives`: `create and read tasks in an approved list`
- `from`: `CLICKUP_API_TOKEN`

A missing token returns `enabled: false`. Boot log prints `off ClickUp`.

### 2. Vendor client in the agent

Add `apps/agent/agent/lib/clickup.ts`. Parse the ClickUp JSON with Zod
the moment it enters the process. Use `@crm/db/safe-fetch` (or the same
SSRF-safe wrapper the portrait code uses). Never log the token.

The function takes domain types (`listId`, `title`, `description`). It
does not take `Record<string, unknown>`.

### 3. Root tool (on demand)

`apps/agent/agent/tools/create_clickup_task.ts`:

- Gate with `enabled("CLICKUP_API_TOKEN")` and `unavailable(...)`.
- Charge `spend(1)` if the call is metered. If ClickUp is unmetered in
  this install, still keep a tight input schema.
- Description must say this is not identity research.
- Do not paste mailbox bodies into the task description. Derived
  summary only. `data-boundaries.md` still applies.

### 4. Team-agent action (on event or schedule)

This is the "when a deal closes, create a ClickUp task" path.

1. Add `CLICKUP_TASK_CREATE: "clickup.task.create"` to
   `AGENT_ACTION_TYPES` in `packages/validation/src/agent-manifest.ts`.
2. Add a Zod variant. Include a **chosen** destination (list id +
   label), not a dest the model invents at run time.
3. Map it in `AGENT_ACTION_EXECUTORS` to `create_clickup_task`.
4. Map it in `AGENT_ACTION_DEPENDENCIES` to
   `{ id: "clickup", label: "ClickUp", resourceId: "clickup:workspace", fix: "Set CLICKUP_API_TOKEN." }`.
5. Duplicate the action variant in
   `apps/agent/agent/subagents/agent_builder/lib/draft-input.ts`.
6. Allow `clickup` in the draft `integrations` enum and in `INTEGRATIONS`.
7. Validate the tag in `builder-runtime.ts` the same way Slack is
   validated.
8. Implement `createRunClickupTask` in `run-runtime.ts`. Lock on an
   idempotency key. Log the action row before the HTTP call. Retry must
   not create a second task.
9. Add
   `apps/agent/agent/subagents/agent_runner/tools/create_clickup_task.ts`
   that calls that function with `runId` and `ctx.callId`.

The runner tool input is the task text. The list id comes from the
manifest. The model does not pick a list.

### 5. Optional connection page

Only add Settings → Connections → ClickUp when a person must OAuth or
pick a default list in the UI. A personal token in `.env` is enough for
a single-tenant self-host. If you add the page:

- Brings in: nothing (unless you import tasks).
- Sends: tasks to a chosen list.
- Owner or admin connects and disconnects.
- `builderResources()` returns `clickup:workspace` when the token
  works.

Do not put a trigger builder on that page.

### 6. If you already run a ClickUp MCP server

Keep MCP outside the sandbox. Wrap each MCP tool you want in
`defineTool` and call the MCP server from `lib/clickup.ts` over stdio or
HTTP you control.

Do not:

- Register the MCP server as an eve connection without a typed schema.
- Forward the model's raw arguments to MCP.
- Give the sandbox egress so it can reach ClickUp itself.
- Send thread bodies, meeting notes, or tokens to the MCP process.

The tool schema is the contract. MCP is an implementation detail of
`lib/clickup.ts`.

## On-demand vs event

| Intent | Mechanism |
| --- | --- |
| Rep types "create a ClickUp task for this deal" | Root tool, current record in `focus` |
| Every `deal.closed` | Team agent, `EVENT` trigger `deal.closed`, action `clickup.task.create` |
| Every morning | Team agent, `SCHEDULE` trigger |

The API still only writes `AgentTask` / `agent-event`. The agent worker
runs the tool.

## Checklist

1. Secret is optional and listed in `.env.example` and Turbo pass-through.
2. `capabilities.ts` knows the id.
3. Zod at the HTTP boundary.
4. One lib module, thin tools.
5. Manifest + runner path if a deployed agent must send.
6. Idempotent writes.
7. `GET /eve/v1/info` lists the new tools.
8. A test covers missing token, happy path, and retry.

## Do not

- Put the ClickUp client in Nest.
- Create a task twice on dispatch retry.
- Let the model choose an arbitrary list at run time.
- Call this done when only the catalogue row exists.
