# Quickstart

## 1. Get it

Download the archive for your system from **Releases** and compare its SHA-256 with
the `.sha256` file beside it. Unpack it; the program is `topospace` (a launcher beside the `bin/` folder that holds the compiled files).

The program is not code-signed. On macOS, allow it in System Settings > Privacy &
Security, or remove the quarantine flag: `xattr -dr com.apple.quarantine .` in the
unpacked folder. On Windows, SmartScreen may ask you to choose More info > Run anyway.

## 2. Run it

```bash
./topospace serve     # the API and the console, on :8000
./topospace chat      # talk to it in this terminal, no server
```

With nothing configured it uses a throwaway key and a stub provider that returns
canned text, so you can see the decisions it makes before you connect a real model.
`serve` prints a link: for the throwaway key it already carries the key, so the
console connects by itself.

## 3. Use your own models and keys

```bash
./topospace models init --out models.json
export OPENAI_API_KEY=...  ANTHROPIC_API_KEY=...  GEMINI_API_KEY=...
./topospace models check models.json
TOPOSPACE_CATALOG=models.json ./topospace serve
```

Edit `models.json` first: model ids, prices and capabilities are examples. See
[docs/CATALOG.md](docs/CATALOG.md). `topospace models ping models.json
--confirm-spend` sends one tiny real request per model to prove a key and a model id
work (a fraction of a cent).

## 4. Send a request

The API is OpenAI-compatible. With the official Python client:

```python
from openai import OpenAI
client = OpenAI(base_url="http://localhost:8000/v1", api_key="ts_local_quickstart_key")
reply = client.chat.completions.create(
    model="auto",   # let the plane choose, or name a model from the catalogue
    messages=[{"role": "user", "content": "How are refunds handled?"}],
    extra_body={
        "topospace_context": [{"id": "policy", "text": "Refunds are issued within 14 days..."}],
        "topospace_role": "writing",   # only if your catalogue defines roles
    },
)
print(reply.choices[0].message.content)
print(reply.model_extra["topospace"])   # the decision: model, path, budget, cost
```

`POST /v1/plan` takes the same body and explains the decision without calling a
provider or charging anything. All fields: [docs/API.md](docs/API.md).

## 5. Your own key

For anything but a local trial, set your own key and pepper, and read
[docs/DEPLOYMENT.md](docs/DEPLOYMENT.md):

```bash
# a key looks like ts_<environment>_<id>_<secret>, with the secret random:
export TOPOSPACE_API_KEY=ts_dev_main_$(openssl rand -hex 16)
export TOPOSPACE_KEY_PEPPER=$(openssl rand -hex 16)      # at least 16 bytes
./topospace check       # validates the environment
```
