# TopoSpace MCP

**An AI execution optimization plane.** For every request it decides, as one joint
decision, *which context is exposed*, *which model runs*, *which execution path is
taken*, and *how many tokens are spent*, across the models you configure: on your
own machine, ChatGPT, Claude, Gemini, or anything that speaks the OpenAI API, each
called with **your own API key**.

This repository is the distribution of the compiled program: binaries, documentation
and the licence. The source is not published.

* **Use your own models.** Describe providers, models, prices, fallbacks and roles in
  a file; keys come from environment variables and never from the file.
  [`docs/CATALOG.md`](docs/CATALOG.md)
* **Fail over across vendors.** If a model's provider fails, the next model in its
  fallback chain answers on its own provider, and the ledger records which one did.
* **Run a model larger than memory.** A 17 GiB mixture-of-experts model ran on a 16 GB
  laptop with its experts streamed from disk, slowly: 1.5-2.7 tokens/s.
* **Explain and replay.** A dry run explains a decision without calling a provider;
  every input to a decision is recorded so it can be replayed exactly.
* **Meter honestly.** A hash-chained usage ledger in integer micro-USD, from
  provider-reported tokens.
* **Look at it.** A web console shows the topology a decision was made over.

What has been measured and what has not: [`docs/MEASUREMENTS.md`](docs/MEASUREMENTS.md).
No saving has been measured on a hosted provider's pricing, and none is claimed.

## Install

Download the archive for your system from **Releases**, check it against the
published SHA-256, and unpack it:

```bash
tar xzf topospace-mcp-<version>-macos-arm64.tar.gz
cd topospace-mcp-<version>-macos-arm64
./topospace serve
```

Or run the Docker image as a service: [`docs/DEPLOYMENT.md`](docs/DEPLOYMENT.md).

With no configuration it runs for local use with a throwaway key and a stub provider,
so the first command does something, and prints a link to the console. The program is
not code-signed; [`QUICKSTART.md`](QUICKSTART.md) says what your system will ask.

## Use your own models

```bash
./topospace models init --out models.json     # edit it
export OPENAI_API_KEY=...  ANTHROPIC_API_KEY=...  GEMINI_API_KEY=...
./topospace models check models.json
TOPOSPACE_CATALOG=models.json ./topospace serve
```

Any OpenAI client works against it: set its base URL to `http://localhost:8000/v1`.
A request may name a **role**, a pool of models you consider fit for that kind of
work, and the planner still trades quality against cost inside it. Planning a request
without sending it, against `examples/models.example.json` (a dry run calls no
provider, so placeholder keys are enough):

```
role chat            -> cheap-hosted  via openai/gpt-4o-mini     budget  200 tok  predicted $0.000088,  1609 ms
role writing         -> cheap-hosted  via openai/gpt-4o-mini     budget  604 tok  predicted $0.000295,  4935 ms
role hard-reasoning  -> gemini-pro    via gemini/gemini-2.5-pro  budget  789 tok  predicted $0.005703, 12260 ms
```

The capabilities in that file are placeholders: set your own from your own evaluation.

## Documentation

* [`QUICKSTART.md`](QUICKSTART.md): install, run, first requests
* [`docs/CATALOG.md`](docs/CATALOG.md): your models, keys, roles and fallbacks
* [`docs/API.md`](docs/API.md): the HTTP API
* [`docs/DEPLOYMENT.md`](docs/DEPLOYMENT.md): running it as a service
* [`docs/RUNBOOK.md`](docs/RUNBOOK.md): what each alert means
* [`docs/SECURITY.md`](docs/SECURITY.md), [`SECURITY.md`](SECURITY.md): trust boundaries and reporting
* [`docs/MEASUREMENTS.md`](docs/MEASUREMENTS.md): what has been measured, and what has not
* [`papers/`](papers/): the whitepaper
* [`EULA.md`](EULA.md), [`CHANGELOG.md`](CHANGELOG.md)

## Licence

Proprietary. The compiled program is licensed under [`EULA.md`](EULA.md): you may use
it; you may not reverse engineer it, redistribute it, or offer it as a service.
Third-party notices are inside each archive. © 2026 Caleb Lee, J1729 Labs.

Contact: director@j1729labs.online
