---
title: "Security audit, acceptance tests, governance"
weight: 60
---

# Security audit, acceptance and governance {#audit}

## Security audit {#security-audit}

| Surface | Risk | Measure |
| --- | --- | --- |
| Shared GPUs | a job reads the GPU memory of another institution | `ConstrainDevices=yes` [7]; no *shard* between institutions [5] |
| vLLM | unauthenticated endpoints outside `/v1` [10] | internal listening only; firewall: only the gateway reaches it |
| Gateway | quota bypass during a Redis/DB outage [11] | *fail-closed* options; Redis and Postgres made redundant |
| Gateway | budgets inoperative on a zero-cost model [11] | non-zero internal cost ([accounting]({{< relref "accounting.md" >}})) |
| Keys | leak through a URL, a command line or an environment variable | files with mode 0600; Continue secrets; rotation; one key per use |
| Logs | confidential prompts stored in the database | disable message-content logging; keep only spend metadata |
| vLLM prefix cache | timing side channel between institutions: a caller detects a cached prefix by its time to first token (CVE-2025-46570) | per-tenant `cache_salt` set by the gateway ([prefix-cache isolation]({{< relref "inference.md" >}}#prefix-cache-isolation)); a vLLM instance per tenant if hard isolation is needed |
| Login node | computation outside Slurm | per-user cgroup limits; SSH access to GPU nodes reserved to jobs |
| Preemption | loss of work of borrowed jobs | `GraceTime` plus checkpoint; `PreemptExemptTime` |
| vLLM priority | a client gives itself priority 0 through `extra_body` (**measured**, [measurements]({{< relref "inference.md" >}}#what-litellm-forwards)) | priority pinned by *header*; client-header forwarding disabled; the relay (or the Go gateway) overwrites |
| Header forwarding | the client's `x-pass-*` and `x-*` reach vLLM if forwarding is on [23] | leave `forward_client_headers_to_llm_api` disabled |
| FP64 usage | double-precision codes on B300 (10 TFLOPS per server) [12] | usage rule in the agreement |

**Chaos to play during acceptance.** Kill a vLLM instance under sustained traffic
(the routing must fail over); cut Redis (the *fail-closed* mode must answer 503,
not let requests through); preempt a borrowed job in the middle of writing a
checkpoint (the resume must read the last *complete* checkpoint); restart
`slurmdbd` during a mass submission.

## Acceptance tests: prove the guarantee holds {#acceptance}

Every guarantee is proven by a test that *could* fail:

1. **B's guarantee.** A fills the 32 GPUs (13 in `instA`, 19 in `borrow`). B
   submits 3 GPUs in `instB`. Measure the start delay: it must stay under 36
   minutes, and under 6 minutes if the borrowed jobs are older than 30 minutes.
2. **Negative control.** The same scenario with the `borrow` QOS without
   `Preempt`: B must wait. If B starts anyway, test 1 was not measuring
   preemption.
3. **Cap.** B submits 8 GPUs in `instB`: immediate rejection (`DenyOnLimit`).
4. **Fair-share.** After a week in which B borrows heavily, `sshare -l` must show
   `LF_B < 1`, and A's `borrow` jobs must go ahead of B's (`sprio`).
5. **Billing.** A job of 1 GPU and 64 cores must be billed 4: the `billing=` field
   of `AllocTRES` in `sacct`.
6. **Inference.** Saturate the model with a key of A; check that a key of B still
   gets about 18.75 % of the throughput (with the Enterprise reservation, or the
   relay) or its static limit.
7. **vLLM priority.** During saturation, a request of the `large-model` group
   (priority 0) must have a time to first token markedly lower than a request of
   `large-model-batch` (priority 10).
8. **Inviolable priority.** Send to `large-model-batch` a request with
   `extra_body: {"priority": 0}` and a header `X-Vllm-Priority: 0`. It must stay
   processed at priority 10. Control: the same request sent directly to vLLM goes
   ahead.
9. **B's window.** One hour before `instB_weekly` opens, submit in `borrow` a 3 h
   job and a 30 min job. Only the second must start on the reserved server. At
   opening, an 8-GPU job of B must start immediately.

**Performance.** Compare throughput (tokens/s) and latency (time to first token,
p50/p99) of vLLM behind the gateway with those of vLLM direct, at equal load. The
gateway's overhead must stay negligible next to generation; if it does not,
increase the number of gateway workers before touching vLLM.

## Governance {#governance}

- **Agreement between institutions**: shares (26/6), guaranteed delay (36 min),
  size of the inference pool, half-life (14 days), the quarterly compensation
  rule, and the possible reservation of a whole server for B.
- **Quarterly technical committee**: real consumption against share, median wait
  times per institution and per QOS, preemption rate, inference saturation.
- **Change of shares** (the purchase of a 5th server, say): change the `Fairshare`
  and `GrpTRES` values. No redeployment is needed.
