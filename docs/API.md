# API

Base URL: your deployment. Authentication: `Authorization: Bearer <key>`.

## `POST /v1/chat/completions`

OpenAI-compatible, with optional TopoSpace extensions. Streaming is supported
via `"stream": true` (SSE).

### Headers

| Header | Meaning |
|---|---|
| `Authorization: Bearer <key>` | required |
| `Idempotency-Key` | a retry of the same key and body returns the stored response instead of executing again. Scoped to the tenant, kept for 24 hours, refused with `409` if the body differs. Not supported on streaming requests. |
| `X-Tenant-Id` | serve as an end-user tenant. Requires the `tenant:delegate` scope; the asserted name is namespaced under the key's own tenant. |
| `X-Provider-Api-Key` | bring-your-own-key: the provider credential to use for this call. Requires `TOPOSPACE_BYOK=true`; never stored, logged or cached, and it cannot change the provider *endpoint*. |

### Request

| Field | Type | Meaning |
|---|---|---|
| `model` | string | `"auto"` lets the plane choose (recommended). A pinned profile name opts out of joint optimisation. |
| `messages` | array | Standard chat messages. Content-part arrays are flattened to text. |
| `max_tokens` | int | **Upper bound** on the budget, not the budget. Omit to let the plane decide. |
| `stream` | bool | Server-sent events. |
| `temperature`, `top_p`, `stop` | n/a | Passed through to the provider. |
| `topospace_state` | object | Runtime state; see below. |
| `topospace_context` | array | Candidate context objects the plane may select from. |
| `topospace_aspects` | array | Additional information needs, with weights. |
| `topospace_corpus` | string | Generate candidates from a dataset the plane holds instead of passing them. Ignored when `topospace_context` is present. |
| `topospace_candidates` | int | Size of the generated candidate pool. Clamped to 1–256. |
| `topospace_coverage` | string | Force a coverage rule: `satisfaction` (one object per aspect, precise) or `additive` (several documents, higher recall). Omit for auto: additive for a single aspect, satisfaction otherwise. |
| `topospace_restrict` | array | Plan over only these candidate ids. This is how a set selected in the topology view becomes a question about routing. Nothing else changes; unmatched ids are ignored. |
| `topospace_expect` | object | Per-request checks the plane should evaluate the answer against. |
| `topospace_max_latency_ms` | number | Refuse (503 `latency_ceiling`) instead of choosing a decision predicted to take longer. Defaults to `TOPOSPACE_MAX_LATENCY_MS` (none). A non-number or negative value is a 400. |
| `topospace_role` | string | Plan over this role's pool of models only (`roles` in the model catalogue, [CATALOG.md](CATALOG.md)). An unknown role, or a `model` pinned outside the role, is a 400. |
| `topospace_expected_calls` | int | Calls expected in this session; divides a local model's start cost. |

#### `topospace_state`

All coordinates are numbers in `[0, 1]`; unknown keys are ignored and
out-of-range values are clamped, so a newer client never breaks an older plane.

| Coordinate | Raises | Effect |
|---|---|---|
| `risk` | consequence of being wrong | conjunctive geometry, safer profiles, verification paths |
| `ambiguity` | under-determination | broader context, more budget |
| `hierarchy` | task depth | longer paths, larger difficulty scale |
| `candidate_pressure` | competition for the window | tighter neighbourhood radius |
| `latency_pressure` | value of a millisecond | faster profiles, fewer retries |

Omit the object entirely and the plane estimates the state from the request;
estimation error then enters as approximation error in the conditioning
variable, so send it if you know it.

#### `topospace_context`

```jsonc
{
  "id": "runbook-42",            // stable identifier, appears in the trace
  "type": "context",             // memory | tool | skill | agent | code | model | context
  "text": "…",                   // the material that would be injected
  "tokens": 180,                 // optional; estimated if absent
  "embedding": [0.1, …],         // optional; a hashing embedding is used otherwise
  "tags": ["payments"],          // optional; used by the structural component
  "sensitivity": "internal"      // public | internal | confidential | restricted
}
```

Selected objects are injected as a single system message ahead of the
conversation. Objects that are duplicated, irrelevant, or not worth their token
cost are dropped, so the token saving is provider-reported rather than notional.

#### `topospace_expect`

Declaring what a good answer looks like turns the quality coordinate from a
prediction into an observation, at no marginal cost and with no drift, and it
is what makes the promotion gate's quality guardrail meaningful for your traffic
(the gate leaves it inert while nothing evaluates the answer).

```jsonc
{
  "json": true,                    // must parse as JSON (a ``` fence is tolerated)
  "json_keys": ["summary", "risk"],// required top-level keys
  "contains": ["retry"],           // all of these substrings, case-insensitively
  "contains_any": ["twice", "2"],  // at least one of these
  "forbid": ["TODO"],              // none of these
  "cites": true,                   // must mention a selected context object id
  "min_chars": 40
}
```

Every satisfied check contributes equally to quality; success is the conjunction,
because a partially satisfied contract is a failed contract from the caller's
point of view.

### Response

Standard OpenAI fields plus:

```jsonc
"topospace": {
  "request_id": "req_…",
  "profile": "balanced",
  "path": "balanced->verify",
  "budget_tokens": 1255,
  "provider": "primary",
  "context_objects": ["runbook-42"],
  "context_tokens": 180,
  "coverage": 0.83,
  "overridden": false,
  "override_reasons": [],
  "coordination_gain": 0.049,
  "cost_micro_usd": 1738,
  "arm": "champion",
  "policy_version": "v3",
  "provider_model": "claude-sonnet-4-5",   // the id that answered: a fallback's after a failover
  "role": "writing",                       // present when the request named one
  "context_reused_tokens": 612   // present when > 0: context written first because
                                 // the serving model already holds it
}
```

`usage.prompt_tokens_details.cached_tokens` is passed through when the
provider reports it, and billed at the price book's `cached_input` price when
one is configured (at the full input price otherwise).

### Streaming

Events in order: `topospace.plan` (the decision, before any tokens), zero or
more unnamed `data:` chunks in OpenAI chunk format, a final chunk carrying
`usage`, then `topospace.result`, then `data: [DONE]`.

Client disconnects cancel the upstream call rather than merely stopping the
read; abandoned streams stop costing money.

### Errors

| Status | `error.type` | Cause |
|---|---|---|
| 400 | `invalid_request_error` | malformed body |
| 401 | `authentication_error` | missing, unknown, revoked or expired key |
| 403 | `permission_denied` | key lacks the required scope |
| 413 | `request_too_large` | body exceeds `TOPOSPACE_MAX_REQUEST_BYTES` |
| 422 | `safety_envelope_empty` | no admissible decision; a configuration fault |
| 503 | `latency_ceiling` | no decision is predicted to finish within `topospace_max_latency_ms`; the message names the quickest |
| 429 | `rate_limit_exceeded` | token bucket; `Retry-After` is set |
| 409 | `idempotency_conflict` | the key was reused with a different body |
| 429 | `quota_exceeded` | window token or cost limit |
| 502/503/504 | `upstream_error`, `circuit_open`, `upstream_timeout` | provider |

## `POST /v1/plan`: dry run

Plans a request and explains the decision **without executing it**: no provider
call, no charge, no ledger entry, no trace. The body is a chat request; the
response is the decision, the context selection with what was filtered and
pruned, the predicted outcome, and the ranked alternatives it beat.

```jsonc
{
  "dry_run": true,
  "decision": {"model": "balanced", "path": "balanced->verify", "budget_tokens": 763, "geometry": {...}},
  "state": {"risk": 0.9, "ambiguity": 0.6, ...},     // what it planned with, including estimates
  "context": {"objects": ["runbook"], "coverage": 0.47, "pruned_by_dilution": ["unrelated"], ...},
  "considered": {"candidates": 3, "filtered_by_radius": 0, "pruned_by_dilution": ["unrelated"]},
  "predicted": {"quality": 1.0, "success": 0.86, "latency_ms": 7379, "cost_micro_usd": 1408},
  "overridden": true,
  "override_reasons": ["high_risk_verification:risk=0.90 requires a verification stage..."],
  "coordination_gain": 0.0316,
  "route": {"provider": "openai", "model": "gpt-4o-mini",      // with a model catalogue
            "fallbacks": [{"provider": "anthropic", "model": "claude-sonnet-4-5"}]},
  "role": "writing", "role_models": ["strong", "cheap-hosted"],
  "alternatives": [{"model": "balanced", "path": "fast", "utility": 2.7512,
                    "held_tokens": 344, ...}],   // what that model already holds of its prompt
  "prompt": {                                     // how the context will be written
    "enabled": true, "placement": "system", "model": "gpt-4o-mini",
    "order": [{"id": "runbook:0", "class": "runbook", "tokens": 169, "held": true}],
    "reused_context_tokens": 169, "shared_prefix_tokens": 0,
    "models_holding": {"gpt-4o-mini": {"shared_tokens": 0, "orders": 1, "reusable_tokens": 169}},
    "max_latency_ms": null
  }
}
```

A dry run reads what the plane remembers sending but records nothing, so
explaining a request never changes what the next one is credited with.

Because the decision is a pure function of its recorded inputs (Theorem 11.10),
a dry run yields the decision the live path would take under the same policy.
What it cannot tell you is the *outcome*, which is why every figure it reports is
labelled a prediction (Proposition 11.11).

## `POST /v1/graph`: the topology of a decision

Plans a request and returns the **neighbourhood it was decided over**: the
objects considered, which survived selection and why, the geometry that decided
it, and the nearest-neighbour structure among them. Like `/v1/plan` it is a dry
run: no provider call, no charge.

```jsonc
{
  "nodes": [
    {"id": "__query__", "kind": "aspect", "distance": 0.0, "reason": "the dominant information need…"},
    {"id": "bill-003", "kind": "exposed", "distance": 0.383, "affinity": 0.617, "dispersion": 0.137,
     "tokens": 130, "label": "overage", "preview": "When a workspace crosses…",
     "reason": "selected: its marginal coverage paid for its tokens"},
    {"id": "hook-001", "kind": "pruned",  "reason": "pruned: marginal dilution exceeded its coverage"},
    {"id": "auth-004", "kind": "filtered", "reason": "outside the radius (ε = 0.45)"}
  ],
  "edges": [
    {"source": "__query__", "target": "bill-003", "distance": 0.383, "weight": 0.617,
     "dominant": "semantic",
     "components": {"semantic": 0.342, "structural": 0.5, "dependency": 0.5, "risk": 0.0}}
  ],
  "geometry": {"radius": 0.453, "exponent": 1.668, "weights": {"semantic": 0.55, …}},
  "path": ["draft"],
  "stats": {"candidates": 33, "exposed": 1, "pruned": 2, "filtered": 12,
            "coverage": 0.617, "exposed_tokens": 130, "offered_tokens": 3347,
            "token_reduction": 0.961, "aspect_margin": 0.0001, "mean_dispersion": 0.081},
  "decision": {...}, "state": {...}, "corpus": "support-playbook"
}
```

`balls` (at most 8, each `{center, radius, units}`) and `operator`
(`union` | `intersection` | `difference`) select a set over the same topology.
The response then carries each ball with its members and its distance to every
node, plus a `set` block with a witness radius per member. The witness is a
radius derived from the triangle inequality for which `B(x, ρ) ⊆ S` in the
ambient space; the smallest of them is the margin by which the set is open, and the members for
which no such radius exists (only possible for a difference, which leaves a
removed ball's boundary behind). `default_ball` names the ball the plane itself
used.

A ball's `center` may be `heaviest` (or `auto`) or `medoid` instead of an object
id, in which case the engine picks the member and reports `requested_center`,
`recentre_offset`, `contains_radius` and `contained_radius`. Its `units` restrict
it to those units, which is the subspace topology of the partition rather than a
display filter. The response also carries a `units` census of the pool.

Exposed nodes also carry `prompt_position` (1-based: the order the model reads
them), `held` (the serving model already read it in that position, so it is not
read again) and `context_class` (the ACES class that keeps a document's chunks
together). The response carries the same `prompt` block as `/v1/plan` and the
`predicted` outcome; the console draws the order as a path through the exposed
nodes, solid over the held prefix. `topospace_max_latency_ms` applies here too.

`neighbours` (1–6, default 3) sets how many nearest neighbours each node keeps.
The graph is built from the planner's own selection rather than recomputed, so
the picture and the routing cannot disagree.

## Datasets

| Endpoint | Scope | Purpose |
|---|---|---|
| `GET /v1/corpus` | `chat` | datasets this process holds, with descriptions and sizes |
| `GET /v1/corpus/{name}` | `chat` | summary plus a 24-object sample |
| `POST /v1/corpus` | `admin:policy` | register a dataset; requires `TOPOSPACE_CORPUS_UPLOAD=true`, refused in production |


## The console

`GET /console` serves a dependency-free operator console: compose a request with
context and state, dry-run it to see the decision and the candidates it beat, run
it for real (streamed or not), manage tenant plans, drive the policy lifecycle,
and watch health, quota, breakers, override rate, the ledger and the audit trail.

The **Topology** tab draws `/v1/graph`: the request's neighbourhood as an
animated force-directed graph where radius *is* composite distance and the dashed
ring is ε. Pick a dataset, ask a question, move a state slider and watch the
radius move; hover a node to see why it was kept, pruned or filtered, with the
component breakdown behind the number. The layout is written in the console
itself, since the Content-Security-Policy forbids external scripts.

Enabled outside production; set `TOPOSPACE_ENABLE_CONSOLE=true` to serve it in
production. It is static, holds no server-side session, and is served under a
Content-Security-Policy that permits only same-origin script and style.

## Other endpoints

| Endpoint | Scope | Purpose |
|---|---|---|
| `GET /v1/models` | `chat` | models with provider, capability, price and roles; the roles |
| `GET /v1/catalog` | `admin:policy` | providers (type, host, whether the key is set, breaker state), models with fallbacks and roles |
| `GET /v1/usage` | `usage:read` | quota snapshot and billing totals |
| `POST /v1/graph` | `chat` | the topology of a decision (dry run) |
| `GET /v1/corpus`, `GET /v1/corpus/{name}`, `POST /v1/corpus` | `chat` / `admin:policy` | datasets |
| `GET /healthz` | n/a | liveness |
| `GET /readyz` | n/a | readiness: store, policy, breakers, override rate, rollout |
| `GET /metrics` | n/a | Prometheus exposition |
| `GET /console` | n/a | operator console (static; the key is entered in the page) |
| `GET/POST /admin/keys`, `POST /admin/keys/{id}/rotate`, `DELETE /admin/keys/{id}` | `admin:keys` | key lifecycle |
| `GET /admin/policy`, `POST /admin/policy/{challenger,rollout,promote,rollback}` | `admin:policy` | policy lifecycle |
| `GET/PUT /admin/tenants[/{id}]` | `admin:keys` | per-tenant quota and rate limits |
| `GET /admin/audit` | `admin:keys` | the privileged-action audit trail |
| `GET /admin/traces` | `admin:policy` | decision traces |
| `GET /admin/ledger` | `usage:read` | ledger totals, hourly usage windows (`windows`) and chain verification |
