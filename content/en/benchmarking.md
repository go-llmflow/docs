---
title: "Benchmarking"
weight: 30
---

# Benchmarking {#benchmarking}

The [`bench`](https://github.com/go-llmflow/bench) repository holds two small
command-line tools, in pure Go, for measuring LLM gateways such as the
[gateway]({{< relref "gateway.md" >}}): `gwbench` measures the latency a gateway
**adds**, and `fakevllm` is the upstream it is measured against.
[Repository →](https://github.com/go-llmflow/bench)

This page is written from the repository's README at **v0.5.0** (and v0.2.0 for the
parts that did not change). v0.5.0 adds a fake slurmrestd and the Slurm end-to-end test
(see [Slurm discovery](#verified-slurm)). v0.4.0 teaches `fakevllm` to mimic Anthropic's API and adds
the `e2e/anthropic` suite (see [fakevllm](#fakevllm) and
[Anthropic upstream](#verified-anthropic)); v0.3.0 added the
`fakeupstream` package (the fake vLLM as a Go package, for tests that embed it with
`httptest`) and the `e2e/oidc` suite, described on the
[Identity]({{< relref "identity.md" >}}#verified) page. No command or flag of
`gwbench` or `fakevllm` changed. Since v0.1.0,
`gwbench` has a `--direct-model` flag (the model name the upstream serves, sent on
the direct path), `fakevllm` is a cobra command line, and an `e2e/` suite checks
the gateway against a real vLLM (see
[Verified against a real vLLM](#verified-vllm)).

```sh
go install github.com/go-llmflow/bench/cmd/gwbench@latest
go install github.com/go-llmflow/bench/cmd/fakevllm@latest
```

## What is measured {#what}

The question is "how much latency does a gateway add?", not "how fast is the
model". The upstream must therefore answer as fast as possible: that is
`fakevllm`, which replies instantly. The number reported is a **difference**
between two paths to the same upstream, not a latency.

`direct` calls the upstream itself. Every other target (`--litellm`, `--llmflow`,
or `--target NAME|URL|KEYFILE`, repeatable) is a gateway forwarding to that same
upstream.

## The method {#method}

### Interleaving {#interleaving}

Each round sends one request to every target, one at a time, in a fresh random
order (seeded with `--seed`). Drift of the machine (thermal state, a background
job, the cache) then spreads evenly over all targets instead of penalising
whichever runs last. The first `--warmup` rounds are discarded.

### Pairing {#pairing}

A gateway's overhead in round *r* is its latency minus the direct latency **of
the same round**. Percentiles are taken over these paired differences, which
cancels what both paths share: the client, the loopback and the upstream.

### Load gating {#load}

Ratios measured on a busy machine are meaningless. A run starts only when the
1-minute load average is at most `--max-load` (default 4). With `--wait`, the tool
waits for the load to fall; without it, it refuses at once. A run whose load
**ends** above the limit is discarded, not averaged. `--runs` kept runs are
collected, and their rounds stay paired when they are merged.

The load is read from `/proc/loadavg` on Linux and from `sysctl vm.loadavg` on
macOS. Elsewhere the tool cannot read it and refuses to measure.

### The control {#control}

`TestPlantedOverheadIsMeasured` puts a proxy that sleeps a known 3 ms and a
transparent proxy in front of a test upstream. The instrument must report about
3 ms for the first and about 0 for the second, and separate the two. If it could
not see a planted delay, its verdict on real gateways would mean nothing.

### Requests {#requests}

Requests are `POST` chat completions with `max_tokens: 1` and one user message:
`ping`, or the content of `--prompt-file`, sent as a JSON string. An HTTP status
other than 200 aborts the run: an error is never timed as a success.

## Keys {#keys}

Keys are read from files, never from the command line or the environment. A
`KEYFILE` may be empty for an open endpoint. `gwbench llmflow-key` creates an
llmflow virtual key through the admin API (`--master-key-file`) and writes it to a
file of mode 0600. Keep key files outside the repository.

## An example {#example}

```sh
fakevllm --addr 127.0.0.1:18000 --out /tmp/requests.jsonl &
llmflow-gateway serve --config llmflow.hcl &      # forwards to 127.0.0.1:18000
gwbench llmflow-key --admin http://127.0.0.1:18080 --master-key-file ~/keys/master --out ~/keys/llmflow
gwbench run --direct http://127.0.0.1:18000/v1/chat/completions \
  --target 'v0.8.0|http://127.0.0.1:18080/v1/chat/completions|'"$HOME"'/keys/llmflow' \
  --rounds 2000 --warmup 200 --runs 8 --wait 2h --max-load 4
```

## The result {#result}

```text
run 7 kept: load 3.50 -> 3.70            # or: run N discarded: load a -> b
kept runs=8, rounds per run=2000, warm-up=200
run 0 overhead p50: v0.1.0=96.125µs v0.6.0=109.999µs
...
target          p50        p90        p99          # raw latencies, all kept runs
direct     81.833µs   93.041µs  138.458µs
v0.6.0    193.875µs  212.292µs  351.834µs
overhead vs direct, paired per round (all kept runs):
v0.6.0    111.042µs  129.541µs  261.624µs          # the answer
```

The per-run p50 lines show whether the runs agree; the last table is the result.
Report it with the load range of the kept runs, the machine, and the `GOMAXPROCS`
of each gateway.

## Dated results {#results}

Overhead is the paired difference against a direct call to the same upstream.
Each result is tied to its date and protocol; none is a promise for another
machine, another load or another request.

**Common to both dates.** Machine: Apple M4 Max (Mac16,5), 16 cores, 128 GiB RAM,
macOS 27.0.1. Each gateway ran with `GOMAXPROCS=1`; LiteLLM 1.106.2 ran as one
process. Upstream: `fakevllm` on loopback, answering instantly. Client: `gwbench`,
requests one at a time, target order shuffled every round. Each kept run is 2,000
rounds after a warm-up of 200, and the 1-minute load was at most 4 at its start and
at its end.

### 2026-10-23: LiteLLM and llmflow, no database {#r-1023}

- **Setups:** LiteLLM without PostgreSQL; llmflow v0.1.0 with the memory store.
- **Request:** the prompt `ping`, non-streaming.
- **Runs:** 8 kept (16,000 paired rounds), at loads 3.30 to 3.94. A first long run
  was discarded, because the load rose to 9.44.

| Overhead | p50 | p90 | p99 | per-run p50 |
| --- | --- | --- | --- | --- |
| LiteLLM 1.106.2 | 4.86 ms | 5.26 ms | 6.21 ms | 4.84 to 4.93 ms |
| llmflow v0.1.0 | 117 us | 161 us | 239 us | 115 to 120 us |

### 2026-10-26, 06:26: what the guardrails cost {#r-1026}

- **Setups:** llmflow v0.1.0, v0.6.0, and v0.6.0 with the `hide_secrets` and `pii`
  guardrails; memory store.
- **Request:** a realistic 2 KB prompt, non-streaming, holding an e-mail, an IBAN,
  a phone number and a fake GitHub token.
- **Runs:** 8 kept, at loads 3.50 to 3.91. One run was discarded: its load ended at
  4.40.
- **Control:** the fake upstream's recording shows that the guarded gateway sent
  `[EMAIL]`, `[IBAN]`, `[PHONE]` and `[TOKEN]` where the others sent the text in
  clear, so the guardrails really ran on that prompt.

| Overhead | p50 | p90 | p99 |
| --- | --- | --- | --- |
| v0.1.0 | 97 us | 115 us | 243 us |
| v0.6.0 | 111 us | 130 us | 262 us |
| v0.6.0 with `hide_secrets` and `pii` | 599 us | 621 us | 1,589 us |

On that request, the two guardrails cost about 490 us at p50 over the same release
without them. It is a measurement of one request shape.

**Not measured:** streaming, concurrency, database writes, and model-based
guardrails.

## Verified against a real vLLM (2026-10-26) {#verified-vllm}

The gateway's design rests on vLLM behaviours read in vLLM's documentation and
source. The `e2e/` suite of `bench` checks them live: it starts `llmflow-gateway`
with freshly generated keys, creates a virtual key per team, and drives it in
front of a running vLLM. Without `BENCH_VLLM_URL` every test skips, so CI runs
none of them.

| Variable | Meaning |
| --- | --- |
| `BENCH_VLLM_URL` | the chat vLLM (required) |
| `BENCH_VLLM_MODEL` | its model name (default: the first of `/v1/models`) |
| `BENCH_GATEWAY_BIN` | the `llmflow-gateway` binary (default: from `PATH`) |
| `BENCH_VLLM_MAX_NUM_SEQS` | the server's `--max-num-seqs` (the scenarios need 2) |
| `BENCH_VLLM_KV_SHORTAGE=1` | vLLM runs with a tiny KV cache: enables the preemption checks |
| `BENCH_VLLM_EMBED_URL`, `BENCH_VLLM_RERANK_URL` | optional embedding and score-model servers |

**Setup.** vLLM 0.32.1rc1 (nightly CPU image) in arm64 Docker on an Apple M4 Max;
models Qwen2.5-0.5B-Instruct (bfloat16), bge-small-en-v1.5 and bge-reranker-base;
gateway v0.8.0. The released v0.32.0 arm64 image failed every CPU generation
(`compute_slot_mapping`), which is why the nightly was used; that failure concerns
the CPU build only. The gateway configuration is organisation A (two teams, share
0.5) and B (one team, share 0.25) on a pool of capacity 2, so A reserves one slot
and every request of B borrows; `cache_isolation = "org"`.

Every ordering check has a **control** straight to vLLM that must come out the
other way, and the cache checks read vLLM's own counters, never timing.

| Check | Result | Evidence |
| --- | --- | --- |
| FCFS control, direct | pass | B3, sent first, served 4.7 s before A1 |
| Header over body | pass | body 100 and header -100: B3 first; body 100 alone: A1 first |
| Within-share before borrowers | pass | via the gateway, A1 (sent later) served 5.2 s before B3 |
| KV preemption | pass | 640-token KV cache, 2 x 384 tokens: the direct control preempts the newest (A1, paused 1 min 35 s); via the gateway the borrower B1 is preempted (1 min 22 s), A1 pauses at most 1.06 s and finishes first |
| Client priority ignored | pass | body, `vllm_xargs`, `extra_body` and header at -100: order unchanged |
| Prefix cache (vLLM's counters) | pass | same organisation 768/814; other organisation 0/814; a client's `cache_salt` 0; cached content without a salt 0; direct control 768/812 |
| Responses | pass | `GET /v1/responses/{id}` through the gateway answers 404; the direct control is readable |
| Messages, tokenize, embeddings, rerank | pass | complete SSE; 6 tokens on both sides; 384 dimensions; "Paris" ranked first (0.998) |
| Accounting | pass | vLLM 132 prompt / 44 generated, the gateway charged 132 / 44 |
| vLLM beyond `--max-num-seqs` | pass | 6 concurrent requests with 2 slots: six 200, no 429 or 503 |
| Fallback on saturation (gateway v0.11.0, #58) | pass | two borrowing streams of b1 fill pool `cpu` (C = 2); the next b1 request is served by `chat-spare` (`X-Llmflow-Model`), a1's within-share request stays on `chat`; `llmflow_fallbacks_total{reason="saturated"}` = 1. Run on 2026-10-26 with `--max-num-seqs 4` (both pools on one vLLM), with a gateway built from gateway#66, before the v0.11.0 release |

The vLLM-beyond-`--max-num-seqs` row is why saturation needs its own rule: vLLM
queues instead of refusing, so a [fallback]({{< relref "gateway.md" >}}#fallbacks)
on an outage never sees a queue. The last row is the check of that rule.
`TestFallbackOnSaturation` needs `BENCH_VLLM_MAX_NUM_SEQS=4` (a vLLM started with
`--max-num-seqs 4`) and a gateway of v0.11.0 or later; it skips otherwise.

**Not covered:** audio transcription (no Whisper model run on CPU) and throughput.

### Anthropic upstream {#verified-anthropic}

`e2e/anthropic` (bench v0.4.0) checks the gateway's
[Anthropic upstream]({{< relref "gateway.md" >}}#anthropic) against `fakevllm` playing
Anthropic. It needs `BENCH_GATEWAY_BIN` set to a gateway of v0.12.0 or later. It checks
that:

- the request leaves with the **organisation's key**, not the client's credential, and
  without the OAuth beta;
- the PII is masked, and `cache_salt` and `priority` are gone, before egress;
- streams work, and `/v1/chat/completions` is refused;
- the spend report, in USD, equals the price list applied to the usage the provider
  reported;
- a 429 with `retry-after` makes the gateway answer 429 itself, for that organisation
  only.

Result: on 2026-10-26 the fake-API suite passes against gateway v0.12.0. It proves the
gateway's behaviour against a **fake**; it says nothing about Anthropic's real API. A
second test, `TestGatewayInFrontOfRealAnthropic`, runs only when
`BENCH_ANTHROPIC_KEY_FILE` names a file holding a key (the model is
`BENCH_ANTHROPIC_MODEL`, default `claude-haiku-5-5`). **It was not run for v0.4.0, since
no key was available.**

### Slurm discovery {#verified-slurm}

Bench v0.5.0 tests the gateway's [Slurm discovery]({{< relref "gateway.md" >}}#slurm-discovery)
two ways.

**A fake slurmrestd, without a cluster.** The package `fakeupstream` provides
`Slurmrestd`: `GET /slurm/<version>/jobs/` answers the jobs given to `SetJobs`, in the
shape of `OPENAPI_JOB_INFO_RESP`; `job_state` is a flag array (`["RUNNING"]`,
`["RUNNING","COMPLETING"]`); a request must carry `X-SLURM-USER-TOKEN` set to `Token`;
`SetDown(true)` simulates an outage (it answers 502).

**`e2e/slurm_test.go` (`TestSlurmDiscoveryE2E`)** runs on a real Slurm node, as root. It
starts the gateway with a `slurm` model; submits, with `sbatch --uid`, a job of user
`alice` in account `physics` that claims the model (it must never be trusted); submits a
job of `svc_llm` named `llmflow:llama:18100:3` and waits for the pool's C to reach 3;
sends a chat request through it; runs `scancel`, and checks that C falls back to 0 and
the model is no longer served. It reads `BENCH_SLURM_RESTD`, `BENCH_SLURM_API` (default
`v0.0.42`), `BENCH_SLURM_JWT_FILE` (the output of `scontrol token username=svc_llm
lifespan=7200`), `BENCH_SLURM_JOB`, `BENCH_SLURM_MODEL`, `BENCH_SLURM_MEM` and
`BENCH_SLURM_WAIT`.

**Setup.** One Debian trixie container with Slurm 24.11.5 (`slurmctld`, `slurmd`,
`slurmrestd`, `slurm-wlm-jwt-plugin`, `munge`), `AuthAltTypes=auth/jwt`, slurmrestd run
as an unprivileged user with data_parser versions v0.0.40 to v0.0.42, on arm64, with
`--privileged` because slurmrestd needs to unshare namespaces. The job serves
Qwen2.5-0.5B-Instruct on CPU and reads its key into `VLLM_API_KEY`, never through
`--api-key`.

| Job (2026-10-26, gateway #77 head, Slurm 24.11.5, data_parser v0.0.42) | Result |
| --- | --- |
| `fakevllm` | admitted 2 s after `sbatch`, removed 2 s after `scancel` |
| vLLM CPU (`--enforce-eager`) | admitted 18 s after `sbatch` (once `/health` and `/v1/models` answered), chat request 200, removed 2 s after `scancel` |
| `alice`/`physics` job claiming the model | rejected once (`reason="account"`) and never contacted |

Two earlier vLLM runs failed inside vLLM (too little `/dev/shm`; torch inductor found no
C++ compiler). Each time the job stayed `RUNNING` with `/health` down, and the gateway
correctly never admitted it. This is one small instance on one node, as the README says;
it is no measure of a production cluster.

## fakevllm {#fakevllm}

```sh
fakevllm --addr 127.0.0.1:18000 --out requests.jsonl --model fake --reply "ok"
```

Flags: `--addr`, `--out`, `--model`, `--reply`, `--delay`, `--token-delay`,
`--max-body`.

| Route | Answer |
| --- | --- |
| `GET /v1/models` | one model, named by `--model` |
| `POST /v1/messages` | an Anthropic message; with `"stream": true`, the event sequence `message_start`, `content_block_start`, `content_block_delta`…, `content_block_stop`, `message_delta` (with `usage.output_tokens`), `message_stop` |
| any other `POST` | an OpenAI `chat.completion`; with `"stream": true`, `chat.completion.chunk` events (role, one chunk per word of `--reply`, finish), the usage chunk with empty `choices` when `stream_options.include_usage` is set, then `data: [DONE]` |

Since v0.4.0 the `/v1/messages` route can mimic Anthropic's behaviour:

| Flag | Effect |
| --- | --- |
| `--status N` | answer every message with that status (429, 529, ...) |
| `--retry-after S` | the `retry-after` header sent with it, in seconds |
| `--cache-read N`, `--cache-write-5m N`, `--cache-write-1h N` | prompt-cache tokens reported in `usage` |
| `--workspace ID` (or `per-key`) | the workspace named in the response headers |
| `--ratelimit-headers` | send the `anthropic-ratelimit-*` headers |

Streams carry a `ping` event after `message_start`.

Usage is predictable: one prompt token per whitespace-separated word of the
request's message and system strings, and one completion token per word of
`--reply`. `--delay` and `--token-delay` slow the answer and the stream.

Every request is appended to `--out` (mode 0600) as one JSON line,
`{"method","path","headers","body"}`; a body that is not JSON is recorded as a
string. **The recording holds the headers as received, `Authorization` included.**
That is by design, since it is a witness of what a gateway forwarded. Use test
keys only. If the record cannot be written, the request gets a 500, so a test
never reads a silent gap as "nothing was forwarded". Bodies are capped by
`--max-body` (1 MiB; larger ones get a 413).
