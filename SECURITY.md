# Security policy

Report vulnerabilities privately to **director@j1729labs.online**. Please do not
open a public issue for an unfixed vulnerability; expect an acknowledgement
within three business days.

The engine's trust boundaries, credential handling, redaction rules and
pre-GA hardening checklist are documented in [`docs/SECURITY.md`](docs/SECURITY.md).

Two rules worth repeating here, because violating either silently removes a
guarantee rather than producing an error:

1. **Tenancy comes from the authenticated key, never from the request body.** A
   client-declared tenant id lets one caller multiply its entitlement and voids
   the isolation bound protecting every other tenant.
2. **The mock provider must never serve production traffic.** It is a regression
   instrument with zero variance and unbounded bias as a measurement instrument;
   any figure derived from it is void. The service refuses to boot with it
   enabled in production.
