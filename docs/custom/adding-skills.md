# Adding skills

There are two skill systems. They are not interchangeable.

| Kind | Who reads it | Where |
| --- | --- | --- |
| eve skill | The running CRM agent | `apps/agent/agent/skills/*.md` |
| Repo coding skill | Coding agents that edit this repo | `.agents/skills/<name>/SKILL.md` |

This guide covers eve skills. Those are the files that change how the
research agent behaves at runtime. For a coding skill, read
`.agents/skills/skill-creator/SKILL.md`.

Read `docs/agent.md` first. Read the eve skills guide in
`apps/agent/node_modules/eve/docs` after `bun install`.

## What an eve skill is

A skill is markdown the model loads when the turn matches its
`description`. It is not a tool. It cannot call an API. It tells the model
how to use tools it already has.

Existing skills:

- `evidence.md` — how to pick an evidence kind
- `identity-matching.md` — how to refuse a wrong person
- `data-boundaries.md` — what may be read, what may leave
- `writing-a-brief.md` — Background panel shape

Subagents do not inherit root skills. `agent_builder` and `agent_runner`
start empty. Put a skill next to that subagent only when that subagent
must follow it.

## File shape

One file per skill, kebab-case, under `apps/agent/agent/skills/`:

```md
---
description: Use when recording a fact — picking the right evidence kind for what you actually saw, and understanding why a claim was written, offered or held.
---

# Evidence

You never set a confidence. You report what you saw, and the ledger prices it.
```

Rules:

- YAML frontmatter is required. `description` is the trigger. Put every
  "when to use" phrase in that field, not only in the body.
- The description names the job and the moment. Short descriptions
  under-trigger.
- The body is the procedure. One idea per section. Name the tools the
  model must call.
- Do not duplicate a tool schema. The tool already describes its input.
  The skill describes judgment the schema cannot encode.
- Do not tell the model to invent a score, a confidence, or a `sourceUrl`
  offered as proof. `docs/agent.md` forbids it.

## When to add a skill vs a tool

| You need | Add |
| --- | --- |
| A rule the model must follow ("never paste a thread into web search") | Skill |
| A side effect or a read from the outside world | Tool |
| A vendor that is sometimes missing | Capability, then a tool that checks it |
| A team-agent action a version may declare | Manifest action, then a runner tool |

If the model cannot obey the skill without a new tool, add the tool first.

## Checklist

1. Write the skill with a pushy `description`.
2. Name the tools it depends on.
3. Keep it short. The model already has `instructions.md`.
4. Restart the agent (`eve dev`) so eve reloads the filesystem.
5. Confirm `GET /eve/v1/info` lists the skill. The `diagnostics` count
   finds files eve ignored.
6. Exercise the path that should load it (identify, brief, builder, and so
   on).

## Do not

- Put a skill in `.agents/skills` and expect the CRM agent to load it.
- Put CRM customer text in an example the model might copy into
  `web_search`.
- Teach the model to overwrite a human-typed field. `lib/facts.ts`
  already refuses that.
- Add a skill that tells the builder to infer a Slack destination from
  prose. Destinations are chosen, never guessed.
