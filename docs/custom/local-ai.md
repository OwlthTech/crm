# Local and self-hosted models

The agent does not read a model id from `.env`. The compiled fallback is
`zai/glm-5.2-fast` in `DEFAULT_AGENT_MODEL` (`packages/db/src/settings.ts`).
Settings → General stores an override in `AppSetting`. Open conversations
keep the model they started with.

eve reaches that model through the Vercel AI Gateway. On Vercel, OIDC is
enough. Off Vercel, set `AI_GATEWAY_API_KEY`.

Read `docs/agent.md` and `docs/environment.md` first.

## What "local AI" means here

People say "local AI route" for three different things. Only one is a
supported setting today.

| Meaning | Supported today |
| --- | --- |
| Pick another Gateway model that has the `tool-use` tag | Yes. Settings → General |
| Run eve against a local OpenAI-compatible server (Ollama, vLLM, llama.cpp) | Not built. Pattern below |
| Put an LLM in the Nest API | Forbidden. Intelligence stays in `apps/agent` |

The chooser in `ModelCatalogService` loads
`https://ai-gateway.vercel.sh/v1/models` and keeps models where
`type === "language"` and `tags` includes `tool-use`. A local server that
is not in that catalogue never appears in the UI.

## Use another Gateway model

1. Set `AI_GATEWAY_API_KEY` in the root `.env` when you are not on Vercel.
2. Add it to Turbo pass-through (already present).
3. Open Settings → General.
4. Pick a `tool-use` model. `settings.setAgentModel` writes `agentModelId`
   and `agentModelContextWindow`.
5. `apps/agent/agent/lib/model.ts` reads that row on `session.started`.
   A failed read logs and keeps the compiled fallback. It never throws.

Team-agent runs pin `modelId` on the version row. Changing the workspace
default does not rewrite a deployed version.

## Point eve at a local server (not built)

eve uses the Gateway by default. A local route needs three facts that this
repo does not yet store:

- base URL (`http://127.0.0.1:11434/v1` or similar)
- model id the local server expects
- context window, because eve does not inherit it

Do not copy platform keys (`MCAI_LLM_API_KEY` and friends) into the
project. If you add a local route, use project-owned names:

```
# Local OpenAI-compatible server for eve. Empty means Gateway.
# USER_LLM_BASE_URL="http://127.0.0.1:11434/v1"
# USER_LLM_API_KEY="ollama"
# USER_LLM_MODEL="llama3.1"
# USER_LLM_CONTEXT_WINDOW="128000"
```

Then:

1. Document every variable in `.env.example`.
2. Pass them through `turbo.json` and `apps/agent/turbo.json`.
3. Keep them out of `env.validation.ts` unless the API reads them. The API
   must not.
4. Teach `selectedModel()` (and the runner's `session.started` hook) to
   return that model id and context window when the local URL is set.
5. Confirm eve's installed docs for custom providers before you wire
   `baseURL`. Guessing typechecks, then talks to the Gateway anyway.
6. The local model still needs tool calling. A completion-only model
   cannot run this agent.
7. Missing URL falls back to Gateway / `DEFAULT_AGENT_MODEL`. Never throw.

Do not hardcode an endpoint in `agent.ts`. Open sessions must keep their
model.

## What not to change

- Do not put the model id in `apps/api` business logic.
- Do not add a second chooser on a connection page.
- Do not skip `modelContextWindowTokens`. eve will not infer it.
- Do not use a model without `tool-use`. The catalogue filter exists
  because this agent is tools first.
- Do not log prompts or completions. The audit hook already files
  `AgentEvent`.

## Local dispatch while you try a model

`eve dev` does not fire cron schedules. Set `AGENT_BRIDGE_SECRET` and
`AGENT_URL=http://127.0.0.1:2000` (IPv4). After a task is queued:

```
bun run --filter=agent dispatch
```

That is the production poke. It spends real credits on vendor tools even
when the model is local.

`GET /eve/v1/info` shows the loaded model, tools, and skills. Send
`Host: agent.example.com` when you curl loopback. `localDev()` accepts any
loopback request and will not prove the bridge.

## Checklist

1. Decide Gateway vs local server.
2. Gateway: set `AI_GATEWAY_API_KEY` off Vercel, pick a model in Settings.
3. Local: add `USER_LLM_*` placeholders, wire `selectedModel()`, keep the
   fallback.
4. Confirm tool-calling works on one identify task before you roll it out.
5. Leave `DEFAULT_AGENT_MODEL` as the compiled safety net.
