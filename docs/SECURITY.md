# Security

## Trust boundaries

| Boundary | Rule |
|---|---|
| Client → plane | Bearer key. **Tenancy is derived from the key, never from the request body.** |
| Plane → provider | Separate upstream credential. Customer keys and provider keys are different secrets with different lifetimes. |
| Plane → telemetry | Identifiers and numbers only. Prompts, completions and embeddings never reach logs or traces. |
| Operator → plane | Scoped admin keys (`admin:keys`, `admin:policy`). |

## The tenancy rule

Reading tenant identity from a client-controlled field is not only a security
error; it is a *correctness* error for the quota subsystem. A single caller
forging `k` identifiers obtains `k` times its entitlement, and the provisioning
hypothesis that protects every other tenant fails without any misconfiguration
(Proposition 11.5). An integration bug that emits a random tenant id per process
produces the same failure with no attacker at all.

The engine never accepts a tenant id from a request body. `APIKeyStore.verify`
returns the record, and the record carries the tenant.

## Credential handling

* Keys are stored as HMAC-SHA-256 hashes under a server-side pepper
  (`TOPOSPACE_KEY_PEPPER`). The store can compare a key; it cannot return one.
* Keys carry 256 bits of entropy, so a single-pass HMAC is sufficient; a work
  factor exists to slow guessing of low-entropy secrets and would only add
  latency here.
* Verification failures return one message for every cause (unknown key, wrong
  secret, revoked, expired), so the error does not enumerate.
* Rotation issues a replacement and expires the old key after a bounded grace
  window; rotation without a window is an outage, and rotation without a bound
  is not rotation.
* Losing the pepper invalidates every stored key. Back it up with the same care
  as the database and rotate it only with a planned re-issue of all keys.

## Redaction

`observability.redact` is applied to every structured log field. It replaces
values under sensitive keys (`messages`, `content`, `authorization`, `api_key`,
`embedding`, …) with a type/size summary, and scrubs secret-shaped substrings
(`sk-…`, `Bearer …`, long base64 blobs) from free text. Decision traces record
object *identifiers*, never object text: a trace store that held prompts would be
a second copy of customer data behind a different access-control surface.

## Reporting a vulnerability

Email `director@j1729labs.online` with a description and reproduction. Please do
not open a public issue for an unfixed vulnerability. Expect an acknowledgement
within three business days.

## Hardening checklist before general availability

- [ ] Threat model reviewed and recorded, including the provider trust boundary.
- [ ] Penetration test covering auth, tenancy forging, and admin scope escalation.
- [ ] Backup and restore drill, ending in `topospace verify-ledger`.
- [ ] Secret rotation runbook exercised for the pepper and both provider keys.
- [ ] Dependency and container scanning in CI, with an SBOM published per release.
- [ ] Log sampling reviewed by someone who did not write the redaction rules.
