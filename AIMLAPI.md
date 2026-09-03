# AI/ML API and this repository — recorded position

**Status: no integration is being proposed for `letta-ai/letta`, and none should be.**

This file exists so that the decision is written down rather than merely absent.
It is fork-only: it must not be offered upstream (see "Why not upstream" below).

Verified 2026-09-03 against `letta-ai/letta` `main` @ `4511fa0` and the
`archive` branch @ `56ba9c2`.

---

## 1. What this repository is today

`main` is a landing page: **12 tracked files**, no source code, no build, no
test suite.

```
.github/ISSUE_TEMPLATE/config.yml   .github/TRUSTED_CONTRIBUTORS
.github/workflows/issue-guard.yml   AGENTS.md
AI_POLICY.md                        CITATION.cff
CONTRIBUTING.md                     LICENSE
PRIVACY.md                          README.md
SECURITY.md                         TERMS.md
```

The retired Letta V1 Python server is preserved on the `archive` branch (1,156
files). That branch *does* contain a real provider surface —
`letta/schemas/providers/`, 23 provider modules (anthropic, azure, baseten,
bedrock, cerebras, chatgpt_oauth, deepseek, google_gemini, google_vertex, groq,
letta, lmstudio, minimax, mistral, ollama, openai, openrouter, sglang, together,
vllm, xai, zai) — so an `aimlapi` provider there is *technically* a small change.
It is the thing the repository's own rules exist to forbid.

The only pre-existing mention of us anywhere in the repository is incidental:
`letta/model_specs/model_prices_and_context_window.json` on `archive` vendors
LiteLLM's price map, whose `litellm_provider: "aiml"` image rows cite
`https://docs.aimlapi.com/` as their price source. That is upstream LiteLLM data
riding along, not a Letta integration.

## 2. Why not upstream — the rules, quoted verbatim

`AGENTS.md`, lines 9–20, under the heading **"Prohibited uses"**:

> Do not use the `archive` branch, an old release or tag from this repository,
> the old Python server packages, or the `letta/letta` Docker image for any of
> the following:
>
> - production, staging, demos, or new applications
> - benchmarks, evaluations, experiments, or academic research
> - comparisons with other agent or memory systems
> - testing current Letta behavior, performance, memory, APIs, or model support
> - **building a new integration, adapter, fork, or compatibility layer**
> - copying an old implementation into another project
>
> Do not patch the archived source, a pinned old package, or the retired Docker
> image to make a benchmark or integration run.

(`AGENTS.md:11`, `AGENTS.md:17`, `AGENTS.md:20`. The emphasis is ours; the
wording is theirs.)

`AGENTS.md:48`:

> Do not open issues or pull requests here for current Letta development. Use
> [`letta-ai/letta-code`](https://github.com/letta-ai/letta-code/issues).

`CONTRIBUTING.md:3`:

> This repository is archived and does not accept changes to the retired Letta
> V1 server.

`AI_POLICY.md:9`:

> Do not file issues or pull requests in this repository with an expectation
> that the archived code will be patched or maintained. The code is open source
> and free to use / adapt if you need to patch anything yourself.

Two things follow, and they are different things:

1. **A provider PR to `letta-ai/letta` is off the table.** Adding an `aimlapi`
   provider to `archive` is the literal example the "Prohibited uses" list names.
   `main` has nothing to add it to.
2. **Keeping a fork with a note in it is not forbidden.** The licence is
   Apache-2.0 and `AI_POLICY.md:9` says the code is "free to use / adapt if you
   need to patch anything yourself". What is forbidden is *presenting* work
   built off the retired V1 server as an integration with, or a measurement of,
   current Letta.

## 3. Where current Letta actually resolves models

`README.md:5` points at `letta-ai/letta-code`. That repository's own
`.skills/adding-models/SKILL.md` describes the ownership boundary directly:

> Letta Code does not bundle a model catalog:
>
> - API and hosted presets come from the server's `GET /v1/models/catalog`
>   response.
> - Local model inventory comes from pi-ai and the active provider runtimes.
>
> Add the model at the source that owns it. A hosted preset belongs in the
> server catalog. A local provider model belongs in pi-ai or that provider's
> discovery runtime.

That is two different surfaces, and only one of them is closed:

- **Hosted / cloud BYOK — closed to us.** `src/backend/api/providers.ts:17-46`
  only *calls* `GET /v1/providers` and `POST /v1/providers/check` with a
  `provider_type`; nothing in `letta-code` serves those routes. A new
  first-class `aimlapi` `provider_type` needs Letta-side (App Server / Cloud)
  acceptance and cannot be completed by a pull request to `letta-code`.
- **Local provider runtime — open, and it is not in either Letta repository.**
  It is [`earendil-works/pi-ai`](https://github.com/earendil-works/pi)
  (`packages/ai`, MIT, ~101k stars, actively pushed), a normal open-source
  package with one file per provider under `packages/ai/src/providers/`. It
  ships 44 providers today, aggregators included — `openrouter`,
  `vercel-ai-gateway`, `cloudflare-ai-gateway`, `opencode` — and **no
  `aimlapi`**. A provider there is roughly the ~25-line `createProvider({ id,
  name, baseUrl, auth, models, api })` shape used by
  `packages/ai/src/providers/openrouter.ts`.
- `letta-code` also has a user-installable **mod** system
  (`~/.letta/mods/`, `LETTA_MODS_DIR`, `docs/examples/mods/README.md`) whose
  `registerPiProvider(name, config)` API
  (`src/backend/dev/pi-provider-mod-registry.ts`) accepts `baseUrl`, `apiKey`,
  `headers`, `models` and `listModels`. A provider can therefore be distributed
  to Letta Code users **without any pull request to any Letta repository**.

**So the correct future target is `earendil-works/pi-ai`, not this repository
and not `letta-code`.** That is a separate piece of work against a separate
project, and it does not belong in this fork.

Note for whoever picks that up: `letta-code`'s `CONTRIBUTING.md` and
`AI_POLICY.md` require a per-PR AI-usage disclosure and auto-close noncompliant
submissions unless the author is listed in `.github/TRUSTED_CONTRIBUTORS`.

## 4. What was verified live

The `openai-completions` path that pi-ai (and therefore current Letta) would use
against `https://api.aimlapi.com/v1` was exercised for real on 2026-09-03, by
building a provider with pi-ai `0.84.4`'s own `createProvider` and streaming
through it — not by curl and not against a mock:

| Model | Chat | Tool call |
| --- | --- | --- |
| `openai/gpt-4o-mini` | `stopReason: stop`, returned `AIMLAPI OK` | `stopReason: toolUse`, `get_weather({"city":"Paris"})` |
| `openai/gpt-5-5` | `stopReason: stop` | `stopReason: toolUse` |
| `anthropic/claude-sonnet-4.6` | `stopReason: stop` | `stopReason: toolUse` |
| `google/gemini-2.5-flash` | `stopReason: stop` | `stopReason: toolUse` |

All four ids pass the id-or-alias check against
`GET https://api.aimlapi.com/v1/models?include=all` (936 entries, 353 of type
`openai/chat-completions`) and all four were called.

Two things worth knowing before anyone writes the pi-ai provider:

- pi-ai omits unset request keys rather than sending `null`. The observed
  outbound body is `model, messages, stream, prompt_cache_key,
  prompt_cache_retention, stream_options, store[, tools]` with **no
  null-valued keys**, so it does not trip the per-model null rejections on our
  side. Any future provider should keep it that way.
- **`Provider.headers` does not reach the wire in pi-ai 0.84.4.**
  `models.d.ts:102` documents `readonly headers?: ProviderHeaders` on
  `Provider`, but `Models.getAuth()` (`dist/models.js:282-289`) merges
  `providerOrModel.headers` — and `applyAuth` passes the **Model**, never the
  Provider. Headers set on `createProvider({ headers })` were confirmed absent
  from the outbound request; the same headers set on the **model** objects were
  confirmed present. Attribution headers therefore belong on the model entries,
  or the upstream merge needs fixing first.

## 5. Connecting aimlapi.com to current Letta today

*(Fork-only section. It is promotional in intent and there is no upstream
surface to attach it to — drop this section, and the commit that adds it,
before any conversation with the Letta maintainers.)*

No pull request and no new provider code are needed to use **aimlapi.com** with
current Letta. Letta Code already ships an `openai-compatible` BYOK entry
(`src/providers/byok-providers.ts:140-153` for cloud, `:297-309` for local),
and our endpoint is OpenAI-compatible:

| Field | Value |
| --- | --- |
| Provider | **aimlapi.com** |
| Base URL | `https://api.aimlapi.com/v1` |
| Endpoint | `POST /v1/chat/completions` |
| API key env var | `AIMLAPI_API_KEY` |
| Catalog | `GET https://api.aimlapi.com/v1/models?include=all` |

```
letta
/provider   ->  OpenAI-compatible API
  API Key   ->  <your aimlapi.com key>
  Base URL  ->  https://api.aimlapi.com/v1
```

Model ids are the catalog's own, e.g. `openai/gpt-5-5`,
`anthropic/claude-sonnet-4.6`, `google/gemini-2.5-flash`, `openai/gpt-4o-mini`
— all four verified live, chat and tool calling, in §4 above.

Two endpoint caveats worth carrying into any future provider entry:

- Declare the **chat completions** path. `POST /v1/completions` does not exist
  and returns 404.
- `POST /v1/responses` exists but serves only a minority of the chat catalog;
  the other ids 404 there while working on `/v1/chat/completions`. Anything
  built on a runtime that defaults to Responses must be pinned to the chat form.

### Attribution headers — deliberately not configured here

Our four attribution headers (`X-AIMLAPI-Partner-ID`, `X-AIMLAPI-Source`,
`HTTP-Referer`, `X-Title`) are **not** set up in this document and no partner id
is recorded. That is intentional on both counts:

- This repository sends no requests, so there is nothing to attach a header to.
- No partner id has been registered for a Letta surface, and inventing one is
  worse than omitting it: a value that does not match
  `^part_[A-Za-z0-9]{1,64}$` is dropped silently at our gateway and earns
  nothing, while looking correct in review.

When a real provider entry is written against `earendil-works/pi-ai`, the
headers go on the **model** entries, not on `createProvider({ headers })` — see
the pi-ai finding in §4 — and a test should assert the id against that regex.

## 6. Verification notes

There is no build and no test suite in this repository to run: `main` is
markdown, one issue template and one workflow. The single workflow,
`.github/workflows/issue-guard.yml`, is triggered by `issues: [opened]` only —
it does not touch pull requests, and nothing in this repository auto-closes
them. The auto-close behaviour is `letta-code`'s, not this repository's.
