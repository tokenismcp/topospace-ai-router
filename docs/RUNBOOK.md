# Runbook

Every alert below names the quantity that moved and the guarantee it threatens.

## `topospace_override_rate` rising

**What it means.** The learned policy is proposing decisions the safety envelope
refuses. By Theorem 7.6 the utility price of safety is bounded by this rate
times the utility range, so a rising rate is a leading indicator of a utility
regression that has not surfaced yet.

**What to do.** Read `GET /readyz → override.by_rule` to see which predicate is
binding and `override.worst_region` for where. Do **not** weaken the envelope as
a first response: the constrained optimum exists and is attainable
(Theorem 7.5), so a high override rate means the learner is repeatedly proposing
actions it will never be allowed to take, which is wasted capacity rather than
foregone value.
Retrain or re-calibrate inside the envelope.

## Breaker open (`topospace_breaker_state = 2`)

**What it means.** A provider failed `PROVIDER_BREAKER_FAILURES` times in a row
and is being isolated. During the outage the upstream sees at most
`m + ⌈T/τ⌉` attempts regardless of offered load (Theorem 10.7).

**What to do.** Check the provider's own status first. If the outage is expected
to be long, raise `PROVIDER_BREAKER_OPEN_SECONDS`; the optimal timeout grows as
the square root of the outage duration (Corollary 10.8). Confirm the fallback
provider is actually independent: shared credentials or shared egress reintroduce
the common-mode floor (Proposition 10.3) and no retry budget crosses it.

## Elevated `topospace_rate_limited_total`

**What it means.** Admission control is doing its job. Per Theorem 11.1 a tenant
may consume `c + rT` in any window of length `T`; more than that is refused.

**What to do.** Confirm the tenant's plan, then check `Σ rates ≤ served rate`
before raising anyone's limit, since raising a burst raises everyone's worst-case
delay, since the backlog bound is a sum (Theorem 11.3). If many tenants are
limited at once, the service rate is the problem, not the buckets.

## `topospace_scarcity_price` at its ceiling

**What it means.** A tenant's window quota is nearly exhausted, so the shadow
price of a token has risen and every budget is being shortened in proportion to
its own difficulty scale (Corollary 8.5).

**What to do.** This is the designed behaviour, and it is better than a flat cut:
easy requests are barely affected while hard ones absorb the reduction. If the
tenant needs more, raise the quota. Do not disable the controller, which would
replace graceful degradation with a hard refusal at the limit.

## Challenger rolled back

**What it means.** The live guard accumulated anytime-valid evidence that the
challenger is worse than the tolerance (Theorem 9.11). The registry has already
rejected the challenger; the champion is unchanged.

**What to do.** Read `GET /readyz → rollout.rollback_reason` for the observed
mean difference. Total exposure-weighted damage is bounded by
`2R√(2N ln(1/α))` (Corollary 9.12). Compute it and record it in the incident.
Do not re-register the same policy without changing something: the guard will
reach the same verdict, more slowly.

## Drift detected

**What it means.** The workload law has changed. Every promoted comparison was
an expectation under the *old* law, and by Theorem 12.5 transporting a utility
estimate across laws carries a bias proportional to their total-variation
distance.

**What to do.** Re-certify the champion rather than only acknowledging the alert.
If a savings claim is published for this tenant, re-run the paired measurement:
the old interval no longer covers the new estimand.

## `quality_judge.configured` is false

**What it means.** Nothing is looking at the answers. The quality coordinate on
every response is the outcome model's *prediction*, and the promotion gate's
quality and success guardrails are inert, so a challenger could be promoted on
cost and latency alone.

**What to do.** This is a deployment gap, not an incident, but it should not be
a permanent one. Either have callers declare `topospace_expect` and set
`TOPOSPACE_JUDGE=structural` (free, reproducible), or configure a rubric judge.
Until then, do not read a promotion as evidence about quality.

## Deferred verdicts are piling up

**What it means.** `quality_judge.pending_verdicts` is large: the rubric grader
is slower than the traffic it samples, or it is failing and retry-free by design.

**What to do.** Check `quality_judge.failures` and `spent_micro_usd` against the
window budget. A judge that has exhausted its allowance degrades to the
heuristic silently by design, which is correct behaviour and worth knowing about.
Lower `TOPOSPACE_JUDGE_SAMPLE_RATE` rather than raising the budget if the grader
is the bottleneck; evidence accrues per observation, so half the sample doubles
the promotion time and halves the spend.

## Readiness reports `draining`

**What it means.** The process received a shutdown signal. It has stopped
reporting ready so the load balancer removes it, and is finishing the requests
it had already accepted.

**What to do.** Nothing, unless it persists. `inflight` on `/readyz` counts what
is left; if it does not reach zero within the drain timeout the process logs
"shutting down with requests in flight" and exits anyway. A drain that regularly
times out means requests are slower than the timeout, which is a latency
question, not a shutdown one.

## Idempotent replays are climbing

**What it means.** `topospace_idempotent_replays_total` counts retries answered
from the store rather than executed. A rise means callers are retrying, usually
because something upstream of the plane is timing out.

**What to do.** This is the mechanism working: those retries cost nothing and
produced no second ledger entry. Look at the caller's timeout against the plane's
p95, not at the plane. A rise in `idempotency_conflict` is different: that is a
client reusing a key with a different body, which is a bug in the caller.

## Ledger verification fails

**What it means.** The hash chain does not recompute: rows were edited, deleted
or reordered after they were written.

**What to do.** Treat it as a data-integrity incident, not a billing bug. Freeze
invoicing for the affected tenant, identify the first broken sequence number
(`topospace verify-ledger` reports the totals; the store holds the rows), and
reconstruct from provider-side usage records. Investigate write paths: the only
supported writer is the engine, and `INSERT … ON CONFLICT DO NOTHING` makes
double-append a no-op rather than a fork.

## Predicted success is low across a segment

**What it means.** The plane is telling you it cannot serve that traffic well at
any admissible cost: demand exceeds the capability the budget can buy.

**What to do.** Three real options: raise `cost_reference_micro_usd` (the segment
is worth more than the configuration says), add a more capable profile to the
catalogue, or accept and route the segment elsewhere. Silently raising budgets
is not a fourth option; the optimiser already considered it.
