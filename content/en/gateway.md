---
title: "Gateway"
weight: 20
---

# gateway {#gateway}

An OpenAI-compatible gateway in pure Go (`CGO_ENABLED=0`) for vLLM clusters
shared by several institutions: virtual keys, budgets, GPU-hour accounting and
**fair-share scheduling with reservation and borrowing**.
[Repository →](https://github.com/go-llmflow/gateway)

## Status {#status}

**In development. v0.1.0 is not released yet.** The state of `main`, from the
merged pull requests:

| Pull request | Content | State |
| --- | --- | --- |
| #1 | skeleton: configuration, cobra command line, CI | merged |
| #2 | `fairshare`: reservation with borrowing, mapped to vLLM priorities | merged |
| #3 | `openai`: request rewrite, SSE pass-through with usage capture, fuzzing | merged |
| #4 | `store`: teams, hashed virtual keys, budgets, usage (memory and PostgreSQL) | open |

What the repository says it will do, and what is **not** yet in `main`: virtual
keys, teams, budgets and rate limits kept in PostgreSQL arrive with #4 and the
pull requests after it. Until they are merged, treat them as design.

What exists today:

```sh
go install github.com/go-llmflow/gateway/cmd/llmflow-gateway@latest
llmflow-gateway version
llmflow-gateway check-config --config gateway.yaml
```

The command line is a cobra root with two subcommands, `version` and
`check-config`. Configuration keys are not listed on this page: the schema is
still moving, and `check-config` is the authority on what a file may hold.

## Architecture {#architecture}

```text
OpenAI client ──HTTPS──> gateway ──internal network──> vLLM instances
 (VS Code,                  │  authenticate the key, find its team
  scripts)                  │  rewrite the request, choose the priority
                            │  stream the answer back, count the usage
                            └─> store (teams, keys, budgets, usage)
```

The code is split into packages, one per concern:

| Package | Role |
| --- | --- |
| `config` | YAML configuration with strict decoding |
| `fairshare` | the reservation and borrowing policy, and a simulator of vLLM's scheduler |
| `openai` | the request rewrite and the usage capture |
| `cmd/llmflow-gateway` | the command line |

### Configuration {#configuration}

Decoding is strict: **an unknown key is an error**, not a silent default.
Validation checks that team shares sum to at most 1, that pools have a positive
capacity, that model groups are bound to declared pools, and that upstream URLs
are http or https with no credentials, query or fragment. Secrets are read from
files only (a bounded, single-line read whose content never appears in an
error message), never from the command line or the environment.

### The request rewrite {#the-request-rewrite}

`openai.Rewrite` re-encodes the body from its parsed form, so that duplicate
keys and escaped spellings of a field name (`"priority"` written with a
Unicode escape) cannot reach vLLM. It:

- maps the model name the client used to the upstream model;
- **strips** `priority`, `extra_body`, `request_id`, `kv_transfer_params`,
  `ec_transfer_params` and `vllm_xargs`, fields taken from vLLM's
  `ChatCompletionRequest`;
- for streamed requests, sets `stream_options.include_usage=true` so that the
  usage is always reported, and remembers whether the client asked for it.

### Streaming and usage {#streaming-and-usage}

`openai.CopySSE` forwards a server-sent-events stream event by event, with a
flush after each one. Nothing beyond one event is held; an event over 1 MiB is
streamed through unexamined. It captures the usage chunk and drops it if the
gateway, not the client, asked for it. The pass-through is byte-identical (a
fuzzed invariant). `ParseUsage` reads a non-streaming response; usage with
negative counts, or cached tokens above prompt tokens, is refused.

## How fair-share with borrowing works {#fair-share-with-borrowing}

The problem: a pool of vLLM capacity is co-owned. Static per-team limits leave
the share of an absent team unused; no limits let one team starve the others.

The gateway keeps, for each team, the number of its requests in flight, and
compares it with the team's **reservation**: its share times the capacity `C`
of the pool.

| Situation | vLLM priority (lower runs first) |
| --- | --- |
| within the reservation | **0** for an interactive request, **1** for a batch one |
| above the reservation (borrowing) | **10 + 10 × rank** for interactive, one more for batch |

`rank` is the dense rank of the team's *level fairshare*, `LF = S / U`
(its share over its decayed usage, the quantity Slurm's Fair Tree computes),
among the teams of the pool. The usage decays with a half-life, as an exact
exponential. (Slurm's `1 - 0.693/hl` per period is its first-order expansion.)

vLLM then does what the numbers say. Its v1 scheduler keeps the waiting
requests in a min-heap on `(priority, arrival time)`. A running request that
cannot get KV-cache blocks preempts the running request with the **largest**
`(priority, arrival)`. A waiting request never preempts a running one. So
within-share requests are admitted first, and when memory runs short the
borrowers are the victims. With no competition a team takes everything; under
saturation each team falls back to its share. The [guide]({{< relref "guides/fairshare/inference.md" >}})
explains vLLM's priority semantics from its source.

### Two choices forced by the simulator {#two-choices}

The `fairshare` package ships a deterministic simulator of vLLM's v1 priority
scheduler, and two natural-looking rules failed in it:

1. **The reservation test counts within-share in-flight requests, not total
   in-flight.** With the total-count rule, any closed-loop client pool larger
   than the reservation keeps the team's count above it forever, so every
   request after the first ones was sent as a borrower: team B obtained 4.35 %
   instead of 18.75 %.
2. **Usage is charged from consumption at release, not from time in flight.**
   A request waiting in vLLM's queue is in flight for the gateway but consumes
   nothing. Integrating in-flight time made the flooding team look as consumed
   as the team it starved (B got 7.35 %).

### Simulator results {#simulator-results}

Capacity C = 16, KV cache of 1000 blocks, 64 closed-loop clients per team, team
B arriving after A had saturated the pool. Shares 0.8125 and 0.1875, tolerance
0.03.

| Scenario | Result |
| --- | --- |
| A alone, B alone | KV utilisation 0.98 and 0.97; mean running 40.5 and 39.0, against reservations of 13 and 3 |
| both saturating | running fraction A 0.789, B 0.211 |
| preemptions | 24 614, all of borrowers |
| control: every request at priority 0 | B gets 0.498, its client fraction and not its share |

The control matters: it shows that the 0.211 comes from the priorities, not
from the simulator being unable to produce anything else.

## Security notes {#security}

- **The client never sets its priority.** `priority`, `extra_body` and the other
  stripped fields are removed before the request leaves the gateway, and the
  priority is decided by the gateway alone. This closes a hole that was
  [measured on LiteLLM]({{< relref "guides/fairshare/inference.md" >}}#what-litellm-forwards):
  a client that puts `priority` in `extra_body` overrides a priority pinned in the
  body of the deployment.
- **Re-encoding, not patching**, so a differently spelled duplicate of a field
  cannot slip through; `FuzzRewrite` checks that no stripped field survives, that
  `include_usage` is forced for streams, and that the rewrite is idempotent.
- **Fuzzed pass-through.** `FuzzCopySSE` checks that the stream is forwarded
  byte for byte. The seed corpus is committed under `openai/testdata/fuzz` and CI
  runs each target for 30 seconds.
- **Strict configuration.** Unknown keys are rejected; upstream URLs cannot carry
  credentials; secrets come from files.
- **CI** runs gofmt, vet, the race detector on amd64 and arm64, a `CGO_ENABLED=0`
  cross-build for linux/amd64 and linux/arm64, and govulncheck.
- **vLLM must not be reachable except through the gateway.** vLLM's `--api-key`
  protects only some paths; see the
  [guide]({{< relref "guides/fairshare/inference.md" >}}#securing-vllm).

## License {#license}

BSD-3-Clause.
