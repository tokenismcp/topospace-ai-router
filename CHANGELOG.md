# Changelog

## 2.17.2

* **Distributed as a compiled program** (`topospace-mcp-<version>-<system>.tar.gz`,
  `.zip` on Windows) and as a Docker image, under the End User License Agreement in
  `EULA.md`. The source is not published. The compiled build is a single replica
  (SQLite or memory); multi-replica operation needs PostgreSQL and Redis support that
  it does not include.
* The compiled build runs with sensible local defaults when nothing is configured (a
  throwaway key, a stub provider, in-memory state) and never in production.

## 2.17.1

* Renamed from TopoSpace Engine to **TopoSpace MCP**. The command and the
  `TOPOSPACE_*` settings are unchanged. The program does not yet implement the Model
  Context Protocol.
* The console connects by itself when run locally: `topospace serve` prints a link
  that carries the throwaway local key only (never a key you set).

## 2.17.0

* **Your own models and keys.** A catalogue file (`TOPOSPACE_CATALOG`) describes
  providers, models, prices, fallbacks and roles; keys come only from environment
  variables, and a literal key in the file is refused.
* **Anthropic and Gemini** are translated natively, besides OpenAI-compatible servers
  (local models, ChatGPT and others). Failover goes across vendors, each with its own
  model id. A request carrying its own provider key never fails over to another vendor.
* **Roles:** a request can name a pool of models to be planned over.
* `topospace models init | check | ping`.

## 2.16.0

* Context is written in an order chosen so prompt caches can reuse a prefix; the
  planner is told which model already holds which prefix; cached tokens are metered.
* Ledger totals are folded per hour, so they do not depend on arrival order.
* A latency ceiling (`topospace_max_latency_ms`) fails a request at once rather than
  making it wait for a slow model.
* Local servers' prompt caches are bounded and counted in the memory budget.
