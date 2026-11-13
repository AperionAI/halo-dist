# Halo

Halo keeps a self-hosted AI agent's model bill under control. It runs on the
same machine as the agent, meters every model call by agent, model and task,
and stops the agent at a hard spend cap, even with no network. Paid tiers cut
the bill itself with caching and compression.

- **See the bill.** Spend per agent, per model and per task. The free tier
  also shows what Cut would have saved on your own traffic.
- **Stop runaway spend.** Local budgets with a hard-cap kill switch. A fresh
  install sets $25 soft and $50 hard per 24 hours, so you're covered before
  you touch the config.
- **Keep secrets out of the model.** Secret cloaking and credential-shape
  detection on MCP tool calls, shared with [Aperion Shield](https://github.com/AperionAI/shield).
- **Keep a record.** A hash-chained, tamper-evident local audit log.

With `relay_url` unset (the default), nothing leaves the machine except the
provider calls you already make. Budgets, the denylist, `halo report` and the
local dashboard all work offline.

Halo is small on purpose. It uses single-file local stores (no Redis, Mongo or
Postgres) and runs no model inside the process.

This repo holds the release binaries, install script, example config and docs.
Halo is distributed as binaries and container images under the
[Halo Binary License](LICENSE).

## Trust model

- **Provider API keys stay on your machine.** They live in your OS keychain
  (macOS Keychain, Linux keyutils, Windows Credential Manager), or in an
  Argon2id + XChaCha20-Poly1305 encrypted file on headless boxes. See
  [`docs/HEADLESS.md`](docs/HEADLESS.md).
- **The relay is optional.** If you point Halo at one, it receives metadata
  only: token counts, model, cost and cache-hit flags. Never prompts,
  responses, tool arguments or vectors. Model traffic never goes through the
  relay. The schema is in [`docs/TELEMETRY_SCHEMA.md`](docs/TELEMETRY_SCHEMA.md).

## Install

No GitHub account needed. See [`docs/INSTALL.md`](docs/INSTALL.md) for every
option, including an OpenClaw `docker-compose` recipe.

```bash
# macOS / Linux, arm64 or x64
curl -fsSL https://halo-get.aperion.ai | sh

# Shared install on a box where the proxy runs as a service user
HALO_INSTALL_DIR=/usr/local/bin sudo -E sh -c 'curl -fsSL https://halo-get.aperion.ai | sh'

# Docker. One image ships both `halo` and `halo-relay`
docker run --rm -v halo-data:/data -p 8787:8787 ghcr.io/aperionai/halo
# or: docker compose up -d halo   (using docker-compose.yml in this repo)
```

Windows: download the `.zip` from [Releases](https://github.com/AperionAI/halo-dist/releases).

Halo governs model calls. The agent can still run shell commands and write
files directly, so pair it with Shield:

```bash
curl -fsSL https://shield-get.aperion.ai | sh
aperion-shield --install-agent-hooks
```

## Quick start

```bash
# 1. Register an agent. This mints a virtual key and stores the real key in
#    your keychain. Paste the real key at the prompt so it stays out of
#    shell history.
halo agent add researcher --provider openai

# 2. Point your runtime at Halo instead of the provider.
export OPENAI_API_KEY=sf_live_researcher_...      # the virtual key printed above
export OPENAI_BASE_URL=http://127.0.0.1:8787/v1

# 3. Run the proxy.
halo serve
```

On an always-on box, install it as a service (`sudo halo service install`)
instead of leaving `halo serve` in a terminal.

Anthropic works the same way: `--provider anthropic`, then set
`ANTHROPIC_API_KEY` to the virtual key and `ANTHROPIC_BASE_URL=http://127.0.0.1:8787`.

OpenClaw and Nous Hermes keep their endpoint in their own config, not in
environment variables. Use `halo openclaw apply --agent <name>` or
`halo hermes apply --agent <name>` and restart the gateway. See
[`docs/OPENCLAW_INTEGRATION.md`](docs/OPENCLAW_INTEGRATION.md) and
[`docs/HERMES_INTEGRATION.md`](docs/HERMES_INTEGRATION.md).

Any OpenAI-compatible provider (Groq, Together, Fireworks, a local vLLM or
Ollama server) works with `--base-url`:

```bash
halo agent add fast-agent --provider openai \
  --key gsk_... --base-url https://api.groq.com/openai
```

Streaming responses pass straight through to the client as they arrive.

## Spend and savings

```bash
halo status      # live spend by agent and current caps
halo report      # local cost and savings view, works offline
halo report --format json --out savings.json
```

`halo serve` also starts a loopback-only dashboard at `http://127.0.0.1:8788`.
It shows spend, savings, requests, cache-hit rate and per-model breakdown,
lists agents with one-click revoke, and lets you edit settings. Anything that
writes needs a local token (`halo dashboard token`). Turn it off with
`dashboard.enabled: false`.

## Budgets and kill switch

Set caps in `~/.halo/config.yaml` (see [`config/halo.example.yaml`](config/halo.example.yaml)).
The hard cap is enforced locally, always, and doesn't depend on any network.
To stop an agent right away:

```bash
halo kill researcher   # revokes its key; the proxy refuses it at once
```

Long streams also get a mid-stream stop-loss on top of the pre-call check.
The built-in price table is an approximation. Override any price with
`price_overrides` in the config.

## MCP interception

List your MCP servers under `mcp_servers` and point your runtime at
`http://127.0.0.1:8787/mcp/<name>`. At that point Halo:

- **cloaks** secrets referenced in tool arguments as `{{cloak:NAME}}`,
  resolving them only on the copy sent upstream and scrubbing leaked values
  out of results;
- **flags** raw credentials going to a tool or echoed back by one, and records
  the kind (never the value) in the audit log.

## Compression and prompt caching (Cut)

Cut saves money even when Halo's own cache never hits:

- **Compression on the wire.** Wordy phrases are shortened and blank-line runs
  and trailing spaces are collapsed. Indentation is never touched, so pasted
  code and YAML stay intact. `compression.mode: aggressive` also shrinks large
  tool and MCP payloads, and keeps the original for the model to ask for back.
- **Anthropic `cache_control`.** Halo marks large or repeated blocks in the
  system prompt, tool definitions and attachments so Anthropic's prompt cache
  picks them up. OpenAI caches prompt prefixes on its own, and Halo reads the
  cached-token count from the response.

`halo report` splits savings into this baseline and the savings from Halo's
own cache.

## Semantic cache (Cut, off by default)

Exact-match caching only catches identical requests. The semantic cache asks
an embeddings API whether a similar prompt was answered before, so a question
asked twice in different words can be served from cache. No model runs
locally.

```yaml
semantic_cache:
  enabled: true
  provider: openai        # openai | ollama (self-hosted) | mock (offline dev)
  model: text-embedding-3-small
  similarity_threshold: 0.85
  max_entries: 500
```

```bash
halo embeddings set-key
```

- Every hit is re-checked against the similarity threshold. Below it, it's a
  miss.
- Answers are reshaped for the endpoint that asked, so a question answered
  through Anthropic can serve a later one routed to OpenAI.
- Tool calls, structured output and multi-turn conversations are never served
  from the semantic cache.
- The embedding call's own cost is shown separately in `halo report`.

## Relay (optional, paid)

Leave `relay_url` unset and nothing is uploaded. For a multi-device view:

```bash
HALO_RELAY_TOKEN=some-shared-token halo-relay --bind 127.0.0.1:8080
```

Then set `relay_url` and `relay_token` in `~/.halo/config.yaml`. The relay
dashboard can also revoke an agent across devices. That's a backstop. Each
machine's local kill switch works with no network.

## Per-channel attribution

When one runtime fans a single key out across many channels or sub-agents,
send `X-Halo-Subject: <channel>:<thread-or-user-id>` on each call. Halo
records it as metadata and rolls spend up by subject in `halo report`.

## Egress denylist and allowlist

Every outbound call Halo makes (provider, embeddings, relay) is checked first.
A starter denylist always applies: cloud metadata hosts, link-local metadata
IPs and a few paste sites. Deny wins over allow.

```yaml
egress:
  denied_upstreams:
    - "evil.example.com"
  allowed_upstreams:
    - "api.anthropic.com"
    - ".yourcompany.com"     # leading "." matches the domain and its subdomains
```

## Encryption at rest (off by default)

The two stores that hold response content on disk can be encrypted with
Argon2id + XChaCha20-Poly1305:

```bash
export HALO_VAULT_PASSPHRASE="correct horse battery staple"
```

```yaml
encrypt_at_rest: true
```

If it's on and the passphrase is missing, `halo serve` refuses to start rather
than run in plaintext. Embedding vectors and metadata stay unencrypted because
they can't be turned back into prompt or response text.

## AI usage registry

A metadata-only export of every agent, its provider and endpoint, status and
lifetime totals, plus the MCP servers Halo fronts (name and command only,
never `env`). It's the inventory a compliance review asks for.

```bash
halo registry export
halo registry export --format csv --out registry.csv
```

## Tiers

The free tier's safety features are never gated: budgets, the kill switch,
the denylist, MCP cloaking, the local audit log and `halo report`.

| Feature | Free | Cut $50 | Route $100 |
|---|---|---|---|
| Local budgets, hard-cap kill switch, `halo kill` | yes | yes | yes |
| Egress denylist and allowlist, MCP cloak, audit log | yes | yes | yes |
| Exact-match cache, compression, prompt-cache marking | shows savings | applies | applies |
| Local history | 7 days | 30 days | 90 days |
| Registered agents | 3 | unlimited | unlimited |
| Semantic cache | no | opt-in | opt-in |
| Per-agent caps, budget alert webhooks | no | yes | yes |
| Failover to a backup agent | no | no | yes |
| Task-class and effort routing | no | no | yes |

Hitting the 3-agent limit doesn't break anything already running.
`halo agent add` just won't mint a fourth key until you revoke one or upgrade.

A paid license is an offline, signed key. Halo never phones home to check it.

```bash
# Cut:   https://deploy.langsmart.app/halo/buy/cut
# Route: https://deploy.langsmart.app/halo/buy/route
halo license apply         # paste the token from the thank-you page
halo license show
```

A missing, invalid or expired key falls back to the free tier. It never stops
Halo from starting.

## License

Halo is distributed as binaries and container images under the
[Aperion AI Halo Binary License](LICENSE). Source code is not distributed.
