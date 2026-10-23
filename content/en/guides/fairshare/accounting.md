---
title: "One accounting, in GPU-hours"
weight: 40
---

# One accounting {#accounting}

Batch is counted in GPU-equivalent hours (`sreport`, after
`TRESBillingWeights`). Inference is counted in tokens, per team (the gateway's
spend logs). They are reconciled as follows:

1. Over the period (a month), read the GPU-hours of the `svc_llm` account:
   `H_svc`.
2. Read each team's spend in the gateway: `D_A`, `D_B`.
3. Charge `H_svc x D_A / (D_A + D_B)` to A, and the remainder to B.
4. Compare each institution's total consumption (batch plus charged inference)
   with its share (81.25 % / 18.75 %).

## Calibrating the internal cost {#calibration}

**Goal:** 1 unit of gateway spend = 1 GPU-hour. A saturated 8-GPU instance then
"spends" 8 units per hour, and `max_budget` becomes a quota of inference
GPU-hours, directly comparable with batch.

**1. Measure the capacity.** Once per model and per configuration (TP, KV-cache
dtype), with `vllm bench serve` [24]. Use a mix of lengths representative of the
real usage (for example 2,000 input tokens and 300 output tokens for a code
assistant). Raise `--max-concurrency` up to the limit of the service target, for
instance a p99 time to first token under 2 s:

```bash
vllm bench serve --backend openai-chat --endpoint /v1/chat/completions \
  --host gpu01.internal --port 8000 --model <model> \
  --dataset-name random --random-input-len 2000 --random-output-len 300 \
  --num-prompts 2000 --max-concurrency 64 --save-result
```

At the chosen concurrency `C` (which also serves the
[fair-share relay]({{< relref "inference.md" >}}#fairq)), read the output
throughput `T_out` and the total throughput `T_tot` in tokens/s, printed by the
tool [24]. The input throughput is `T_in = T_tot - T_out`.

**2. Split the cost of an instance-hour.** Prefill (input) costs much less per
token than generation. Fix a ratio `r = c_out / c_in`, typically `r = 4`, and
solve:

```text
3600 (c_in T_in + c_out T_out) = 8
  =>  c_in = 8 / (3600 (T_in + r T_out)),   c_out = r c_in
```

*A fictitious example*, to illustrate the computation (the real figures come out
of step 1): `T_in = 40,000`, `T_out = 6,000` tokens/s, `r = 4`. Then
`c_in = 8 / (3600 x 64,000) = 3.5e-8` and `c_out = 1.4e-7` unit per token.

**3. Tokens served from the prefix cache.** LiteLLM warns that a model with
custom pricing bills these tokens at 0 as long as `cache_read_input_token_cost`
is not set (a warning observed at LiteLLM 1.106.2 start-up). Set it to about
`c_in / 10`: these tokens are not recomputed, but they occupy KV memory.

**Why not a cost per second?** LiteLLM accepts `cost_per_second` [22]. But vLLM
processes dozens of requests *simultaneously* on one instance: billing each by its
duration would make it pay for the whole instance, and the sum would far exceed
the real 8 GPU-hours. The per-token cost is the right model.

**Recalibrate** at each change of model, vLLM version or KV-cache dtype, and check
once a month. The GPU-hours of the `svc_llm` account read from `sreport` should
stay close to the month's total gateway spend divided by the average occupancy.

## Compensation {#compensation}

If, over a quarter, an institution consumed markedly more than its share through
inference, the committee may correct the Slurm `Fairshare` values of the next
quarter. Batch fair-share already rebalances by itself within the half-life
window: compensation is only an exceptional adjustment.
