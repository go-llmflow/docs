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

This page is written from the repository's README at **v0.1.0**.

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

## fakevllm {#fakevllm}

```sh
fakevllm --addr 127.0.0.1:18000 --out requests.jsonl --model fake --reply "ok"
```

| Route | Answer |
| --- | --- |
| `GET /v1/models` | one model, named by `--model` |
| `POST /v1/messages` | an Anthropic message; with `"stream": true`, the event sequence `message_start`, `content_block_start`, `content_block_delta`…, `content_block_stop`, `message_delta` (with `usage.output_tokens`), `message_stop` |
| any other `POST` | an OpenAI `chat.completion`; with `"stream": true`, `chat.completion.chunk` events (role, one chunk per word of `--reply`, finish), the usage chunk with empty `choices` when `stream_options.include_usage` is set, then `data: [DONE]` |

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
