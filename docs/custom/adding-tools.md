# Adding tools agents can use

eve loads every `tools/*.ts` file in an agent directory. The filename is
the tool name. `defineTool` exports the default.

Three agents have three tool lists. They inherit nothing from each other.

| Agent | Directory | Job |
| --- | --- | --- |
| Research (root) | `apps/agent/agent/tools/` | Identify people, fill facts, write briefs |
| Builder | `apps/agent/agent/subagents/agent_builder/tools/` | Draft a READY version |
| Runner | `apps/agent/agent/subagents/agent_runner/tools/` | Execute one pinned version |

To hide an inherited eve tool, export `disableTool()` from a file with that
name. The root `agent.ts` tool does this. Builder disables `bash`, `grep`,
`glob`, `web_search`, and others the same way.

Read `docs/agent.md` first. Read the eve tools guide in
`apps/agent/node_modules/eve/docs`.

## Tool shape

Copy `apps/agent/agent/tools/research_person.ts`:

```ts
import { defineTool } from "eve/tools";
import { z } from "zod";
import { enabled, unavailable } from "../lib/capabilities";
import { spend } from "../lib/focus";

export default defineTool({
	description:
		"Research a person or company on the open web for sales context — recent news, funding, launches, public statements. Returns cited claims. NOT a source of truth for someone's identity or job title; use get_linkedin_profile for that.",
	inputSchema: z.object({
		question: z.string().describe("A specific question."),
		deep: z.boolean().default(false),
	}),
	async execute({ question, deep }) {
		if (!(await enabled("PERPLEXITY_API_KEY"))) {
			return unavailable("PERPLEXITY_API_KEY");
		}

		const charge = spend(deep ? 2 : 1);
		if (!charge.ok) return { ok: false as const, reason: charge.reason };

		return { ok: true as const, answer: "…" };
	},
});
```

Rules:

- Default export only. No comments in the file.
- `description` tells the model when to call it and when not to.
- `inputSchema` is Zod. Parse at this boundary. Do not take
  `Record<string, unknown>`.
- Return a typed object. On a missing capability return `unavailable(id)`.
  That is not a failure. Retrying will not help.
- Vendor calls go through `spend()`. One unit is one metered call. Brand
  lookup and person enrich each charge 2.
- Put HTTP, signing, and retries in `apps/agent/agent/lib/`. The tool file
  stays thin.
- Never log customer text, tokens, or mailbox bodies.

## Research tools

Use the root folder when a rep on a record sheet, or a queued `AgentTask`,
must read or write CRM research.

Existing patterns to copy:

| Need | Copy |
| --- | --- |
| Free CRM read | `read_crm_history.ts` |
| Optional vendor | `research_person.ts` |
| Fact write | `record_fact.ts` (goes through `lib/facts.ts`) |
| Custom field | `set_field_value.ts` |

Do not add a confidence input. Tools report what they observed.
`lib/evidence.ts` prices it.

Do not overwrite a human. Do not re-offer a dismissed value. Do not write
a fact without a primary evidence kind.

## Runner tools (team agents)

A runner tool may only do what the pinned version declared. The destination
comes from the manifest, not from the model.

Copy `apps/agent/agent/subagents/agent_runner/tools/post_slack_message.ts`.
The execute function reads `runId` from the session and calls
`apps/agent/agent/lib/run-runtime.ts`.

A new side effect needs all of these, not only the tool file:

1. A literal in `AGENT_ACTION_TYPES` (`packages/validation/src/agent-manifest.ts`).
2. A Zod variant on `agentManifestAction`.
3. An executor name in `AGENT_ACTION_EXECUTORS` (`apps/agent/agent/lib/agent-actions.ts`).
4. A dependency row in `AGENT_ACTION_DEPENDENCIES` when a connection is
   required.
5. The same action variant in
   `apps/agent/agent/subagents/agent_builder/lib/draft-input.ts`.
6. Validation in `apps/agent/agent/lib/builder-runtime.ts`.
7. The runtime function in `run-runtime.ts`. Make it idempotent. Log the
   action before it executes.
8. The runner tool that calls that function.

Empty scope never means all records. Do not infer a grant from prose.

## Builder tools

The builder drafts. It never deploys. It never posts to Slack. It never
creates a CRM activity.

Add a builder tool only when the draft needs a new read (context, catalog,
schema). Side effects belong on the runner.

## Checklist

1. Choose the agent directory.
2. Name the file after the verb the model should say.
3. Gate optional vendors with `enabled()` / `unavailable()`.
4. Charge `spend()` for metered calls.
5. Keep writes in `lib/facts.ts`, `run-runtime.ts`, or an equivalent
   single write path.
6. Restart the agent. Check `GET /eve/v1/info`.
7. Add an integration test next to the existing agent tests when the tool
   writes.

## Do not

- Give the sandbox network. `sandbox.ts` is `deny-all` on purpose.
- Put mailbox text into `/workspace`.
- Put customer text in a third-party query.
- Call Slack for a live channel list from the builder. Read the cache.
- Add the same tool to root, builder, and runner "so it is available".
  Each list is a permission boundary.
