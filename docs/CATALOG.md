# The model catalogue: your own models, your own API keys

Before 2.17 the plane could route to three models written into the source. A
catalogue file replaces that: you name the providers you have accounts with, the
models on them, what each costs and how capable it is, who takes over when one
fails, and which models fill which role. Nothing about a model or a provider is
in the code, and no key is ever in the file.

```bash
topospace models init --out models.json     # a starter to edit
export OPENAI_API_KEY=...  ANTHROPIC_API_KEY=...  GEMINI_API_KEY=...   # your own keys
topospace models check models.json          # validates it; shows which keys are set
TOPOSPACE_CATALOG=models.json ./topospace serve
```

Use any mix: a model on this machine (Ollama, llama.cpp, vLLM, LM Studio), OpenAI,
Anthropic, Google, or anything that speaks the OpenAI API. The plane's own
request format is one thing; each provider type translates it into that
provider's wire format and reads usage back from it.

## What a request does

1. The planner chooses a model, a path and a budget jointly, as before -- among
   all the catalogue's models, or among the pool of one **role** if the request
   names one (`"topospace_role": "writing"`).
2. The model's **provider** is called with the id that provider knows it by.
3. If that provider fails (after its retries, or with its circuit open), the
   model's **fallback** is tried, on *its* provider with *its* id, then that
   model's fallback, up to three deep. A fallback is a different model on
   perhaps a different vendor, so the answer comes from whichever served it, and
   the ledger and the response say which: `provider` and `provider_model` are the
   ones that answered, priced at that model's price.
4. A streamed request fails over only **before its first chunk**. After that,
   switching would splice two answers into one, so the error is passed on.
5. A request carrying a **bring-your-own-key** credential has no fallbacks. The
   credential belongs to one provider; sending it to another vendor would hand a
   customer's key to a third party, and falling back on the deployment's own key
   would bill the deployment for a customer who chose to pay their own way.

## The file

JSON or TOML (`.json`, `.toml`). `examples/models.example.json` and `.toml`
are complete. Keys that start with `_` are ignored, for notes.

```json
{
  "providers": {
    "ollama":    {"type": "openai", "base_url": "http://localhost:11434/v1", "auth": "none"},
    "openai":    {"type": "openai", "api_key_env": "OPENAI_API_KEY"},
    "anthropic": {"type": "anthropic", "api_key_env": "ANTHROPIC_API_KEY"}
  },
  "models": [
    {"name": "local-small", "provider": "ollama", "model": "gemma3:4b",
     "capability": 0.55, "input_usd_per_mtok": 0, "output_usd_per_mtok": 0,
     "tier": 0, "fallback": "cheap-hosted"},
    {"name": "cheap-hosted", "provider": "openai", "model": "gpt-4o-mini",
     "capability": 0.62, "input_usd_per_mtok": 0.15, "output_usd_per_mtok": 0.6,
     "cached_input_usd_per_mtok": 0.075, "fallback": "strong"},
    {"name": "strong", "provider": "anthropic", "model": "claude-sonnet-4-5",
     "capability": 0.86, "input_usd_per_mtok": 3, "output_usd_per_mtok": 15}
  ],
  "roles": {"chat": ["local-small", "cheap-hosted"], "writing": ["strong", "cheap-hosted"]}
}
```

### Providers

| field | meaning |
| --- | --- |
| `type` | `openai` (any server exposing `POST {base_url}/chat/completions` in OpenAI's format), `anthropic` (Messages API), `gemini` (`generateContent`), or `mock` (development only; refused without `ALLOW_MOCK_PROVIDER`) |
| `base_url` | defaults: OpenAI `https://api.openai.com/v1`, Anthropic `https://api.anthropic.com`, Gemini `https://generativelanguage.googleapis.com` |
| `api_key_env` | the **name** of the environment variable holding the key. A literal `api_key` is refused, and so is a value that looks like a key where a name belongs |
| `auth` | `"none"` for an endpoint that needs no key (a local server) |
| `headers` | extra non-secret headers; any that carry credentials (`Authorization`, `x-api-key`, ...) are refused |
| `timeout_seconds` | per-call deadline for this provider |

A provider whose key variable is unset stops the engine from starting, naming the
variable, rather than failing the first request routed to it.

### Models

| field | meaning |
| --- | --- |
| `name` | what you pin (`"model": "writing-model"`), what traces and the console show |
| `provider`, `model` | which provider, and the id sent to it |
| `capability` | **required**, in (0, 1]: how likely the model is to satisfy a request, from your own evaluation. The planner trades it against price and speed, so it is the most important number here (see below) |
| `input_usd_per_mtok`, `output_usd_per_mtok` | **required**, USD per million tokens as providers publish them; `0` for a local model. Stored as integer micro-USD per 1k tokens (the ledger is integral); a price needing finer resolution is rounded up and reported |
| `cached_input_usd_per_mtok` | optional: the price of an input token the provider served from its prompt cache. Setting it is also what tells the planner the provider caches prefixes |
| `first_token_ms`, `tokens_per_second` | optional; defaults 600 ms and 60 tokens/s, reported as a warning when used. They only affect predicted latency |
| `max_context_tokens` | optional; default 128,000 |
| `tier` | optional; default 0 below capability 0.7, 1 below 0.9, else 2. The safety envelope bars tier 0 above a risk threshold |
| `fallback` | optional: another model's `name` |
| `supports_tools`, `supports_streaming`, `max_sensitivity` | optional |

Two models may not share an id with different prices, because the price book is
keyed by the id sent upstream.

### Roles

`"roles": {"writing": ["strong", "cheap-hosted"]}` is a pool. A request naming
the role is planned over that pool only, so the planner still trades quality
against cost -- inside the models you said are fit for that kind of work. A
request that names an unknown role is a 400 that lists the roles; pinning a
model that is outside the named role is a 400 too. The role and its pool are
recorded in the trace and are part of the replay digest.

A role is a restriction, not a pipeline: one request still goes to one model.
Running a draft on one model and a check on another is not implemented.

## Where capability comes from

There is no default. A number guessed from a model's size ranked an 8B model above
a 4B that answered more questions correctly, and a catalogue
that ships invented capabilities would be passing that guess off as data. Start
from your own questions: `topospace fleet-calibrate` measures local models on 24
probes, `topospace calibrate` fits constants from your own traffic. Until you have measured, use relative
ordering you believe and treat the planner's choices as a first draft.

## Mixing in the local fleet

With `TOPOSPACE_FLEET=true` as well, the fleet's models (GGUF files on this
machine, including a mixture-of-experts model larger than memory) join the catalogue under the provider name `local`, with the
planner's usual wake-cost and memory logic. Their names must not collide with
the file's, and a catalogue provider may not be called `local`.

```bash
TOPOSPACE_FLEET=true TOPOSPACE_CATALOG=models.json ./topospace serve
```

## What each provider type translates

| | OpenAI-compatible | Anthropic Messages | Gemini `generateContent` |
| --- | --- | --- | --- |
| system messages | passed as messages | joined into `system` | joined into `systemInstruction` |
| roles | as given | `user`/`assistant`; unknown roles become `user`; consecutive turns merged; a leading assistant turn gets a placeholder user turn | `user`/`model`; consecutive turns merged |
| max tokens | `max_tokens` | `max_tokens` (required there) | `generationConfig.maxOutputTokens` |
| input usage | `prompt_tokens` | `input_tokens` + cache reads + cache writes | `promptTokenCount` |
| cached input | `prompt_tokens_details.cached_tokens` | `cache_read_input_tokens` | `cachedContentTokenCount` |
| output usage | `completion_tokens` | `output_tokens` | `candidatesTokenCount` + `thoughtsTokenCount` |
| key sent as | `Authorization: Bearer` | `x-api-key` | `x-goog-api-key` (never in the URL) |
| streaming usage | final chunk (`include_usage`) | `message_start` + `message_delta` | the last chunk's cumulative `usageMetadata` |

Limits to know about:

* **Text chat only.** Tool or function calls, images and other content parts are
  not translated, in any provider type.
* Anthropic **cache writes are priced as ordinary input**, because the usage type
  cannot separate them; where a provider charges a premium for them the ledger
  understates the bill. Gemini's thinking tokens are counted as output, as billed.
* Provider-specific options (Anthropic `top_k`, Gemini `thinkingConfig`, ...)
  have no channel yet; `ProviderRequest.extra` is used only by the OpenAI type.
* **Verification.** The Anthropic and Gemini adapters are tested against local
  servers that speak the documented wire formats, and have **not** been run
  against the live services. `topospace models ping` does that with your key, at a
  fraction of a cent. The OpenAI-compatible type has run against a real
  `llama-server` and a real Ollama, and not against hosted vendors.
  See [MEASUREMENTS.md](MEASUREMENTS.md).

## The CLI

```bash
topospace models init [--out FILE] [--force]    # a starter catalogue
topospace models check [FILE]                    # validate; keys set or not; routes and roles
topospace models ping [FILE] [--model NAME ...] --confirm-spend [--max-spend-usd 0.05]
```

`ping` sends one tiny request (at most 8 output tokens) to each model to prove
the key and the model id work. It is real spend, so it refuses without
`--confirm-spend`, prints the worst case first, and refuses a set of calls whose
worst case exceeds `--max-spend-usd`.

## The API and the console

* `POST /v1/chat/completions`, `/v1/plan`: `"topospace_role": "writing"`. The
  response's `topospace` block gains `provider_model` (the id that answered, which
  is a fallback's after a failover) and `role`; the dry run gains `route`
  (provider, model id, fallbacks), `role` and `role_models`.
* `GET /v1/models` lists each model's provider and roles, and the roles.
* `GET /v1/catalog` (`admin:policy`): providers by type, **host** and whether
  their key is set (never the key, never the path or credentials of a URL),
  circuit-breaker state, and the models with their fallbacks and roles.
* The console reads these: the model list and a role selector in the Playground,
  a "via provider / model" chip on each decision, and a **Models & providers**
  card in Observability. The topology graph is unchanged: a role changes which
  model is chosen, not which context is selected.

## Settings

| variable | meaning |
| --- | --- |
| `TOPOSPACE_CATALOG` | path of the catalogue file. A catalogue carries explicit prices, so it satisfies the production price requirement on its own; it cannot be combined with `PRIMARY_PROVIDER_*` / `FALLBACK_PROVIDER_*` |
| the variables your providers name | e.g. `OPENAI_API_KEY`. Never in the file |

Catalogue changes take effect on restart. The catalogue's content hash is part of
the recorded catalogue fingerprint, so two deployments that differ only in a
price or a capability do not look identical to replay.
