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

**v0.1.0 is released** ([release notes](https://github.com/go-llmflow/gateway/releases/tag/v0.1.0)).
It ships:

- virtual keys (stored hashed), teams, and budgets in GPU-hours, in PostgreSQL or
  in memory;
- per-key RPM, TPM and in-flight limits;
- per-request accounting in PostgreSQL and a CSV export to reconcile with Slurm
  `sreport`;
- fair-share with reservation and borrowing;
- Prometheus metrics and streaming pass-through without buffering.

```sh
go install github.com/go-llmflow/gateway/cmd/llmflow-gateway@latest
llmflow-gateway check-config -c examples/gateway.hcl   # validates the file and every secret file it names
llmflow-gateway serve -c examples/gateway.hcl
llmflow-gateway export -c examples/gateway.hcl --from 2026-10-01 --to 2026-11-01 > october.csv
```

The binary `llmflow-gateway` has four subcommands: `serve`, `export`,
`check-config` and `version`.

## Architecture {#architecture}

```text
clients (OpenAI SDKs, Authorization: Bearer llmf-...)
   |
   v
llmflow-gateway
   1 auth: virtual key -> team (SHA-256, constant-time, cached)
   2 body <= max_body_bytes, JSON re-encoded, client priority fields stripped
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

### Routes {#routes}

Client routes: `POST /v1/chat/completions`, `POST /v1/completions`,
`POST /v1/embeddings`, `GET /v1/models`. Also `GET /healthz`, which returns 503
when the budget state is stale and fail-closed is on, and `GET /metrics`
(Prometheus text format, optional bearer token). `/v1/responses` is not proxied.

### Admin API {#admin-api}

All admin routes require `Authorization: Bearer <master key>`.

| Route | Purpose |
| --- | --- |
| `GET`, `POST /admin/teams`; `DELETE /admin/teams/{name}` | list, create, disable (revokes all the team's keys) |
| `GET /admin/keys?team=`, `POST /admin/keys`, `DELETE /admin/keys/{id}` | list (no secrets), create (the `llmf-...` secret is returned once), revoke |
| `GET /admin/budgets`, `PUT`/`DELETE /admin/budgets/{team}` | period `day`, `week` or `month` (aligned on UTC) and a maximum cost in GPU-hours |
| `GET /admin/usage.csv?from=&to=` | per team and pool: requests, tokens, GPU-hours (same as `llmflow-gateway export`) |
| `GET /admin/fairshare` | reservations, in-flight, borrowing, usage, level fairshare and rank per pool and team |

## Configuration (HCL2) {#configuration}

The configuration is written in
[HCL2 native syntax](https://github.com/hashicorp/hcl/blob/main/hclsyntax/spec.md);
the repository's `examples/gateway.hcl` is parsed by its test suite, so it stays
valid. An extract:

```hcl
master_key_file = "/etc/llmflow/master.key"

team "etabA" { share = 0.8125 }
team "etabB" { share = 0.1875 }

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
| `team "<name>" { share }` | fraction of every pool reserved for the team (sum at most 1) |
| `pool "<name>" { capacity }` | `C`, the sustainable concurrency of the pool (1 to 1048576) |
| `model "<name>" { ... }` | client model name; `upstream_model`, `pool`, `class` (`interactive` or `batch`), `key_file`, `cost { input_per_mtok, output_per_mtok, cached_per_mtok }` in GPU-hours per million tokens, and `upstream "<name>" { base_url, key_file }` blocks |
| `store { ... }` | `kind` (`memory` or `postgres`), `dsn_file`, `fail_closed` (default true), `queue_size`, `batch_size` |
| `fairshare { half_life }` | usage decay half-life (default `"168h"`) |
| `limits { ... }`, `health { ... }` | body and header sizes, timeouts, maximum request duration and response size; health-check interval, timeout, cooldown and path |
| `auth_failures_per_minute` | failed authentications allowed per client network before a 429 (default 60, 0 disables) |

Design decisions of the format:

- **Static evaluation.** The file is decoded with no evaluation context: no
  variables, no functions. `env.HOME`, `file("/etc/shadow")` or `"${x}"` are
  errors, so a configuration cannot read the environment or other files.
- **Quoted strings are templates.** A literal `${` must be written `$${`, and a
  literal `%{` must be written `%%{`. This applies to every value, URLs, paths
  and model names included.
- **Labelled blocks** for named things: `team`, `pool`, `model`, `upstream`.
  Duplicate labels are errors.
- **Every diagnostic carries `file:line:column`**, from the parser, the decoder
  and the semantic checks; `check-config` prints them one per line.
- **Secrets only by file** (`master_key_file`, `metrics_token_file`,
  `store.dsn_file`, `key_file`). An upstream URL with credentials is rejected.
- **Durations are strings** in Go syntax (`"30s"`, `"168h"`). Absent and zero
  differ: `retries = 0` disables retries, leaving it out takes the default.
- The parser is fuzzed (`FuzzParse`): the configuration is treated as untrusted
  input.

Cost of a request, in GPU-hours:
`((prompt - cached) x input + cached x cached_price + completion x output) / 10^6`,
from the `usage` vLLM reports.

## How fair-share with borrowing works {#fair-share-with-borrowing}

A pool has a capacity `C`: the concurrency it sustains without preemption,
measured externally. Each team owns an integer reservation, obtained by
apportioning `C` over the shares with the largest-remainder (Hamilton) method, so
reservations never sum to more than `C` (C = 256 gives A 208 and B 48).

For every request:

| Situation | Priority sent to vLLM |
| --- | --- |
| the team has fewer within-share requests in flight than its reservation | `0` interactive, `1` batch |
| otherwise (borrowing) | `10 + 10 x rank + class`, rank at most 9 |

Among borrowers, `rank` is the dense rank of the team's **level fairshare**
`LF = S / U` (Slurm Fair Tree): the most under-served team has rank 0, so the
most over-served borrower gets the largest number, is admitted last and is
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

`gateway/bench_test.go` interleaves, on the same client and keep-alive
connections, one request straight to an upstream that answers instantly and one
through the gateway (memory store; auth, budgets, limits, fair share, accounting
and metrics all on the path).

v0.1.0, 2026-10-23, Apple M4 Max, one core, 5 runs of 20,000 interleaved pairs,
1-minute load 7.8 to 8.1 (min to max over the runs):

| | p50 | p99 |
| --- | --- | --- |
| direct to the fake upstream | 37.1-37.8 us | 102.2-105.8 us |
| through the gateway | 92.0-93.1 us | 233.5-239.9 us |
| **added by the gateway** | **54.9-55.3 us** | **131.2-134.2 us** |

LiteLLM's own overhead header read 3.2 to 4.9 ms per request on the same kind of
upstream, about 60 to 90 times the gateway's added p50. **The two numbers are not
measured the same way**: LiteLLM's is self-reported per request, the gateway's is
the end-to-end difference against a direct call. Both exclude the upstream's own
time.

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
- **Keys.** Virtual keys are 256-bit random secrets, stored as SHA-256,
  compared in constant time and shown once. Secrets come from files only and
  never appear in logs, errors or listings.
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

### Aborted requests {#aborted}

vLLM's usage chunk comes last. When the client leaves before it, the gateway
stops reading (vLLM is not kept generating for nobody) and charges an
**estimate**: prompt bytes / 4 plus the content deltas already forwarded. The
record is flagged, in an `estimated_requests` column of the CSV export and in
PostgreSQL. The estimate undercounts tool-call and reasoning tokens.

## Limitations {#limitations}

Open defects of v0.1.0, as the repository lists them:

1. **State is per process.** In-flight counts, rate limits and the key cache live
   in memory: run **one gateway replica per pool**. Several replicas would each
   grant the full reservation.
2. **Budgets are checked before the cost is known.** A burst of long requests can
   overspend by what is in flight, and records still queued at a budget refresh
   are uncounted until the next one.
3. **Priority is fixed at admission.** A borrowing request already queued in vLLM
   keeps its priority when its team drops back under its reservation; vLLM has no
   API to re-prioritise a queued request.
4. **No TLS** in the gateway: put it behind a TLS proxy. The **admin API shares
   the client listener**. `/v1/responses` is not proxied.
5. **`C` is static configuration**; the gateway does not measure or adapt it.
6. Institutions share vLLM's prefix cache, a timing side channel between them;
   `cache_salt` is not set yet.
7. Accounting can drop records when its queue is full or a batch write fails twice
   (both are counted in metrics).

## Roadmap {#roadmap}

[Issue #7](https://github.com/go-llmflow/gateway/issues/7) tracks shipping every
LiteLLM Enterprise feature in the open-source gateway, in four milestones:
**M1** (v0.2.0) governance and multi-tenancy, **M2** (v0.3.0) identity and
access, **M3** (v0.4.0) audit and logging, **M4** (v0.5.0) guardrails. None of it
is in v0.1.0.

## License {#license}

BSD-3-Clause.
