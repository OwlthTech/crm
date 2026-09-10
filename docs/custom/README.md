# Custom developer guides

These guides are for this fork. They tell a developer how to extend the
product without breaking the rules in `AGENTS.md`.

Read the area doc first. Then read the matching guide here.

| Work | Read first | Then this |
| --- | --- | --- |
| A Next.js page or a tRPC procedure | `docs/design.md`, `docs/api.md` | `adding-routes-and-pages.md` |
| An eve skill the research agent loads | `docs/agent.md` | `adding-skills.md` |
| A tool the research agent or a team agent calls | `docs/agent.md` | `adding-tools.md` |
| An optional source, connection, or team-agent action | `docs/connections.md`, `docs/environment.md` | `adding-capabilities.md` |
| A vendor such as ClickUp, including MCP | `docs/agent.md`, `docs/connections.md` | `adding-mcp-and-vendor-tools.md` |
| A local or self-hosted model | `docs/agent.md`, `docs/environment.md` | `local-ai.md` |

## The rules that every change here must keep

- Intelligence lives in `apps/agent`. The API writes an `AgentTask` row. It
  never calls a vendor, scores a person, or matches identity.
- A connection is a capability. An automation is an agent. Settings never
  author an automation.
- A missing key removes a capability. It never throws.
  `apps/agent/agent/lib/capabilities.ts` is the list.
- Parse untyped data at the boundary with Zod. Shapes that cross a package
  live in `packages/validation/src`.
- One `.env` at the repo root. Add every new variable to `.env.example` and
  to `turbo.json` `globalPassThroughEnv`.
- Never add code comments.
- Shared UI lives in `packages/ui`. A client file never imports `@crm/auth`
  or `@crm/db`.

eve's own docs ship in `apps/agent/node_modules/eve/docs` once the package
is installed. Read the matching eve guide before you write eve code.
