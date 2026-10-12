# Deployment

The compiled program runs as a **single replica**: one process, a SQLite system of
record (or memory), in-process rate limiting. That is correct and fully guaranteed
for one process. Running several replicas needs shared state (a database and Redis)
that the compiled distribution does not include; the engine refuses to start with
`TOPOSPACE_REPLICAS` above 1 without it.

## As a service

```bash
export TOPOSPACE_ENV=production
export TOPOSPACE_API_KEY=ts_live_...          # bootstrap credential
export TOPOSPACE_KEY_PEPPER=...               # at least 16 bytes; do not rotate casually
export TOPOSPACE_CATALOG=/etc/topospace/models.json   # your models, prices, fallbacks
export OPENAI_API_KEY=... ANTHROPIC_API_KEY=...        # the variables the catalogue names
export TOPOSPACE_DATABASE_URL=/var/lib/topospace/topospace.db
./topospace check && ./topospace serve --host 0.0.0.0 --port 8000
```

In production the program refuses to start without a catalogue (or a price book),
because every routing decision, quota figure and ledger row derives from your
prices; with the stub provider enabled; without durable storage; or with a weak
pepper. The refusal messages name the setting. `.env.example` lists every setting.

Keep the data directory on a durable volume: a restart that forgets it forgets key
revocations, quota windows and the billing ledger.

## Docker

```bash
docker run -p 8000:8000 --env-file .env -v topospace-data:/data \
  ghcr.io/tokenismcp/topospace-ai-router:<version>
```

The image runs as a non-root user and writes only to `/data`.

## Health and metrics

* `GET /healthz`: liveness; never depends on the store.
* `GET /readyz`: readiness: store health, policy identity, circuit breakers, safety
  override rate, rollout. Use it as the load-balancer gate.
* `GET /metrics`: Prometheus exposition.

## Upgrading

Unpack the new archive beside the old one and point it at the same data directory.
Check `GET /readyz` for the expected policy fingerprint, and watch the safety
override rate and breaker states for a full traffic cycle before relying on it.

## What the compiled build does not include

Multi-replica operation (PostgreSQL and Redis support), and the Node migration
adapters. Both exist in the source, which is not distributed.
