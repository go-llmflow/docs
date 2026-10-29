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

**The current release is v0.18.0.** The versions so far:

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
| [v0.9.0](https://github.com/go-llmflow/gateway/releases/tag/v0.9.0) | **security audit and chaos release** (2026-10-26): findings A1 to A9 fixed, among them a high one (A3). See [Security]({{< relref "security.md" >}}) |
| [v0.10.0](https://github.com/go-llmflow/gateway/releases/tag/v0.10.0) | **durable accounting. Breaking: `store.spool_dir` is mandatory with the postgres store.** `sso.min_refresh`; dependencies at the go 1.27.3 floor. See [Upgrading to v0.10.0](#upgrading-v0100) |
| [v0.11.0](https://github.com/go-llmflow/gateway/releases/tag/v0.11.0) | **fallback on saturation (#58)**: `fallback_on` and `saturation_waiting`. **Breaking for dashboards:** `llmflow_fallbacks_total` has a `reason` label. See [Fallbacks](#fallbacks) |
| [v0.12.0](https://github.com/go-llmflow/gateway/releases/tag/v0.12.0) | **Anthropic as an opt-in upstream, and cost units**: see [Anthropic as an upstream](#anthropic) and [Cost units](#cost-units). A **PostgreSQL migration** (a `unit` column) runs at start-up. `x-api-key` is accepted as a credential |
| [v0.13.0](https://github.com/go-llmflow/gateway/releases/tag/v0.13.0) | **finance reports**: [project attribution](#attribution) by header, the `lab` and `user` dimensions, the `/ui/reports` page, exports (CSV, XLSX, PDF) and reconciliation with Slurm's `sreport`. See [Finance reports]({{< relref "finance-reports.md" >}}). Existing configurations stay valid |
| [v0.12.1](https://github.com/go-llmflow/gateway/releases/tag/v0.12.1) | **`/v1/messages/count_tokens`** is routed: exact counts for vLLM models, the organisation's key for Anthropic. See [Anthropic Messages API](#messages) |
| [v0.13.1](https://github.com/go-llmflow/gateway/releases/tag/v0.13.1) | CLI export bounds (`--max-days`, `--max-rows`, `--timeout`, audited). **Published after v0.14.0, on a commit that already holds the HA code: see the note below** |
| [v0.14.0](https://github.com/go-llmflow/gateway/releases/tag/v0.14.0) | **high availability**: an optional `ha` block, several replicas sharing the pools through PostgreSQL. See [High availability](#ha). Off by default |
| [v0.14.1](https://github.com/go-llmflow/gateway/releases/tag/v0.14.1) | the same commit as v0.13.1 |
| [v0.15.0](https://github.com/go-llmflow/gateway/releases/tag/v0.15.0) | **Slurm discovery**: a `slurm` block inside a `model` follows vLLM jobs on Slurm (elastic pools). See [Slurm discovery](#slurm-discovery) |
| [v0.15.1](https://github.com/go-llmflow/gateway/releases/tag/v0.15.1) | the report charts (pies, bars and series) all drawn by go-plot v0.3.0 with plain-text labels; the gateway's own SVG chart engine is removed |
| [v0.15.2](https://github.com/go-llmflow/gateway/releases/tag/v0.15.2) | **Slurm discovery works with `ha`** (#83), and go-pdfkit v0.20.0: the binary no longer links a widget toolkit (42,110,114 to 39,790,834 bytes, 5.5 % smaller). Open defect 18 is fixed |
| [v0.16.0](https://github.com/go-llmflow/gateway/releases/tag/v0.16.0) | **mutual TLS on the internal replica channel** (`tls_cert_file`, `tls_key_file`, `tls_ca_file`, `tls_server_name` in the `ha` block). **It refused every real PEM file: use v0.16.1** |
| [v0.16.1](https://github.com/go-llmflow/gateway/releases/tag/v0.16.1) | the mutual-TLS files are read as PEM over several lines (file paths or `vault:` references); the key file must not be readable by group or others |
| [v0.17.0](https://github.com/go-llmflow/gateway/releases/tag/v0.17.0) | **hard budgets** (#87): the worst case is reserved before admission, escrow grants in PostgreSQL, settlement in the usage record's transaction. A **PostgreSQL migration** and a behaviour change for small budgets: see [Hard budgets](#hard-budgets). A budget created mid-period could still be exceeded: use v0.17.1 |
| [v0.17.1](https://github.com/go-llmflow/gateway/releases/tag/v0.17.1) | **a budget created mid-period** starts arming and no longer lets earlier work push a period past its limit (#92). A small **PostgreSQL migration**: upgrade every replica before creating budgets. See [Budget created mid-period](#budget-mid-period) |
| [v0.18.0](https://github.com/go-llmflow/gateway/releases/tag/v0.18.0) | **workload profiles** (#90): a profile in days per week, enforced by hard budgets; workday calibration, session window, overflow, delegation. A **PostgreSQL migration**: upgrade every replica before assigning profiles. See [Workload profiles]({{< relref "workload-profiles.md" >}}) |

**Which tag to take.** v0.13.1 was published **after** v0.14.0, and on a commit that
already contains the HA code of v0.14.0 (it descends from it); v0.14.1 is the very same
commit. A tag number going down in time is therefore not a downgrade here. Take
**v0.14.1 or later** (today v0.18.0), not v0.13.1, so that your version number says
what the binary holds. The HA code does nothing without an `ha` block.

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
- **From v0.9 to v0.10.** `store.spool_dir` is mandatory with the postgres store;
  see the next heading.

### Upgrading to v0.10.0 {#upgrading-v0100}

**`store.spool_dir` is mandatory with the postgres store: without it the gateway
refuses to start.** It is the directory
  of the durable accounting spool: every batch of usage records is appended to a
  file there and fsync'd before it is inserted, so that records survive a
  PostgreSQL outage or a crash, within the bounds listed under
  [Limitations](#limitations) (the gateway's README, "Durable accounting"). The
  directory is created with mode 0700 and refused if group- or world-writable; files
  are mode 0600. `store.spool_max_bytes` caps it (256 MiB per process by default).
  Metrics: `llmflow_accounting_spooled`, `llmflow_accounting_replayed_total`; alert
  `accounting_records_lost`. A process that dies leaves its file, and the next
  process on the directory adopts and stores it. Also new: `sso.min_refresh` (at
  least 5 s, as for `jwt`).

```hcl
store {
  kind            = "postgres"
  dsn_file        = "/etc/llmflow/pg.dsn"
  spool_dir       = "/var/lib/llmflow/spool"
  spool_max_bytes = 268435456
}
```

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
| `POST /admin/budgets/increase` | a temporary increase with an expiry; `force` as on PUT (see [Hard budgets](#hard-budgets)) |
| `GET`, `PUT`, `DELETE /admin/users/{user}/profile`; `GET /admin/profiles`, `/admin/profiles/people`, `/admin/profiles/capacity`, `/admin/profiles/calibrate`; `POST /admin/profiles/calibrate` | workload profiles (since v0.18.0): see [Workload profiles]({{< relref "workload-profiles.md" >}}#api) |
| `POST /admin/budgets/release-replica` | release the grants of a dead replica (an org admin: on their organisation's budgets); only a proxy admin also drops its registration; audited |
| `GET /admin/spend?from=&to=&group_by=&format=` | spend report grouped by any of `org`, `lab`, `team`, `project`, `user`, `key`, `model`, `pool`, `tag`, `day`, `profile`; `json`, `csv` or `parsable2`; filtered to what the caller may see; 60 s deadline |
| `GET /admin/export?from=&to=&format=csv\|xlsx\|pdf` | the report as a download, audited; see [Finance reports]({{< relref "finance-reports.md" >}}#exports) |
| `GET /admin/usage.csv?from=&to=` | the v0.1 export |
| `GET /admin/fairshare` | reservations, in-flight, borrowing, usage, level fairshare and rank per pool and organisation |

A request is refused with 429, naming the scope, as soon as one budget it falls
under cannot hold its worst-case cost ([Hard budgets](#hard-budgets), since v0.17.0). `soft_limit` triggers one alert per budget and period, to a
webhook and/or by e-mail through an SMTP relay.

## Configuration (HCL2) {#configuration}

The configuration is written in
[HCL2 native syntax](https://github.com/hashicorp/hcl/blob/main/hclsyntax/spec.md);
the repository's `examples/gateway.hcl` is parsed by its test suite, so it stays
valid. This extract was run through `llmflow-gateway check-config` v0.18.0 (with
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
| `store { ... }` | `kind` (`memory` or `postgres`), `dsn_file`, `spool_dir` (**required with postgres**, see [Upgrading to v0.10.0](#upgrading-v0100)), `spool_max_bytes` (default 256 MiB), `fail_closed` (default true), `queue_size`, `batch_size` |
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

**Why.** vLLM *can* keep responses in the memory of the instance that produced
them: it does so only when started with `VLLM_ENABLE_RESPONSES_API_STORE=1`. With
that on, anyone holding an id could read a response back, continue it or cancel it,
and behind a gateway that state would be shared by every tenant. The gateway forces
`store: false` and refuses the stateful fields **regardless**, so a deployment with
the store enabled stays safe. Verified against a real vLLM (see
[Benchmarking]({{< relref "benchmarking.md" >}}#verified-vllm)): `GET /v1/responses/{id}` through the gateway answers 404.

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
and Anthropic's top-level `"type": "error"`.

**`/v1/messages/count_tokens`** (since v0.12.1) goes to vLLM's own endpoint, which
renders the chat template and counts the prompt's tokens, so the count is **exact**
(`{"input_tokens": N}`). Claude Code calls it, and falls back to a character-based
estimate without it. It is authenticated, allow-listed and rate limited like
`/tokenize`; it is **not charged and not guarded**. `cache_salt` and the engine fields
are stripped, and `chat_template_kwargs` is kept, so the count matches the
`/v1/messages` request that carries the same fields.

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
  pool               = "b300"
  fallbacks          = ["llama-8b"]
  fallback_on        = ["unavailable", "saturated"]  # default ["unavailable"]
  saturation_waiting = 4                             # optional
  # ...
}
```

**`"unavailable"` (the default): an outage.** Before anything has reached the client,
a request moves to the next fallback when no upstream of the group took it, or when
vLLM answered 429 or 503.

**`"saturated"` (since v0.11.0, [issue #58](https://github.com/go-llmflow/gateway/issues/58)).**
Measured on 2026-10-26, a vLLM above its `--max-num-seqs` does not refuse: it queues the
requests and answers 200 (six requests over the limit, six 200s, no 429 or 503). So
the gateway decides saturation itself:

- A pool is **saturated** when it already had `capacity` (C) requests in flight when
  the request arrived. With `saturation_waiting = N`, it is also saturated when the
  group's upstreams report N or more requests waiting in `vllm:num_requests_waiting`
  (read from each upstream's `/metrics`, which vLLM serves without authentication, so
  keep its ports internal; an upstream that cannot be read counts as unknown, never
  as saturated).
- **Only a borrowing request moves.** A request within its organisation's share keeps
  its reservation and queues in its own pool; the README states this as a property,
  tested over 200 random schedules and failing on a planted regression.
- **Only into room**, and the check of the target pool's room and the admission there
  are one atomic step. A request moves **at most once**; with no room anywhere it
  waits in its own pool, so two saturated groups naming each other cannot ping-pong.

In both cases the target must be in the team's model list and within its budgets. The
request acquires in the **target's pool** (so it gets its organisation's priority
there), is rewritten for the target, and is **charged and recorded under the model that
served it**. The response carries `X-Llmflow-Model`, and
`llmflow_fallbacks_total{from,to,reason}` counts the moves, `reason` being
`unavailable` or `saturated`. Any other answer, a 400 for instance, is final.

**Upgrading to v0.11.0.** `llmflow_fallbacks_total` has a new `reason` label (breaking
for a dashboard or an alert that matches its labels exactly). And `fallback_on` now
also governs the outage case: a model set to `["saturated"]` alone no longer falls back
when its upstream is unavailable. The real-vLLM check is on the
[Benchmarking]({{< relref "benchmarking.md" >}}#verified-vllm) page.

### Anthropic as an upstream (opt-in, since v0.12.0) {#anthropic}

Several institutions can share **one Anthropic organisation's API usage** through the
gateway, with its keys, budgets, fair share, audit and guardrails. The only path is an
**Anthropic API key under the Commercial Terms**: pooling Claude Pro or Max
subscriptions is forbidden by the Consumer Terms, and Claude Code's documentation
forbids routing requests through Free, Pro or Max credentials on behalf of users
([legal and compliance](https://code.claude.com/docs/en/legal-and-compliance)). The
gateway therefore never relays a subscription: it drops the OAuth `anthropic-beta`
capability and never forwards a client's `Authorization` or `x-api-key`. This is not
legal advice: check your institution's contracts with your DPO.

This block, from the gateway's `examples/anthropic.hcl` with this page's names, passes
`llmflow-gateway check-config` v0.18.0 (with the key files present, and the
organisations, the `b300` pool and the local `llama-3.3-70b` model of that example
around it):

```hcl
pool "anthropic" { capacity = 32 }   # concurrent requests the institutions share at Anthropic

model "claude-sonnet" {
  provider        = "anthropic"
  upstream_model  = "claude-sonnet-5-5"
  pool            = "anthropic"
  allow_external  = true
  orgs            = ["instA", "instB"]
  anthropic_betas = ["context-management-2025-06-27"]
  cost {
    unit                    = "USD"
    price_date              = "2026-10-26"
    input_per_mtok          = 2
    output_per_mtok         = 10
    cached_per_mtok         = 0.1
    cache_write_5m_per_mtok = 2.5
    cache_write_1h_per_mtok = 4
  }
  upstream "api" { base_url = "https://api.anthropic.com" }
  org_key "instA" { key_file = "/etc/llmflow/anthropic-instA.key" }
  org_key "instB" { key_file = "/etc/llmflow/anthropic-instB.key" }
  fallbacks   = ["llama-3.3-70b"]
  fallback_on = ["unavailable"]
}

guardrail "pii" {
  type = "pii"
  orgs = ["instA", "instB"]
}
guardrail "secrets" {
  type = "hide_secrets"
  orgs = ["instA", "instB"]
}
```

(The model names, prices and `price_date` are the README's example; read the current
price list and put its date in `price_date`.) What `check-config` enforces:

- **`allow_external = true`** is required, because prompts leave the site, with an
  explicit `orgs` or `teams` allow-list (others get 404) and https.
- **`external_requires_guardrails`** (top level, default true): every allowed
  organisation or team needs pre-call `pii` and `hide_secrets` guardrails, masking or
  blocking and fail-closed. They are checked at configuration time and again on every
  request, which answers 503 when they are missing.
- **One API key per organisation** (`org_key`) when more than one may use the model.
  Anthropic's prompt cache is isolated per **workspace** (the README quotes the prompt
  caching page, read on 2026-10-26), which is the timing side channel that `cache_salt`
  closes for vLLM; Anthropic accepts no salt. Two organisations naming the same key
  file are refused, and if two ever resolve to the same workspace
  (`anthropic-workspace-id` on the answers) both are refused with 503. With
  `cache_isolation = "none"`, one key may serve everybody, and the residual risk is a
  shared cache between institutions.
- An external model is **never a fallback target** and never shares a pool with vLLM
  models.

**Routes.** `/v1/messages` only, JSON and streaming. `/v1/chat/completions` and the
other routes answer **400** for an Anthropic model, because Anthropic's
OpenAI-compatibility layer "is not considered a long-term or production-ready
solution". A client may authenticate with
`Authorization: Bearer llmf-...` **or `x-api-key: llmf-...`**, as Anthropic SDKs and
Claude Code do.

**`/v1/messages/count_tokens`** (since v0.12.1) is sent to Anthropic's endpoint with the
organisation's key and the same filtered headers. It is **not charged**: Anthropic says
token counting "is free to use". It is rate limited per key like every route. The
pre-call `pii` and `hide_secrets` guardrails **still mask the prompt**, because a count
also sends it off the site, and the count is then that of the prompt `/v1/messages`
really sends. Anthropic rate-limits counting separately from message creation, so a 429
on a count is passed through **without starting the organisation's messages back-off**,
and a messages back-off does not stop the counts. A shared workspace refuses both, and
an organisation that is not allowed gets **404**.

**What reaches Anthropic.** Headers are rebuilt: `x-api-key` is the organisation's
key; `anthropic-version` is the client's if allow-listed; `anthropic-beta` carries the
allow-listed values only (add the betas your users need, or `"*"`; never an OAuth
one); no priority, request id or trace context is sent. The body goes through the same
rewrite as for vLLM. Back to the client go the provider's `request-id` and
`retry-after`, but not its `anthropic-ratelimit-*` headers, which describe the shared
organisation's usage.

**Back-off.** The pool's `capacity` is the concurrency the institutions share at
Anthropic, apportioned by their shares. A 429 or 529 starts a back-off **for that
organisation**, for `retry-after` seconds (at most 10 minutes; 5 seconds when the
header is absent). During it, requests are not sent: they go to the model's
`fallbacks` (a local vLLM group) when `fallback_on` includes `"unavailable"`, or get
429 with `Retry-After`. Other organisations are not affected.

**Accounting** uses Anthropic's usage fields (input, cache reads, cache writes split by
5-minute and 1-hour lifetime, output), each priced at its own rate from the `cost`
block, in USD, with its price date.

### Hard budgets: a budget is never exceeded (since v0.17.0) {#hard-budgets}

A request starts only when **every budget it falls under can pay its worst case**, as
Slurm's `AccountingStorageEnforce=safe` starts a job only if the job will be able to run to
completion ([resource limits](https://slurm.schedmd.com/resource_limits.html)). Before
v0.17.0 the cost was known only when a request ended, so a burst could overspend by what
was in flight; that limitation (number 2 in the list below) is removed.

**The worst case W** comes from vLLM's own bound: it refuses an input longer than
`max_model_len` and generates at most `min(max_tokens, max_model_len - input)` per choice
(`get_max_tokens`). With `L = max_model_len`, `M` the `max_tokens` sent, `n` choices and
the highest price that can apply (`p_in`: input, cached, cache writes, either tier;
`p_out`: output), per prompt:

```text
W = max((L - M) * p_in + n * M * p_out, L * p_in)
```

Completion prompt lists, embedding inputs and rerank documents count one prompt each. For
an Anthropic model, `W = context_window * p_in + M * p_out`. Costs are integers of
nano-units, so there is no float drift in `spent + held <= limit`.

**`max_tokens`.** Under a budget the gateway always sends one. The client's value is
**never lowered**: when its W does not fit, the request is refused with 429
(`insufficient_quota`) naming the scope, the room left and the request's maximum cost.
When the client sent none, the gateway picks the largest that fits, from the model's
`max_output_tokens` (or the context length) down to `budgets.min_max_tokens`, and says so
in the `X-Llmflow-Max-Tokens` response header. Every answer under a budget carries
`X-Llmflow-Budget-Reset` (RFC 3339): when the budget that refused resets or, on success,
when the tightest one does.

**Refused under a budget (400, with the reason)**, because nothing bounds their cost:
Responses built-in tools; Anthropic server tools, `mcp_servers` and `container`; the
`compact-*` betas, and a `context-1m-*` beta wider than `context_window`; transcriptions
of a priced model (priced per audio duration). A model whose context length is unknown is
refused under a budget with 503, rather than guessed.

**The escrow.** PostgreSQL holds, per budget and period, `spent` and, per replica process,
a **grant**; every transaction keeps `spent + sum(grants) <= limit`. A replica admits
locally while its grant covers what it has in flight, with no database round trip per
request. A grant that runs short grows in one transaction (all of the request's budgets or
none, rows locked in one order), by the larger of what is missing and `grant_share` of the
room left; near the limit grants shrink to one request, and an unused grant goes back after
`grant_idle`. Storing a request's usage record **settles its hold in the same
transaction**; a record replayed from the spool settles nothing twice. A request that costs
more than its W is charged anyway and counted in `llmflow_budget_hold_exceeded_total`
(alert on any). The estimate of an aborted stream is capped at W.

**Crashes.** A dead process's grants are never released on a timer, since its spool may
still hold records: the process that replays the spool releases them once every record is
stored. A replica that never comes back is released with
`POST /admin/budgets/release-replica {"replica": "<name>"}`, audited, by an org admin (that
organisation's budgets) or a proxy admin (see [below](#budget-mid-period)).

**Configuration.** The fragments below pass `llmflow-gateway check-config` v0.18.0 (with the
key files present, and organisations, pools and keys around them):

```hcl
budgets {
  grant_share    = 0.1   # share of the room left that a refill takes (0..1]
  grant_idle     = "5s"  # an unused grant goes back after this (100ms..10m)
  min_max_tokens = 16    # the smallest max_tokens the gateway picks
}

model "llama-3.3-70b" {
  max_model_len     = 32768  # else read from the upstreams' /v1/models
  max_output_tokens = 4096   # caps max_tokens (a larger one: 400)
  # ... upstream_model, pool, key_file and upstream blocks as before
}

model "claude-sonnet" {
  provider       = "anthropic"
  context_window = 200000    # required for a budget to admit the model
  # ... as in "Anthropic as an upstream" above
}
```

Without `max_model_len` the gateway reads it from each upstream's `GET /v1/models` at
health check and sends a held request only to an upstream whose length it has read since
that upstream last became healthy. **Set `max_model_len` on every model a budget can apply
to**, so it is never learned late, for example from a Slurm instance that just joined.
`check-config` bounds every price to 1e6 per million tokens (so the integers cannot
overflow) and refuses a context beta wider than `context_window`: with
`anthropic_betas = ["context-1m-2025-08-07"]` or `["*"]` with `context_window = 200000`,
v0.17.0 answers `Context beta wider than context_window ... set context_window = 1000000
(or more), or do not allow the beta` (both were run). A `compact-*` beta passes
`check-config` and is refused at request time under a budget, as listed above (that part
was not run here).

**Admin API.**
- `PUT /admin/budgets` and `POST /admin/budgets/increase` accept `"force": true`. Lowering
  a budget below what the period committed is refused with **409** without it; with it
  (proxy and organisation admins only, audited) new admissions stop and what was admitted
  finishes. Other replicas see it at their next budget refresh.
- A new budget kind, **`user`**: the person as the reports name them (a key's owner, or a
  JWT's user), across all their keys and tokens. Set by the admins of an organisation or
  team the user belongs to; the user reads their own.
- `GET /admin/budgets` returns `committed` (the cost of the requests settled under the
  budget in the period) and `held` (the grants the replicas hold for requests in flight)
  beside `spent`.
- A temporary increase is lent only to requests that end before it does
  (`limits.max_request_duration`); a budget created mid-period starts arming (see
  below); a request is charged to the period that admitted it.

**Metrics.** `llmflow_budget_hold_exceeded_total` (alert on any),
`llmflow_budget_grant_refills_total`, `llmflow_budget_hold_refused_total` and
`llmflow_budget_grant_return_failures_total`.

### Budget created mid-period (since v0.17.1) {#budget-mid-period}

In **v0.17.0** a budget created over a scope in use could be exceeded: the requests
admitted before it existed held nothing on it, yet their cost landed in it afterwards, and
a replica that had not yet loaded it kept admitting in its scope without a hold. **v0.17.1
(#92) fixes this by creating a new budget *arming*.**

- **Arming.** Every gateway process keeps, for every request it admitted (budget or not),
  its worst case W until its record is stored. When it loads the new budget (at once on the
  replica that created it, by `NOTIFY` on the others under `ha`, at the latest at
  `budget_refresh`) it **reports** the W it still has unsettled in the budget's scope, and
  lowers the report as those requests settle. **No grant is made until every registered
  process has reported.** After that, the reports count against the limit
  (`spent + sum(grants) + sum(reports) <= limit`) until all are back to 0, and the budget is
  then active.
- **The 429.** While a budget is arming, requests in its scope get 429 with a message that
  reads "was just created and is arming ... retry in a moment". An `org` or `team` budget
  created over a scope with heavy traffic may therefore refuse for a moment after its
  creation.
- **A dead process** stays registered until the process that replays its spool releases it,
  or a proxy admin releases its replica (registration included) with
  `POST /admin/budgets/release-replica`: the
  budget stays arming for its scope meanwhile. Never on a timer.
- **Records.** Every stored record now counts in every budget of its scope, held or not;
  that is how the cost of work admitted before a budget existed lands in it. A period's
  starting spend is read with the usage inserts blocked (`LOCK TABLE ... IN SHARE MODE`),
  so no record falls between that read and the later settlements.

**The guarantee, stated honestly.** No admission control can refuse work admitted before the
budget existed, or by a replica in the instant before it loaded the budget. So a period never
ends above **max(limit, the cost of the work admitted before the budget existed, which held
nothing on it)**: what the budget itself admits never takes the period past its limit, or
past the cost of that earlier work when it alone is larger.

**`release-replica`** is audited and has two levels. An **org_admin** releases only that
replica's grants on their own organisation's budgets. Only a **proxy_admin** also drops the
replica's registration, which arming budgets of every organisation wait for.

**Other edges.** A forced lower (`"force": true`) is an explicit admin act: what was already
admitted finishes. A record dropped by a full accounting queue is not billed, as before,
with its hold staying reserved until its replica restarts or is released, which errs on the
safe side. And **with `store.fail_closed = false`, a replica that cannot load the budgets
keeps admitting without them: that voids the guarantee, which needs the default,
fail-closed.**

**Profiles on top of budgets.** Since v0.18.0 a person's workload profile is a set of these
hard budgets, managed by the gateway: see [Workload profiles]({{< relref "workload-profiles.md" >}}).
Rolling windows (`"period": "rolling:5h"`) are a general budget period.

**Upgrading from v0.17.0 to v0.17.1.** Start-up adds the tables `llmflow_budget_holders` and
`llmflow_budget_arming` and the column `arming` on `llmflow_budgets_v2`; existing budgets
stay active. **Upgrade every replica before creating a budget**: a v0.17.0 process does not
register, so its unheld requests would not be reported.

**Upgrading from v0.16 to v0.17.** The migration runs at start-up: it adds the tables
`llmflow_budget_ledger` and `llmflow_budget_grants`, `limit_epoch` on `llmflow_budgets_v2`,
and `cost_nano` and `admitted_at` on `llmflow_usage`; each period's ledger starts from its
stored usage. Back the database up first. Budgets that used to admit while under their limit
now admit only what fits at the worst case: **set `max_tokens` (or `max_output_tokens`) to
keep small budgets usable**. Spool files written by v0.17 carry the holds and are **not
readable by v0.16**: drain the spool before a downgrade.

**What was measured** (see [Benchmarking]({{< relref "benchmarking.md" >}}#verified-hard-budgets)):
a live test against a real vLLM kept usage within W in six cases, and the gateway
repository has a PostgreSQL property test that kills replicas and checks that no period
passes its limit, with a planted control that does. **No performance figure exists yet**
for the extra admission cost: the machine's load was above the measurement gate.

### Cost units (since v0.12.0) {#cost-units}

A cost has a **unit**. vLLM models cost GPU-hours by default; a model can be priced in
a currency, with the date its prices were read (`unit = "USD"`, `price_date`).
**GPU-hours and a currency are never added together**: a budget carries a unit
(`{"kind":"org","name":"instA","unit":"USD","period":"month","max_cost":500}`) and
counts only the requests of models priced in that unit; a spend report is split by
unit, with `cost` and `unit` columns (CSV and parsable2 gain them at the end);
currency costs go to the metric `llmflow_cost_total{team,pool,unit}`, while
`llmflow_gpu_hours_total` stays GPU-hours only. A pool must not mix units
(`check-config` refuses it).

**Upgrading to v0.12.0.** PostgreSQL gains a `unit` column on `llmflow_usage` and on
`llmflow_budgets_v2`, whose primary key becomes (kind, name, model, unit). Existing
rows are GPU-hours. **The migration runs at start-up, once**: back the database up
first, as for any schema change.

### Project attribution, labs and users (since v0.13.0) {#attribution}

Coding clients do not say which project a request belongs to. Claude Code sends no
project header of its own, but it sends whatever `ANTHROPIC_CUSTOM_HEADERS` holds;
Continue sends the headers of `requestOptions.headers`. A request can therefore name
its project in a header (`X-Llmflow-Project` by default), and a team can carry an
optional `lab`:

```hcl
attribution {
  project_header  = "X-Llmflow-Project"  # the default
  default_project = "general"            # optional
}

org "etabA" {
  share = 0.5
  team "physics-1" { lab = "physics" }   # optional lab: org, lab, team, project
}
```

This block passes `llmflow-gateway check-config` v0.18.0 (with the key files present,
and a pool and a model around it).

- **What the header may name.** A project **name**, looked up in the key's own team
  only: another team's or organisation's project cannot even be named. The project
  must exist and be enabled; a key pinned to a project accepts only that one. For a
  key with an owner, the owner must still hold a stored `member`, `team_admin`,
  `org_admin` or `proxy_admin` role on the team; for a JWT the team comes from its
  verified `map`. **Every refusal is the same generic 403** ("project not allowed for
  this key"), counted in `llmflow_denied_total{reason="project"}`.
- **Without the header**, the key's project applies, then `default_project` if the
  team has a project of that name, else none.
- **The effective project** is charged, budgeted, written to the request log and
  recorded. The lookups read state loaded with the budgets, so the request path does
  not read the store.
- **SSO roles do not count.** A role that comes only from SSO ID-token claims is not
  stored, so it does not satisfy the check: grant the role in the gateway, or through
  SCIM.
- **Two reporting dimensions.** `lab` is an accounting level only (RBAC does not use
  it), stamped on each record at request time, so moving a team to another lab does
  not rewrite history. `user` is the key's owner, or a JWT's username when it is a
  valid gateway user name, otherwise the JWT's stable synthetic key id; never a token.
  A lab is an attribute rather than a nesting so that every existing configuration
  stays valid.

How a coding agent sets the header: [Coding agents]({{< relref "agents.md" >}}#claude-code-project).

### Traces {#traces}

A `tracing` block exports OpenTelemetry traces over OTLP/HTTP or gRPC. The W3C
`traceparent` of the upstream span is sent to vLLM, so vLLM's own spans join the
trace. A client's trace context is **restarted** unless `trust_client_context =
true`. Spans carry metadata only (organisation, team, key id, model, pool, priority,
status, token counts, guardrail verdicts), **never a prompt or an answer**.

This block, with the fallbacks and audio settings above, passes
`llmflow-gateway check-config` v0.18.0 (with the key files present, and
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

## High availability: several replicas (since v0.14.0) {#ha}

Without an `ha` block nothing changes: one replica, the fair-share state in its memory.
With one, several replicas share the pools and **PostgreSQL stays the only shared
component** (no etcd, no Redis). Design discussion:
[issue #74](https://github.com/go-llmflow/gateway/issues/74).

```hcl
store {
  kind      = "postgres"          # required by ha
  dsn_file  = "/run/secrets/dsn"
  spool_dir = "/var/spool/llmflow"
}
ha {
  replica_id           = "gw-1"             # optional, default the host name
  listen               = "10.0.0.5:9444"    # internal port, replicas only
  advertise            = "10.0.0.5:9444"    # default: listen (an IP literal)
  peer_networks        = ["10.0.0.0/24"]    # owners are dialled only there
  secret_file          = "/run/secrets/ha"  # shared, at least 32 bytes
  lease_ttl            = "6s"               # renewed every third
  heartbeat            = "1s"               # grant reports to owners
  forward_timeout      = "250ms"
  usage_persist        = "10s"
  on_owner_unreachable = "lowest_priority"  # or "reject" (503)
}
```

This block passes `llmflow-gateway check-config` v0.18.0 (with the key, DSN and secret
files present, and organisations, a pool and a model around it). `check-config` also
checks the block: the postgres store, an advertised IP literal inside `peer_networks`,
`heartbeat` at most a third of `lease_ttl`, `forward_timeout` at most a heartbeat, and a
secret of at least 32 bytes.

The optional **mutual-TLS** lines go in the same `ha` block (since v0.16.0; **use
v0.16.1 or later**, see below):

```hcl
ha {
  # ... as above, then:
  tls_cert_file   = "/etc/llmflow/ha-cert.pem"  # this replica's certificate (PEM)
  tls_key_file    = "/etc/llmflow/ha-key.pem"   # its key: one PEM block, mode 0600
  tls_ca_file     = "/etc/llmflow/ha-ca.pem"    # signs every replica
  tls_server_name = "llmflow-ha"                # the default; in the certificate's SAN
}
```

This form is **tested live**: bench v0.6.0's
[`TestHAMutualTLSLive`](https://github.com/go-llmflow/bench/blob/v0.6.0/e2e/ha_mtls_test.go)
(see [Benchmarking]({{< relref "benchmarking.md" >}}#verified-ha-mtls)) ran on 2026-10-27
against PostgreSQL 18.6 with two gateway v0.16.1 processes configured from PEM files
(ECDSA and Ed25519 leaves, SAN `llmflow-ha`, key files at mode 0600, which the loader
requires): 8 of 8 admissions sent to the non-owner were forwarded to the owner over mTLS.
`check-config` v0.18.0 also accepts such files. The
repository's README also shows the certificate and key as `vault:` references, such as
`vault:secret/llmflow/ha#cert`; the loader accepts both a file path and a `vault:`
reference, and that form needs a `vault` block which is not shown here and was not run.
The certificate file may carry intermediates after the leaf, the CA file may bundle
several CAs, and `check-config` refuses a certificate outside its validity period.

**Use v0.16.1 or later for mutual TLS.** v0.16.0 read these three values with the
one-line secret reader and **rejected every multi-line PEM file**: `check-config` v0.16.0
answered `ha tls_cert_file: secret ... holds more than one line` for a real certificate,
and the live test fails against a v0.16.0 binary for that reason. Fixed in v0.16.1 (the
PEM-aware loader). The live test's **control**: a replica with a certificate from another
CA gets no answer, and its 3 requests are admitted without an owner, at a borrowing
priority. In the positive run `llmflow_ha_unowned_total` stayed at 0 and the owner's
`llmflow_inflight` reached 3. An in-process equivalent in the gateway repository,
`TestHAMutualTLSProcesses`, runs when `LLMFLOW_TEST_PG_DSN` is set. This is one pair of
replicas on one machine; it does not measure the latency of the channel.

- **One owner per pool.** Each pool's fair share (in-flight counts, reservations, decayed
  usage) is decided by one replica at a time: the holder of the pool's **lease row in
  PostgreSQL**. The lease's `epoch` is a **fencing token** that grows at every change
  of owner. Expiry is computed with PostgreSQL's clock, never a replica's, and a takeover
  is one conditional statement, so of several replicas racing on an expired lease exactly
  one wins.
- **No split brain.** The owner decides only until the monotonic time at which its last
  successful renewal *started*, plus 0.9 x `lease_ttl`; PostgreSQL cannot hand the lease
  to anyone before that renewal expires. An owner cut from PostgreSQL therefore stops
  deciding before another replica can start. Its usage writes are fenced at the storage,
  and a replica that has seen a newer epoch refuses an older owner's answer.
- **Forwarding the decision, not the request.** The other replicas send *acquire* and
  *release* to the owner over the internal port, and still send the request to vLLM
  themselves: one small round trip per admission (the release is asynchronous), and the
  streamed answer does not cross a second gateway. Every message is signed with
  HMAC-SHA256 (key derived from `secret_file`), bound to the sender's instance, a sequence
  number and a timestamp (5 minutes of clock difference accepted); replays are refused with
  a sliding window per instance (RFC 4303, 3.4.3). The owner's address is read from the
  database and dialled **only inside `peer_networks`**, without redirects, with
  `forward_timeout`.
- **Mutual TLS on the internal port (since v0.16.0, working files since v0.16.1),
  optional.** With `tls_cert_file`, `tls_key_file` and `tls_ca_file` (file paths or
  `vault:` references, **all three or none**: `check-config` refuses a partial set) the channel runs over mutual TLS 1.3. Each
  replica presents its certificate, accepts only peers the CA signed, and checks that the
  server's certificate carries `tls_server_name` (default `llmflow-ha`; replicas are
  dialled by IP). The HMAC signatures stay: they bind each answer to its request and
  refuse replays. Tested in the repository: a peer without a certificate, with one from
  another CA, speaking plain HTTP, or expecting another server name gets nothing, even
  though it holds the HMAC secret. The cost (a handshake per kept-alive connection, then
  AES-GCM on bodies of a few hundred bytes) is **not measured yet**.
- **Failover.** Every replica reports its live grants to the owner each `heartbeat`. A new
  owner (epoch + 1) restores the decayed usage the previous one persisted every
  `usage_persist`, adopts the grants the surviving replicas report, and counts the
  previous owner's last persisted in-flight requests as **phantoms**: a safety margin that
  expires after `limits.max_request_duration` and shrinks as reports account for them.
- **Without a reachable owner** (the seconds of a failover) a request is admitted at the
  **lowest borrowing priority**, which cannot take anyone's reservation, or refused with
  503 under `on_owner_unreachable = "reject"`. Such a request cannot move to a fallback on
  saturation, because the pool's room is unknown.
- **Also shared through PostgreSQL.** SSO logins and sessions (the row key is the SHA-256
  of the cookie, the content sealed with AES-256-GCM under a key derived from
  `secret_file`); soft-budget alerts, through an outbox table, delivered at least once
  with the same `Idempotency-Key`; key revocations, broadcast with `NOTIFY` and `LISTEN`
  so every replica drops its cached identities at once.
- **Metrics.** `llmflow_ha_owner`, `llmflow_ha_epoch`, `llmflow_ha_local_grants` (per
  pool), `llmflow_ha_unowned_total`, `llmflow_inflight_phantoms` (per pool and
  organisation) and `llmflow_ha_revocations_total`. The fair-share gauges are published by
  the owner only.

**Chaos tests** (real PostgreSQL 18, 2026-10-26, from the repository's README):

| Fault | Result |
| --- | --- |
| Owner killed with SIGKILL mid-traffic (`lease_ttl` 1.5 s) | survivor owner after **1.3 s**, epoch 1 to 2; 207 requests served meanwhile, **0 failed** (203 of them at the lowest priority before the takeover) |
| Owner partitioned from PostgreSQL (`lease_ttl` 900 ms) | the old owner stops deciding 804 ms after the cut, the other replica owns from 904 ms; **0 of 805 samples with two owners**; it does not take the lease back after the heal |
| Two replicas racing for a lease (8 claimants x 20 rounds) | exactly one owner each time |
| Clock skew of +/- 2 minutes on one replica | ownership does not move, forwarding works |
| A key revoked on one replica while cached on the other (`key_cache_ttl` 1 h) | refused on the other replica after **3 ms** |
| SSO login started on replica A, callback on B, B killed, session used on C | the session is served by C; the state cannot be replayed |
| The same soft-budget alert queued by two replicas | one delivery |
| Random partitions, kills and restarts of 4 replicas under traffic, 8 s | never two owners at one epoch, over 14,000 to 27,000 decisions per run |

**Not measured, and open.** The **added latency** of a forwarded admission is not measured
yet (the machine's load was above the gate's limit); a Go benchmark compares a single
replica, the owner and a forwarded admission. **Per-key rate limits** (`rpm`, `tpm`,
`max_inflight`) are still counted **per replica**: N replicas allow up to N times the
limit. The fencing token is checked by PostgreSQL and by the replicas, not by vLLM, so a
decision an owner made just before a long pause can still reach vLLM. Without the `tls_*`
files the internal channel is **authenticated but not encrypted**: keep it on a private
network, or set them (see above). Since v0.15.2 an `ha` block **can** be combined with
[Slurm discovery](#slurm-discovery).

## Slurm discovery: vLLM instances as Slurm jobs (since v0.15.0) {#slurm-discovery}

A model can take its upstreams from **Slurm** instead of, or on top of, static
`upstream` blocks. This is the `slurm` block **inside a `model` block**; the top-level
`slurm` block of [Finance reports]({{< relref "finance-reports.md" >}}#slurm) is a
different thing (the input of the reconciliation). The vLLM instances run as long jobs of
a service account, for example account `svc_llm` with a QOS `service`, and can be stopped
at night so that the GPUs can be lent to a borrowing QOS. The gateway **follows** the
running jobs; it never starts or stops one. Slurm (`scrontab`) or the operator does that.

```hcl
pool "gpu" { capacity = 0 }   # 0: C is the sum of the instances' capacities

model "llama" {
  upstream_model = "meta/llama-3.3-70b"
  pool           = "gpu"
  key_file       = "/etc/llmflow/vllm.key"     # the instances' API key
  slurm {
    url         = "https://slurmrestd.cluster:6820"   # from here only
    api_version = "v0.0.44"                      # slurmrestd -d list
    token_file  = "/etc/llmflow/slurm.jwt"       # any secret reference
    user_name   = "svc_llm"                      # X-SLURM-USER-NAME (optional)
    accounts    = ["svc_llm"]                    # required
    partitions  = ["gpu"]                        # required
    users       = ["svc_llm"]                    # optional allow-list
    interval    = "15s"
    stale_after = "5m"
    instance_capacity     = 8                    # when the job does not say
    max_instance_capacity = 64
  }
}
```

This block passes `llmflow-gateway check-config` v0.18.0 (with the key files present,
and organisations around it).

- **Convention.** A job serves model `M` on port `P` when its name (or else its comment)
  is `llmflow:M:P` or `llmflow:M:P:C`. `C` is the instance's contribution to the pool's
  capacity; without it, `instance_capacity` applies.
- **Discovery.** Every `interval` the gateway reads `GET <url>/slurm/<api_version>/jobs/`
  with `X-SLURM-USER-TOKEN`. It keeps the jobs that are `RUNNING` and carry none of the
  `COMPLETING`, `CONFIGURING`, `STOPPED`, `SIGNALING`, `RESIZING` or `REQUEUED` flags, in
  an allowed account and partition (and user, when `users` is set). An instance is added
  only after `GET /health` answers 200 and `GET /v1/models` lists `upstream_model`.
- **Removal.** When the job leaves the list, or fails health 3 polls in a row, the
  instance is **drained**: no new request, not even as a last resort. It is removed once
  its requests in flight have finished, or after `limits.max_request_duration`.
- **Capacity.** The pool's C is its configured `capacity` plus the capacities of its
  non-draining instances. Every membership change re-apportions the organisations'
  integer reservations with the same Hamilton method, and is **audited**
  (`slurm.member_added`, `slurm.member_drained`, `slurm.member_removed`,
  `pool.capacity`). Requests in flight keep their status.
- **Safety.** Only the configured accounts, partitions and users are trusted: without
  that, anyone could submit a job named `llmflow:llama:8000` and capture prompts. An
  untrusted job claiming a model is logged once and counted in
  `llmflow_slurm_rejected_jobs_total{reason}`. The URL comes from the configuration only;
  `batch_host` must be a plain host name; the JWT is a secret reference and is never sent
  across a redirect (redirects are refused); the answer is bounded to 32 MiB and 100,000
  jobs.
- **Fail-closed.** While slurmrestd cannot be read, the last known members are kept and
  still health-checked. Past `stale_after` the gateway logs an error and **admits no
  instance** until a poll succeeds. Healthy members are never dropped because of an outage.
- **Metrics.** `llmflow_pool_capacity`, `llmflow_slurm_polls_total{outcome}`,
  `llmflow_slurm_last_success_timestamp_seconds`, `llmflow_slurm_rejected_jobs_total`.
- **With an `ha` block (since v0.15.2).** Every replica runs its **own** discovery and
  health checks, for its own routing: publishing the healthy set through PostgreSQL was
  not chosen, because it would tie every replica's routing to the owner's liveness and to
  PostgreSQL, and because what is healthy depends on a replica's network path to the
  instance. **Only the pool's owner's view sets C.** It decides with it and publishes C
  and the integer reservations with the lease's epoch (`llmflow_pool_capacity` shows the
  decided C on every replica); a replica that sees fewer instances still routes only to
  those it sees. A replica takes a pool only after its discovery has read the members once;
  until then requests are admitted at the lowest priority (or refused under
  `on_owner_unreachable = "reject"`). An owner whose slurmrestd stays unreadable past
  `stale_after` **yields**: a replica that still reads the jobs takes the lease (epoch + 1)
  and stays quiet for one `lease_ttl`, until the previous owner's validity has surely
  ended. If no replica reads slurmrestd, the owner keeps the pool with its last known
  members. Chaos tests with a real PostgreSQL: the owner killed while a job joins (the
  survivor owns with the new C at epoch + 1); slurmrestd lost by the owner only (handover
  after **1.2 s**, never two owners in about 950 samples, and the test fails without the
  quiet period); two replicas that disagree (each routes to what it sees, the fair share
  is the owner's). The sum of the reservations never exceeded C in any sample.
- **The token is read at start only.** Issue it with `scontrol token username=svc_llm
  lifespan=...` and pick a lifespan longer than your restart cycle: the default is 1800 s.
  Once it has expired, polls fail and the fail-closed rule applies. slurmrestd needs
  `AuthAltTypes=auth/jwt`.

**An instance as a job**, from the README's template (the key is read into
`VLLM_API_KEY`, never passed with `--api-key`, because the node's process list shows every
command line to every user):

```bash
#!/bin/bash
#SBATCH --job-name=llmflow:llama:8000:8
#SBATCH --account=svc_llm --qos=service --partition=gpu
#SBATCH --nodes=1 --gres=gpu:4 --exclusive --time=7-00:00:00
#SBATCH --signal=B:TERM@120
VLLM_API_KEY=$(cat /etc/llmflow/vllm.key); export VLLM_API_KEY
exec vllm serve meta/llama-3.3-70b --served-model-name meta/llama-3.3-70b \
  --host 0.0.0.0 --port 8000 --tensor-parallel-size 4
```

The node's vLLM port must be reachable by the gateway only (firewall): `/metrics` is
unauthenticated and the key protects `/v1` only.

**Day and night with `scrontab`** (needs `ScronParameters=enable`):

```text
#SCRON --account=svc_llm --qos=service --partition=service -t 0:05:00
0 7 * * 1-5 sbatch /opt/llmflow/vllm-llama.sbatch
#SCRON --account=svc_llm --qos=service --partition=service -t 0:05:00
0 20 * * 1-5 scancel --account=svc_llm --name=llmflow:llama:8000:8
```

At 20:00 the jobs end, the gateway drains them, its pool shrinks and the reservations
follow. At 07:00 the new jobs are admitted as soon as vLLM answers.

**Tested against a real Slurm** (bench `TestSlurmDiscoveryE2E`, 2026-10-26, Slurm
24.11.5, slurmrestd data_parser v0.0.42, arm64, a vLLM CPU job serving Qwen2.5-0.5B):
admitted 18 s after `sbatch`, a chat request answered 200, removed 2 s after `scancel`;
a job of another account claiming the model was rejected and never contacted; a vLLM job
whose engine failed to start stayed `RUNNING` and was never admitted. See
[Benchmarking]({{< relref "benchmarking.md" >}}#verified-slurm). That is one small
instance on one node, not a production cluster.

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
| Profiles (since v0.18.0) | each person's workload profile, the capacity check and the workday calibration: see [Workload profiles]({{< relref "workload-profiles.md" >}}#ui) |
| Reports (since v0.13.0) | the finance view of the spend report: period, drill-down, previous period, charts, exports, Slurm reconciliation. See [Finance reports]({{< relref "finance-reports.md" >}}) |
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
with its virtual key. This block passes `llmflow-gateway check-config` v0.18.0
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
below are released in **v0.9.0**; each has a test that fails without the fix. The
full account (method, findings A1 to A9, chaos results, residual risks) is on the
[Security]({{< relref "security.md" >}}) page.

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
  accounting outage was an open defect (limitation 6), fixed in v0.10.0 by the
  durable accounting spool.

### Aborted requests {#aborted}

vLLM's usage chunk comes last. When the client leaves before it, the gateway
stops reading (vLLM is not kept generating for nobody) and charges an
**estimate**: prompt bytes / 4 plus the content deltas already forwarded. The
record is flagged, in an `estimated_requests` column of the spend reports and in
PostgreSQL. The estimate undercounts tool-call and reasoning tokens.

## Limitations {#limitations}

Open defects of v0.18.0, as the repository lists them:

1. **Rate limits and the key cache are per process.** Since v0.14.0 the fair-share
   state is shared between replicas **with an `ha` block** (see
   [High availability](#ha)); without one, run **one gateway replica per pool**, since
   several would each grant the full reservation. Per-key `rpm`, `tpm` and
   `max_inflight` limits are still counted per replica (N replicas allow up to N times
   the limit). Without an `ha` block, a revocation reaches another replica's key cache
   after `key_cache_ttl` (with one, at once by `NOTIFY`). During a failover (up to
   `lease_ttl`) requests are admitted at the lowest priority without an owner's
   decision; a new owner's safety margin is the in-flight count persisted up to
   `usage_persist` earlier; the fencing token is checked by PostgreSQL and by the
   replicas, not by vLLM. Without the `tls_*` files (mutual TLS, usable from v0.16.1) the
   internal channel is authenticated, not encrypted.
2. *(Removed in v0.17.0: budgets no longer lag; the mid-period case was closed in v0.17.1.
   See [Hard budgets](#hard-budgets) and [Budget created mid-period](#budget-mid-period).)*
   What remains: **with `store.fail_closed = false` the guarantee is void**, because a
   replica that cannot load the budgets keeps admitting without them; and the guarantee for
   a new budget is max(limit, the cost of the work admitted before it existed).
3. **Priority is fixed at admission.** A borrowing request already queued in vLLM
   keeps its priority when its organisation drops back under its reservation, and
   teams inside an organisation share its reservation without sub-shares of
   their own. vLLM has no API to re-prioritise a queued request.
4. **Aborted requests are charged with an estimate** (see above).
5. **Prefix-cache isolation is per scope, not absolute** (see above).
6. **Accounting drops records only when its bounds are reached.** A record is
   dropped when the in-memory queue (`queue_size`) is full, and lost when the
   durable spool (`spool_max_bytes`) is full or cannot be written and the store
   refuses it too (counted in `llmflow_accounting_failed_total`, alerted as
   `accounting_records_lost`). A record still in the queue when the process dies (at
   most about 1 s of traffic) is lost. Everything that reached the spool survives a
   store outage and a crash. (The README no longer says that budgets count the cost
   only until the next refresh: since v0.17.0 holds and settlement are in PostgreSQL.)
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
    already marked as alerted. (With an `ha` block alerts go through an outbox
    table in PostgreSQL and survive a restart.)
11. **Without an `ha` block, SSO sessions live in one replica's memory**: behind
    several replicas, use sticky sessions. With one, they are in PostgreSQL.
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
18. ~~**The PDF export links more than it uses.**~~ **Fixed in v0.15.2.** go-pdfkit
    v0.20.0 moved its widget bridge to the subpackage `pdfkit/widget`, so the gateway no
    longer links go-widgets/toolkit and the code behind it. The binary went from
    42,110,114 to 39,790,834 bytes, 5.5 % smaller (darwin/arm64,
    `CGO_ENABLED=0 -trimpath`, the same tree but for the bump; the README's
    measurement on that commit). The library is documented at
    [go-pdfkit](https://go-pdfkit.github.io/docs/).

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
