---
title: "Gateway"
weight: 20
---

# gateway {#gateway}

An OpenAI-compatible gateway in pure Go (`CGO_ENABLED=0`) that sits in front of
[vLLM](https://docs.vllm.ai) on a GPU cluster shared by several institutions.
It replaces LiteLLM for that role and adds fair-share scheduling with
reservation and borrowing, mapped onto vLLM's own priority scheduler.
[Repository →](https://github.com/go-llmflow/gateway)

## Status {#status}

**The current release is v0.8.0.** The versions so far:

| Version | Content |
| --- | --- |
| [v0.1.0](https://github.com/go-llmflow/gateway/releases/tag/v0.1.0) | virtual keys, teams and budgets in PostgreSQL, HCL2 configuration, admin API, accounting and CSV export, fair-share with borrowing |
| [v0.2.0](https://github.com/go-llmflow/gateway/releases/tag/v0.2.0) | **governance (roadmap milestone M1)**: the hierarchy organisation → team → project → key, roles, budgets on every scope, tags, soft-budget alerts, spend reports. **Breaking**, see below |
| [v0.3.0](https://github.com/go-llmflow/gateway/releases/tag/v0.3.0) | **security**: per-tenant prefix-cache isolation. **Breaking**: a new mandatory key file, see [Security](#security) |
| [v0.4.0](https://github.com/go-llmflow/gateway/releases/tag/v0.4.0) | **identity and access (roadmap milestone M2)**, built on [go-authn](https://github.com/go-authn): OpenID Connect tokens on the inference API (`jwt`), administrator single sign-on (`sso`), SAML federations through a bridge, SCIM provisioning (`scim`), key rotation (`key_rotation`), trusted proxies, IP allowlists, route restrictions, secrets from OpenBao or Vault. See [Identity]({{< relref "identity.md" >}}) |
| [v0.5.0](https://github.com/go-llmflow/gateway/releases/tag/v0.5.0) | **audit and logging (roadmap milestone M3)**: a hash-chained admin audit log with signed Ed25519 checkpoints and external anchors, per-team request logs (content off by default), retention, request policies, retried alerts. See [Audit and logging]({{< relref "audit.md" >}}) |
| [v0.6.0](https://github.com/go-llmflow/gateway/releases/tag/v0.6.0) | **guardrails (roadmap milestone M4)**: checks on prompts and completions, `hide_secrets`, `pii` (built on [go-llmflow/pii]({{< relref "pii.md" >}})), Llama Guard and Prompt Guard; the HTTP adapter is off by default. See "Guardrails" in the repository's README |
| [v0.7.0](https://github.com/go-llmflow/gateway/releases/tag/v0.7.0) | **more endpoints (issue #38)**: stateless `/v1/responses`, the Anthropic `/v1/messages`, rerank, tokenization, audio transcription, model fallbacks, OpenTelemetry traces. See [More endpoints](#more-endpoints) and [Coding agents]({{< relref "agents.md" >}}) |
| [v0.8.0](https://github.com/go-llmflow/gateway/releases/tag/v0.8.0) | **admin web UI and MCP gateway (issue #38, items 6 and 7)**. See [Admin web UI](#admin-ui) and [MCP gateway](#mcp) |

```sh
go install github.com/go-llmflow/gateway/cmd/llmflow-gateway@latest
llmflow-gateway check-config -c examples/gateway.hcl   # validates the file and every secret file it names
llmflow-gateway serve -c examples/gateway.hcl
llmflow-gateway export -c examples/gateway.hcl --from 2026-10-01 --to 2026-11-01 > october.csv
llmflow-gateway export -c examples/gateway.hcl --from 2026-10-01 --to 2026-11-01 \
  --group-by org,team,project --format parsable2   # side by side with sreport --parsable2
```

The binary `llmflow-gateway` has four subcommands: `serve`, `export`,
`check-config` and `version`. `export` takes `--group-by` and
`--format json|csv|parsable2`.

### Upgrading {#upgrading}

- **From v0.1 to v0.2.** Shares are now per organisation: `team "x" { share }`
  becomes `org "x" { share = ... team "lab" {} }`. In the admin API,
  `PUT /admin/budgets` takes `{"kind","name","model","period","max_cost","soft_limit"}`
  and `DELETE` takes `?kind=&name=&model=`. The fair-share metrics carry the label
  `org`.
- **From v0.2 to v0.3.** `cache_salt_key_file` is mandatory unless
  `cache_isolation = "none"`. The first start with a new key, and every key
  rotation, leaves every tenant with a cold prefix cache.

## Architecture {#architecture}

```text
clients (OpenAI SDKs, Authorization: Bearer llmf-...)
   |
   v
llmflow-gateway
   1 auth: virtual key -> team (SHA-256, constant-time, cached)
   2 body <= max_body_bytes, JSON re-encoded, client priority fields stripped,
     per-tenant cache_salt set
   3 budget (GPU-hours per day / week / month), fail-closed by default
   4 per-key RPM / TPM / max in-flight
   5 fair share: within reservation -> priority 0/1, borrowing -> 10 and more
   6 route: least in-flight healthy upstream, retry before the first byte
   7 stream SSE event by event, capture usage
   8 cost -> budget counter, fair-share usage, async PostgreSQL record
   |
   v  headers built from scratch: Authorization (internal key), X-Vllm-Priority
vLLM x N  (--scheduling-policy priority --api-key ..., reachable only from the gateway)
```

Packages: `fairshare` (the algorithm, no I/O), `openai` (request rewrite, SSE
usage capture), `store` (memory and PostgreSQL), `gateway` (HTTP server),
`config`, and `cmd/llmflow-gateway` (cobra command line).

### Hierarchy and roles {#hierarchy}

The tree mirrors a Slurm account tree: **organisation** (institution) →
**team** (lab) → **project** → **key**. Budgets can be set on every scope
(organisation, team, project, key, tag), and for one model group. Administration
is delegated by role:

| Role | Scope | May |
| --- | --- | --- |
| `proxy_admin` | everything | everything the master key may |
| `proxy_viewer` | everything | read every listing, report and fair-share state |
| `org_admin` | one organisation | manage its teams, projects, keys, budgets and users below it |
| `team_admin` | one team | manage its projects and keys, grant `member` and `viewer` on the team |
| `member` | one team | create keys (owned by the member), list and revoke their own keys |
| `viewer` | one organisation or team | read its teams, projects, keys (no secrets), budgets and spend |

Users authenticate with admin tokens (`llma-...`, 256 random bits, stored as
SHA-256, shown once). Nobody can grant a role above their own scope. Teams can
restrict which model groups they may use.

### Routes {#routes}

Client routes: `POST /v1/chat/completions`, `/v1/completions`,
`/v1/embeddings`, `/v1/responses` (stateless only), `/v1/messages` (Anthropic
Messages API), `/v1/rerank`, `/v2/rerank`, `/v1/audio/transcriptions`,
`/tokenize`, `/detokenize`, and `GET /v1/models`. Also `GET /healthz`, which
returns 503 when the budget state is stale and fail-closed is on, and
`GET /metrics` (Prometheus text format, optional bearer token). All are
described under [More endpoints](#more-endpoints); how coding agents use them is
on the [Coding agents]({{< relref "agents.md" >}}) page.

### Admin API {#admin-api}

Admin routes take `Authorization: Bearer <master key>` or a user's `llma-...`
token.

| Route | Purpose |
| --- | --- |
| `GET /admin/whoami` | the caller and their roles |
| `GET /admin/login`, `/admin/callback`, `/admin/session`; `POST /admin/logout` | administrator single sign-on (see [Identity]({{< relref "identity.md" >}}#sso)) |
| `GET`, `POST /admin/users`; `DELETE /admin/users/{name}`; `.../tokens`; `.../roles` | users, their tokens and their roles; names may contain `@` and `+` |
| `GET`, `PUT /admin/allowlists` | IP allowlists on an organisation, a team or a key |
| `POST /admin/keys/{id}/rotate`; `PUT /admin/keys/{id}/rotation` | rotate a key's secret now, or on a schedule |
| `/scim/v2` | SCIM 2.0 provisioning, with a `scim` block |
| `GET`, `POST /admin/orgs`; `DELETE /admin/orgs/{name}` | organisations (one created here has no share: it only borrows) |
| `GET /admin/teams?org=`, `POST /admin/teams`; `DELETE /admin/teams/{name}` | teams |
| `GET`, `POST /admin/teams/{team}/projects`; `DELETE .../projects/{name}` | projects |
| `GET`, `PUT /admin/teams/{team}/models` | the model groups a team may use (empty list = every configured model) |
| `GET /admin/keys?team=`, `POST /admin/keys`; `DELETE /admin/keys/{id}` | keys (the `llmf-...` secret is returned once); a key may carry `tags` |
| `GET`, `PUT /admin/budgets`; `DELETE /admin/budgets?kind=&name=&model=` | budgets: `{"kind","name","model","period","max_cost","soft_limit"}` with `period` `day`, `week` or `month` |
| `POST /admin/budgets/increase` | a temporary increase with an expiry |
| `GET /admin/spend?from=&to=&group_by=&format=` | spend report grouped by any of `org`, `team`, `project`, `key`, `model`, `pool`, `tag`; `json`, `csv` or `parsable2`; filtered to what the caller may see |
| `GET /admin/usage.csv?from=&to=` | the v0.1 export |
| `GET /admin/fairshare` | reservations, in-flight, borrowing, usage, level fairshare and rank per pool and organisation |

A request is refused with 429, naming the scope, as soon as one budget it falls
under is exhausted. `soft_limit` triggers one alert per budget and period, to a
webhook and/or by e-mail through an SMTP relay.

## Configuration (HCL2) {#configuration}

The configuration is written in
[HCL2 native syntax](https://github.com/hashicorp/hcl/blob/main/hclsyntax/spec.md);
the repository's `examples/gateway.hcl` is parsed by its test suite, so it stays
valid. This extract was run through `llmflow-gateway check-config` v0.8.0 (with
the key files present):

```hcl
master_key_file     = "/etc/llmflow/master.key"
cache_salt_key_file = "/etc/llmflow/cache-salt.key"

org "etabA" {
  share = 0.8125
  team "lab-nlp" {}
  team "lab-vision" {}
}
org "etabB" {
  share = 0.1875
  team "lab-chem" {}
}

pool "b300" { capacity = 256 }

model "llama-3.3-70b" {
  upstream_model = "meta-llama/Llama-3.3-70B-Instruct"
  pool           = "b300"
  class          = "interactive"
  key_file       = "/etc/llmflow/vllm.key"
  cost {
    input_per_mtok  = 0.02
    output_per_mtok = 0.20
    cached_per_mtok = 0.002
  }
  upstream "gpu01" { base_url = "http://10.0.0.11:8000" }
  upstream "gpu02" {
    base_url = "http://10.0.0.12:8000"
    key_file = "/etc/llmflow/gpu02.key"
  }
}
```

| Block or attribute | Meaning |
| --- | --- |
| `org "<name>" { share, team "<name>" {} ... }` | an organisation (institution) with the fraction of every pool reserved for it (sum at most 1) and the teams declared under it; team names are global |
| `pool "<name>" { capacity }` | `C`, the sustainable concurrency of the pool (1 to 1048576) |
| `model "<name>" { ... }` | client model name; `upstream_model`, `pool`, `class` (`interactive` or `batch`), `key_file`, `cost { input_per_mtok, output_per_mtok, cached_per_mtok }` in GPU-hours per million tokens, and `upstream "<name>" { base_url, key_file }` blocks |
| `store { ... }` | `kind` (`memory` or `postgres`), `dsn_file`, `fail_closed` (default true), `queue_size`, `batch_size` |
| `fairshare { half_life }` | usage decay half-life (default `"168h"`) |
| `limits { ... }`, `health { ... }` | body and header sizes, timeouts, maximum request duration and response size; health-check interval, timeout, cooldown and path |
| `tags { header, allowed }` | the request header clients may tag requests with, and the allowed tags |
| `cache_isolation`, `cache_salt_key_file` | prefix-cache tenant scope (`org` default, `team`, `key`, `none`) and the HMAC key file (at least 32 bytes; required unless `none`) |
| `jwt { issuer, audience, type, rpm, map { claim, value, team, project } }` | OpenID Connect access tokens on the inference API; see [Identity]({{< relref "identity.md" >}}#jwt) |
| `sso { issuer, client_id, client_secret_file, redirect_url, scopes, session_ttl, role { claim, value, role, team } }` | administrator single sign-on |
| `scim { token_file, base_url, group "<name>" { team, role } }` | SCIM 2.0 provisioning |
| `vault { address, token_file, ca_file, namespace, cache_ttl }`, `key_rotation { vault_prefix, check_interval }` | OpenBao or Vault secret references (`vault:<mount>/<path>#<field>`), and scheduled key rotation |
| `trusted_proxies`, `routes { inference, admin, metrics, healthz { allow = [...] } }` | reverse proxies whose `X-Forwarded-For` is read, and the networks allowed per route class |
| `audit { signing_key_file, checkpoint_interval, anchor, anchor_max_lag, retention, prune_interval }`, `export "<name>" { kind, endpoint, bucket, ... }` | the admin audit log and where its checkpoints, and the request logs, are exported; see [Audit and logging]({{< relref "audit.md" >}}) |
| `logging "<name>" { teams, destination, content, ... }` | per-team request logs; content is off by default |
| `policy "<name>" { teams, require, require_tags, max_body_bytes, max_response_bytes }` | required fields and tags, and per-team or per-key size limits (`POST /admin/keys {"policy"}`) |
| `alerts { webhook_url, webhook_token_file, smtp { server, from, to, username, password_file } }` | where soft-budget alerts go |
| `auth_failures_per_minute` | failed authentications allowed per client network before a 429 (default 60, 0 disables) |

Design decisions of the format:

- **Static evaluation.** The file is decoded with no evaluation context: no
  variables, no functions. `env.HOME`, `file("/etc/shadow")` or `"${x}"` are
  errors, so a configuration cannot read the environment or other files.
- **Quoted strings are templates.** A literal `${` must be written `$${`, and a
  literal `%{` must be written `%%{`. This applies to every value, URLs, paths
  and model names included.
- **Labelled blocks** for named things: `org`, `team`, `pool`, `model`,
  `upstream`. Duplicate labels are errors. Teams nest inside their organisation.
- **Every diagnostic carries `file:line:column`**; `check-config` prints them one
  per line.
- **Secrets only by file** (`master_key_file`, `metrics_token_file`,
  `store.dsn_file`, `key_file`, `cache_salt_key_file`, and so on). An upstream URL
  with credentials is rejected.
- **Durations are strings** in Go syntax (`"30s"`, `"168h"`). Absent and zero
  differ: `retries = 0` disables retries, leaving it out takes the default.
- The parser is fuzzed (`FuzzParse`): the configuration is treated as untrusted
  input.

Cost of a request, in GPU-hours:
`((prompt - cached) x input + cached x cached_price + completion x output) / 10^6`,
from the `usage` vLLM reports.

## More endpoints {#more-endpoints}

All of these go through the same pipeline as chat: authentication, policies,
budgets, rate limits, guardrails, fair-share priority and the tenant's
`cache_salt`.

### Responses API: stateless only {#responses}

`POST /v1/responses` is proxied to vLLM's own Responses endpoint. The gateway
**forces `"store": false`** and refuses `previous_response_id`, `background`,
`conversation` and a stored `prompt` with a 400 that says so. `GET /v1/responses/{id}`
and the cancel route are not routed. Clients send the whole conversation in
`input`, as with chat completions.

**Why.** vLLM keeps every response in the memory of the instance that produced it
(`store` defaults to true there). Anyone holding the id can read it back, continue
it or cancel it. Behind a gateway that state would be shared by every tenant and
would live on one upstream among several, so **a tenant holding an id could read or
continue another tenant's conversation**.

Usage is read from the Responses usage, including the final `response.completed`
event of a stream. Guardrails read `instructions` and the `input` items; input
items they cannot read are refused. A streamed Responses request under
**post-call** guardrails is refused (400; ask with `"stream": false`).

### Anthropic Messages API {#messages}

`POST /v1/messages` is proxied to **vLLM's own Anthropic endpoint**, with no
translation in the gateway, so coding agents that speak the Messages API can use
local models. A virtual key is the bearer token. Guardrails cover `system`, text
and `thinking` blocks, `tool_result` content and every string inside `tool_use`
inputs; blocks they cannot read (a `document`) are refused. Guarded streams pass
through an **Anthropic event filter**. Errors carry both OpenAI's `{"error": ...}`
and Anthropic's top-level `"type": "error"`. `/v1/messages/count_tokens` is not
routed.

### Rerank, tokenization, audio {#rerank}

- `POST /v1/rerank` and `/v2/rerank` serve score models; they are guarded and
  charged on the usage's prompt tokens.
- `POST /tokenize` and `/detokenize` are authenticated and rate limited but
  **not charged and not guarded**: no model runs, and masking would make the count
  wrong.
- `POST /v1/audio/transcriptions` takes a bounded multipart form, rebuilt from an
  allow-list of fields before it is forwarded. The file is limited by
  `limits { max_audio_bytes }` (25 MiB by default), and a model is charged with
  `cost { audio_per_hour }` in GPU-hours per hour of audio. The audio itself is not
  inspected, and the content log never holds it.
- A client `chat_template` is stripped from every request.

### Fallbacks {#fallbacks}

A model group can name fallbacks, tried in order (at most 4, not a guard model, not
itself):

```hcl
model "llama-70b" {
  pool      = "b300"
  fallbacks = ["llama-8b"]
  # ...
}
```

Before anything has reached the client, a request moves to the next fallback when
no upstream of the group took it, or when vLLM answered 429 or 503. The target must
be in the team's model list and within its budgets. The request acquires in the
**target's pool** (so it gets its organisation's priority there), is rewritten for
the target, and is **charged and recorded under the model that served it**. The
response carries `X-Llmflow-Model`, and `llmflow_fallbacks_total{from,to}` counts
the moves. Any other answer, a 400 for instance, is final.

### Traces {#traces}

A `tracing` block exports OpenTelemetry traces over OTLP/HTTP or gRPC. The W3C
`traceparent` of the upstream span is sent to vLLM, so vLLM's own spans join the
trace. A client's trace context is **restarted** unless `trust_client_context =
true`. Spans carry metadata only (organisation, team, key id, model, pool, priority,
status, token counts, guardrail verdicts), **never a prompt or an answer**.

This block, with the fallbacks and audio settings above, passes
`llmflow-gateway check-config` v0.8.0 (with the key files present, and
organisations, pools and models around it):

```hcl
limits {
  max_audio_bytes = 26214400
}

tracing {
  endpoint             = "https://otel.example.org:4318"
  protocol             = "http"
  sample_ratio         = 0.1
  token_file           = "/etc/llmflow/otel.token"
  insecure             = false
  trust_client_context = false
}

model "whisper" {
  upstream_model = "openai/whisper-large-v3"
  pool           = "small"
  class          = "interactive"
  key_file       = "/etc/llmflow/vllm.key"
  cost {
    audio_per_hour = 0.05
  }
  upstream "gpu06" { base_url = "http://gpu06.internal:8000" }
}
```

## Admin web UI {#admin-ui}

With an `sso` block, `/ui/` serves a web console for administrators
(`/admin/login?next=ui` signs in and lands there). It is **server-rendered HTML
with no JavaScript**, under a strict policy on every `/ui/` response:

```text
Content-Security-Policy: default-src 'none'; style-src 'self'; img-src 'self'; form-action 'self'; frame-ancestors 'none'; base-uri 'none'
```

plus `X-Content-Type-Options: nosniff`, `Referrer-Policy: no-referrer` and
`Cache-Control: no-store`. The one stylesheet is served by the gateway; no
third-party asset is loaded.

| Page | What it does |
| --- | --- |
| Keys | list per team; create (the secret is shown **once**, on a page that is not cached), rotate with a grace period, revoke |
| Budgets | budgets with spent and limit; set and delete |
| Spend | GPU-hours, requests and tokens over a period, grouped by organisation, team, project, key, model, pool or tag |
| Fair share | live state per pool: reservation, in flight, borrowing, level fairshare and rank |
| Audit | entries page by page, and a button that verifies the chain (status of the checkpoints and of the anchor) |

The UI holds no privilege of its own: every page calls the admin API's handlers
in-process with the signed-in person, so it has the **same RBAC as the API** (an
organisation administrator sees their organisation only) and every change is
audited as through the API. Only an SSO session is accepted; without an `sso`
block `/ui/` answers 404. Two admin routes were added for it:
`GET /admin/audit?from=&limit=` and `POST /admin/audit/verify` (proxy admins and
viewers only). The verification uses the gateway's own key; for an independent
check use `llmflow-gateway audit verify` with a public key kept elsewhere (see
[Audit and logging]({{< relref "audit.md" >}}#verifying)).

## MCP gateway {#mcp}

`POST /mcp` is an [MCP](https://modelcontextprotocol.io/specification/2026-07-28)
server in front of the tool servers of the configuration. It speaks the Streamable
HTTP binding of revision **2026-07-28** and, for older clients, the initialize
handshake of **2025-11-25** (and 2025-06-18, 2025-03-26). A client authenticates
with its virtual key. This block passes `llmflow-gateway check-config` v0.8.0
(with the key files present, and an `sso` block, organisations, a pool and a model
around it):

```hcl
mcp_server "github" {
  url                = "https://mcp.example.org/github/mcp"
  token_file         = "/etc/llmflow/mcp-github.token"
  protocol           = "2026-07-28"
  timeout            = "60s"
  max_response_bytes = 8388608
}
```

- **Tools are named `<server>.<tool>`.**
- **Per-team allow-lists, closed by default.** A team with no list sees no tool.
  An organisation administrator sets it with `PUT /admin/teams/{team}/tools`
  `{"tools":["github.*","docs.search"]}` (a pattern is `*`, `<server>.*` or
  `<server>.<tool>`, and must name a configured server). Team administrators can
  read the list but cannot widen it. A call to any other tool answers
  `-32602 Unknown tool`, like a tool that does not exist, and the refusal is
  audited (`tool.denied`).
- **Every call is audited before it is made.** A `tool.call` entry names the key,
  its organisation, team and project, the tool, and the SHA-256 and size of the
  arguments; **the arguments are hashed, never stored**. If that entry cannot be
  written, the call is not made (503). A `tool.result` entry follows with the
  outcome. `llmflow_tool_calls_total{team,server,status}` counts the calls, and the
  audit retention accepts the kind `tool`.
- **Only configured servers are reached.** Clients name tools, never URLs; the
  gateway sends requests only to the `url` of an `mcp_server` block, follows no
  redirect, uses no environment proxy and never fetches what a server returns. The
  tool servers' tokens come from the secret source and the client's key is never
  forwarded.
- Tool calls count against the key's RPM and in-flight limits and cost no
  GPU-hours.

**Not covered:** OAuth 2.1 discovery (virtual keys are static bearer tokens),
resources and prompts, **guardrails on tool arguments and results**, and SSE toward
MCP clients.

## How fair-share with borrowing works {#fair-share-with-borrowing}

A pool has a capacity `C`: the concurrency it sustains without preemption,
measured externally. Each **organisation** owns an integer reservation, shared by
all its teams and projects, obtained by apportioning `C` over the shares with the
largest-remainder (Hamilton) method, so reservations never sum to more than `C`
(C = 256 gives A 208 and B 48).

For every request:

| Situation | Priority sent to vLLM |
| --- | --- |
| the organisation has fewer within-share requests in flight than its reservation | `0` interactive, `1` batch |
| otherwise (borrowing) | `10 + 10 x rank + class`, rank at most 9 |

Among borrowers, `rank` is the dense rank of the organisation's **level
fairshare** `LF = S / U` (Slurm Fair Tree): the most under-served has rank 0, so
the most over-served borrower gets the largest number, is admitted last and is
preempted first. Usage is the **cost actually consumed**, charged when a request
finishes and decayed with a half-life (the exact exponential, of which Slurm's
`1 - 0.693/hl` is the first-order expansion).

vLLM's v1 scheduler keeps waiting requests in a min-heap on
`(priority, arrival_time)`; when a running request runs out of KV blocks, the
running request with the largest `(priority, arrival_time)` is preempted; a
waiting request never preempts a running one. A within-share request is therefore
admitted before any borrower and is never the victim while a borrower runs. The
[guide]({{< relref "guides/fairshare/inference.md" >}}) explains vLLM's
priority semantics from its source.

Two choices were forced by the simulator: the reservation test counts
**within-share** requests in flight (not all requests), and usage is **charged at
release** (not from time in flight, since a request waiting in vLLM's queue
consumes nothing).

| Scenario (C = 16, 64 clients per team) | Result |
| --- | --- |
| A alone, B alone | KV utilisation 0.98 and 0.97; 40.5 and 39.0 running, for reservations of 13 and 3 |
| both saturating, B arriving after A filled the pool | running fraction A 0.789, B 0.211 (shares 0.8125 and 0.1875) |
| preemptions | 24,614, every one a borrower |
| control: every request at priority 0 | B gets 0.498, its fraction of clients, not its share |

## Overhead {#overhead}

An independent A/B against LiteLLM, measured on **2026-10-23**.

**Protocol.** A Go tool, `gwbench`, with tests; its control test sees a planted
3 ms delay. Requests go one at a time, the order of the targets is randomised at
every round, and the overhead is paired round by round against a direct call to
the same fake upstream.

**Targets.** LiteLLM 1.106.2 (one process, no database) and llmflow v0.1.0
(`GOMAXPROCS=1`, memory store), on a small non-streaming request.

**Load.** Each run of 2,000 rounds starts only at a 1-minute load of 4 or less,
and is discarded if the load ends above 4. A first long run was discarded (the load
rose to 9.44). Eight runs were kept, at loads 3.30 to 3.94.

**Result**, over 16,000 paired rounds:

| Overhead | p50 | p90 | p99 |
| --- | --- | --- | --- |
| LiteLLM | 4.86 ms | 5.26 ms | 6.21 ms |
| llmflow | 117 us | 161 us | 239 us |

That is about **40 times lower at the median and 26 times at p99**. The per-run
median varied from 4.84 to 4.93 ms for LiteLLM and from 115 to 120 us for llmflow.

**What it does not measure.** No streaming, no concurrency, and no database write:
the gateway ran with its memory store. These are to be measured on the cluster.

## Security {#security}

- **Clients cannot choose their priority.** The body is parsed and re-encoded
  (duplicate keys and escaped spellings cannot survive); `priority`, `extra_body`,
  `request_id`, `kv_transfer_params`, `ec_transfer_params` and `vllm_xargs` are
  removed. Upstream headers are built from scratch, so a client's
  `X-Vllm-Priority`, `x-pass-*`, `Authorization` and `X-Forwarded-*` never reach
  vLLM. This closes a hole
  [measured on LiteLLM]({{< relref "guides/fairshare/inference.md" >}}#what-litellm-forwards).
  The rewrite and the SSE copier are fuzzed (`FuzzRewrite`, `FuzzCopySSE`).
- **vLLM must only be reachable from the gateway.** Its `--api-key` covers only
  `/v1`, `/v2` and `/inference`; `/invocations` exposes inference without
  authentication. Firewall the vLLM ports
  ([guide]({{< relref "guides/fairshare/inference.md" >}}#securing-vllm)).
- **Identity (v0.4.0).** OpenID Connect access tokens are verified by
  go-authn/oidc (no `alg: none`, no HMAC, `aud` and `exp` required, https key
  fetches rate-limited), administrator sessions use a `__Host-` cookie with CSRF
  token and Origin check, and every refusal on the inference API is the same 401.
  `X-Forwarded-For` is read from the right and only from `trusted_proxies`; routes
  can be restricted to networks; IP allowlists must all match. Details on the
  [Identity]({{< relref "identity.md" >}}) page.
- **Audit (v0.5.0).** Admin actions are in a hash-chained log with signed
  checkpoints, which an anchor on write-once storage makes hard to rewrite. What
  stays open is stated on the [Audit and logging]({{< relref "audit.md" >}}#threat-model)
  page.
- **Keys.** Virtual keys and admin tokens are 256-bit random secrets, stored as
  SHA-256, compared in constant time and shown once. Secrets come from files only
  and never appear in logs, errors or listings.
- **No SSRF.** Upstreams come from the configuration only; client input selects a
  model group, never a URL. Upstream redirects are not followed.
- **Brute force.** Failed authentications are limited per client network (IPv4
  address or IPv6 /64, from the connection, not from proxy headers) by
  `auth_failures_per_minute`; valid and missed keys live in two separate bounded
  caches, so a spray of random keys cannot evict valid ones.
- **Limits** on body, headers, timeouts, request duration and upstream response
  size; a bounded accounting queue.
- **Budgets fail closed**: when PostgreSQL cannot be read for 3 refresh periods,
  every request gets 503 (`fail_closed = false` to fail open).
- **CI**: race tests on amd64 and arm64, PostgreSQL integration tests, fuzz smoke,
  `govulncheck`, `CGO_ENABLED=0` cross-builds.

### Prefix-cache isolation (v0.3.0) {#prefix-cache-isolation}

**Why.** vLLM reuses cached KV blocks across requests that share a prompt prefix.
That reuse is a **timing side channel**: a caller can test whether a guessed
prefix is already cached by measuring the time to first token, and so learn what
another tenant sent. vLLM's [security guide](https://docs.vllm.ai/en/stable/usage/security/)
documents it, and it is tracked as CVE-2025-46570. On a cluster shared by several
institutions this is a leak between institutions.

**What the gateway does.** It removes any `cache_salt` the client sends and
**always sets its own**: `base64url(HMAC-SHA256(key, scope))`, 43 characters (256
bits). The key is read from `cache_salt_key_file`, and the scope is set by
`cache_isolation`:

| `cache_isolation` | One prefix cache per |
| --- | --- |
| `org` (default) | organisation (institution) |
| `team` | team |
| `key` | virtual key |
| `none` | nothing: a single tenant |

vLLM mixes the salt into the hash of the first KV block and every later block
chains from it, so different scopes never share cached blocks. The salt is
**stable** per scope (a random salt per request would disable prefix caching).
Multimodal `uuid` values in the message content parts are **rewritten per
tenant**, not stripped: a client can keep referring to media it sent earlier,
while the same uuid sent by two tenants names two different cache entries.

**Operating it.** `cache_salt_key_file` must hold at least 32 random bytes and be
readable by its owner only (mode 0600); it is mandatory unless
`cache_isolation = "none"`. **Rotating the key means a cold prefix cache for
everyone.**

**Residual risk.** Within one scope the cache is shared. With the default `org`,
users of the same institution can still probe each other by timing; `team` or
`key` narrow that, at the cost of less cache reuse. And vLLM itself states that
`cache_salt` is not an isolation boundary: tenants that need hard isolation need
**a vLLM instance of their own**.

### Security audit of 2026-10-26 {#audit-2026-10-26}

A security audit and a chaos run of v0.8.0 were made on 2026-10-26. The fixes
below are merged on `main` **after the v0.8.0 tag**, so they are not in a release
yet (the latest tag is v0.8.0); each has a test that fails without the fix.

- **A1, multimodal `uuid`s on every route (medium).** They were scoped per tenant
  on chat completions and Responses only. On `/v1/rerank`, `/v2/rerank`,
  `/tokenize` and `/v1/embeddings` the client's uuid reached vLLM unchanged, so a
  tenant could refer to another tenant's cached media (and probe it through a rerank
  score or a token count). The walk now follows every shape vLLM accepts, and a fuzz
  target, `FuzzScopeEveryRoute`, checks every route.
- **A2, a client that never reads its answer (medium).** It held its in-flight slot,
  its fair-share grant and up to `max_response_bytes` of memory for ever. Every
  response now has a write deadline of `max_request_duration` plus 5 s.
- **A3, re-enabling a disabled user (high).** An organisation admin could re-enable
  a disabled `proxy_admin` by creating the user again. Only a proxy admin may now.
  A test, `TestAuthzMatrix`, covers every admin route against 12 principals; no
  other cross-organisation hole was found on the 39 routes.
- **A4, disabled teams (low).** A disabled team or project now disables its keys,
  including one created during the disable, and the cached JWTs of its people, at
  once.
- **A5, export queues (medium).** With the export target down and content logging
  on, up to 8 GiB of logs could pile up in memory. A queue is now also bounded to
  256 MiB, and what is dropped is counted and alerted.
- **A6 and A7, store faults (medium and low).** Key lookups and tool-list reads give
  up after 5 s (503), a budget refresh after 30 s, and admin, UI and SCIM requests
  after 60 s, instead of hanging when PostgreSQL is partitioned. Two replicas can
  now start together on an empty database. Chaos tests cover PostgreSQL down,
  partitioned and restarted, an upstream restart, a key rotation under load, two
  replicas, and a clock stepping back. The loss of usage records during an
  accounting outage stays an open defect (limitation 6).

### Aborted requests {#aborted}

vLLM's usage chunk comes last. When the client leaves before it, the gateway
stops reading (vLLM is not kept generating for nobody) and charges an
**estimate**: prompt bytes / 4 plus the content deltas already forwarded. The
record is flagged, in an `estimated_requests` column of the spend reports and in
PostgreSQL. The estimate undercounts tool-call and reasoning tokens.

## Limitations {#limitations}

Open defects of v0.8.0, as the repository lists them:

1. **State is per process.** In-flight counts, rate limits and the key cache live
   in memory: run **one gateway replica per pool**. Several replicas would each
   grant the full reservation.
2. **Budgets lag.** The cost is known when a request ends, so a burst of long
   requests can overspend by what is in flight. Several gateways see each other's
   spend only at their next refresh.
3. **Priority is fixed at admission.** A borrowing request already queued in vLLM
   keeps its priority when its organisation drops back under its reservation, and
   teams inside an organisation share its reservation without sub-shares of
   their own. vLLM has no API to re-prioritise a queued request.
4. **Aborted requests are charged with an estimate** (see above).
5. **Prefix-cache isolation is per scope, not absolute** (see above).
6. **Accounting can drop records** when its queue is full or a batch write fails
   twice (both are counted in metrics); budgets count the cost until the next
   refresh only.
7. **No TLS** in the gateway: put it behind a TLS proxy. The **admin API shares
   the client listener**. `/v1/responses` is served
   statelessly only (see [above](#responses)).
8. **`C` is static configuration**; the gateway does not measure or adapt it.
9. **Admin principals are read from the store on every admin request**, with no
   cache: role changes apply at once, but the admin API fails with the store.
10. **Soft-budget alert retries live in memory.** Each channel (webhook, SMTP) is
    retried on its own, up to 6 attempts with exponential backoff; a 4xx other
    than 408 and 429 is not retried; every webhook attempt carries the same
    `Idempotency-Key`. A gateway stopped mid-retry does not resume: the period is
    already marked as alerted.
11. **SSO sessions live in one replica's memory**: behind several replicas, use
    sticky sessions.
12. **A person holding a JWT cannot be revoked individually**: disable the team,
    change the `map` blocks, or revoke at the identity provider; a token already
    verified stays cached until its `exp` or `key_cache_ttl`.
13. **SCIM does not expand nested groups**, and a provisioned role on a team that
    does not exist is logged, not created.
14. **The audit log's guarantees end at its newest anchor.** Without an anchor
    export, truncation, rollback and deletion of the database go undetected; with
    one, what happened after the newest exported checkpoint can still be removed,
    and a stolen signing key can sign new history after it. Keep the key in
    OpenBao or Vault, verify with a public key kept elsewhere, and keep an anchor
    on write-once storage. See the
    [threat model]({{< relref "audit.md" >}}#threat-model).
15. **What guardrails do not inspect.** Streamed tool-call arguments cannot be
    masked, so a guarded stream that carries them is stopped (ask without
    `stream`). Reasoning text, refusals, log probabilities and unknown fields are
    removed from guarded answers, not inspected. A secret whose start appears more
    than 256 bytes before it can match is not seen in a stream, and a span still
    open after 64 KiB is released masked only as far as it has reached. Guardrail
    audit entries are written asynchronously and dropped, counted and alerted when
    the queue is full. **A Responses stream under post-call guardrails is
    refused.**
16. **Phone numbers have no checksum**: a PII guardrail that masks them also masks
    10-digit references that start with 0.
17. **The performance of v0.6 and v0.7 is not measured yet.** The
    [overhead figures](#overhead) are those of v0.1.0; an independent benchmark of
    the current release is waiting for a quiet machine.

## Roadmap {#roadmap}

[Issue #7](https://github.com/go-llmflow/gateway/issues/7) tracks shipping every
LiteLLM Enterprise feature in the open-source gateway. **M1 (governance and
multi-tenancy) is done**, in v0.2.0, **M2 (identity and access)** in v0.4.0, and
**M3 (audit and logging)** in v0.5.0, and **M4 (guardrails)** in v0.6.0. The
version numbers written in the issue shifted by one when v0.3.0 was taken by the
security patch.

A further milestone, [issue #38](https://github.com/go-llmflow/gateway/issues/38)
("M5"), covers more endpoints and operations. Its first five items shipped in v0.7.0
(the endpoints, fallbacks and tracing described above), and the last two, the admin
web UI and the MCP gateway, in v0.8.0. **Issue #38 is closed and M5 is complete.**

## License {#license}

BSD-3-Clause.
