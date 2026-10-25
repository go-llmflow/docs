---
title: "Inference: vLLM and the gateway"
weight: 30
---

# Inference {#inference}

Shared inference is vLLM running inside Slurm, behind a gateway that is the
**only HTTPS entry point**. In the original design the gateway is LiteLLM; the
measurements below are of LiteLLM 1.106.2. The Go
[gateway]({{< relref "gateway.md" >}}) of this organisation implements the
fair-share policy that LiteLLM's free edition lacks.

## vLLM in Slurm {#vllm-in-slurm}

Each vLLM instance is a long Slurm job in the `svc_llm` account and the `service`
QOS. It is therefore accounted, cgroup-isolated, and never preempted. For a
whole server:

```bash
#!/bin/bash
#SBATCH -A svc_llm --qos=service
#SBATCH --nodes=1 --gres=gpu:b300:8 --exclusive
#SBATCH --time=7-00:00:00 --job-name=vllm-large-model
vllm serve <model> \
  --host <node-internal-ip> --port 8000 \
  --tensor-parallel-size 8 \
  --scheduling-policy priority \
  --gpu-memory-utilization 0.92 \
  --enable-prefix-caching \
  --max-num-seqs 256 \
  --api-key "$(cat /etc/vllm/internal-key)"
```

| Option | Role (from the engine arguments documentation [9]) |
| --- | --- |
| `--tensor-parallel-size` | number of tensor-parallel groups; 8 = one NVLink server |
| `--data-parallel-size` | replicas; for MoE models experts are spread over TP x DP |
| `--enable-expert-parallel` | expert parallelism instead of tensor parallelism for MoE layers |
| `--scheduling-policy` | `fcfs` (default) or `priority`: order by priority, then arrival |
| `--max-num-seqs` | maximum number of sequences per iteration |
| `--max-num-batched-tokens` | maximum number of tokens per iteration |
| `--gpu-memory-utilization` | share of GPU memory given to vLLM (default 0.92) |
| `--enable-prefix-caching` | reuses the KV cache of common prefixes (shared system prompts) |
| `--kv-cache-dtype` | KV cache type (`fp8`, `nvfp4`...): more context in memory |
| `--enable-lora`, `--max-loras` | per-team LoRA adapters on one base model |

## Per-request priority {#priority}

A request may carry an integer `priority`: the **lower** it is, the sooner the
request runs (default 0). The `X-Vllm-Priority` header takes precedence over the
JSON body. Any non-zero value is an error if the server was not started with
`--scheduling-policy priority` [10].

What the scheduler does, from its source code [25]:

- the waiting queue is a heap sorted by `(priority, arrival_time)`: the smallest
  priority comes out first, arrival breaks ties;
- when a *running* request cannot find KV-cache blocks any more, vLLM preempts
  the running request with the **largest** `(priority, arrival_time)` and puts
  it back at the head of the queue;
- a high-priority *waiting* request does not preempt a running request: it takes
  the first slot that frees up.

Priority therefore orders admission and designates the victims when memory runs
short. It does not interrupt generations in progress. Since a generation lasts a
few seconds, the effect on latency is quick. We use it to put interactive use
(VS Code, chat) ahead of API batch processing, and for the split between
institutions ([below](#fair-share-inference)).

## Securing vLLM {#securing-vllm}

{{< callout type="warning" >}}
`--api-key` protects only the `/v1`, `/v2` and `/inference` paths. Other
endpoints of the same server, such as `/invocations`, **are not authenticated**.
The documentation asks not to rely on `--api-key` alone and to put vLLM behind a
proxy [10].
{{< /callout >}}

Hence the rules: vLLM listens on the cluster's internal interface only; a
firewall lets only the gateway reach the vLLM ports; the internal key is read
from a file, never passed in clear in the environment or in shell history.

### Prefix-cache isolation {#prefix-cache-isolation}

With `--enable-prefix-caching`, vLLM reuses cached KV blocks across requests that
share a prompt prefix. That reuse is a **timing side channel between tenants**: a
caller can tell from the time to first token whether a guessed prefix was cached,
hence whether another institution sent it (vLLM's
[security guide](https://docs.vllm.ai/en/stable/usage/security/); CVE-2025-46570).
On a cluster shared by A and B this must be closed. The vLLM mechanism is the
`cache_salt` request field, mixed into the hash of the first KV block. The Go
[gateway]({{< relref "gateway.md" >}}#prefix-cache-isolation) (v0.3.0) always sets
a stable per-tenant salt (an HMAC of the tenant under a secret key file of at
least 32 random bytes) and drops the client's own; `cache_isolation` chooses the
tenant (`org` by default, `team`, `key`, or `none`). Two limits to keep in mind:
users inside one scope can still probe each other (`team` or `key` narrow it, at
the cost of cache reuse), and `cache_salt` is not a hard isolation boundary, which
only a vLLM instance per tenant is. Rotating the key empties everyone's cache.

## Elastic pool {#elastic-pool}

At night and at weekends interactive inference drops. An `scontrol update` or a
`scrontab` can stop one instance in two: 8 GPUs go back to the `borrow` QOS,
without changing A's and B's guarantees. On the gateway side the second instance
leaves the routing; its requests go to the first, which stays available.

## The gateway {#the-gateway}

The gateway is the only HTTPS entry point of inference. It carries:

- **identity**: a virtual key per user, attached to a *team* = an institution;
- **quotas**: rate (tokens/min, requests/min, concurrent requests) and a budget
  per team;
- **the 81.25 / 18.75 split** under saturation;
- **accounting**: spend in tokens per team, the basis of the
  [re-charging]({{< relref "accounting.md" >}}).

{{< callout type="warning" >}}
With LiteLLM, budgets are enforced only with a **Postgres** database. Without one
they are not enforced at all [11]. With several LiteLLM instances **Redis** is
needed too, or each instance counts on its own and limits can be exceeded. Enable
`fail_closed_budget_enforcement` and `fail_closed_rate_limit_enforcement`: if
Redis or the database fails, the gateway answers 503 instead of letting requests
through uncounted [11].
{{< /callout >}}

### Models and routing {#routing}

This is LiteLLM's configuration; the interactive group pins priority 0 and the
batch group pins priority 10, on the same instances.

```yaml
model_list:
  # Interactive group (VS Code, chat): vLLM priority pinned to 0
  - model_name: large-model
    litellm_params:
      model: hosted_vllm/<model>
      api_base: http://gpu01.internal:8000/v1
      api_key: os.environ/VLLM_INTERNAL_KEY
      extra_headers: {X-Vllm-Priority: "0"}
    model_info: &cost            # internal cost, calibrated on the accounting page
      input_cost_per_token: <c_in>
      output_cost_per_token: <c_out>
      cache_read_input_token_cost: <c_cache>
  - model_name: large-model
    litellm_params:
      model: hosted_vllm/<model>
      api_base: http://gpu02.internal:8000/v1
      api_key: os.environ/VLLM_INTERNAL_KEY
      extra_headers: {X-Vllm-Priority: "0"}
    model_info: *cost
  # Batch group (API scripts): same instances, priority 10
  - model_name: large-model-batch
    litellm_params:
      model: hosted_vllm/<model>
      api_base: http://gpu01.internal:8000/v1
      api_key: os.environ/VLLM_INTERNAL_KEY
      extra_headers: {X-Vllm-Priority: "10"}
    model_info: *cost
  # (... same for gpu02 ...)

router_settings:
  routing_strategy: least-busy
  allowed_fails: 3
  cooldown_time: 30
  redis_host: os.environ/REDIS_HOST
  redis_port: os.environ/REDIS_PORT
  redis_password: os.environ/REDIS_PASSWORD

general_settings:
  master_key: os.environ/LITELLM_MASTER_KEY
  database_url: os.environ/DATABASE_URL
```

Two deployments sharing a `model_name` form a group, and the routing strategy
spreads requests between them. `least-busy` picks the deployment with the fewest
calls in progress. A failing deployment goes into *cooldown* alone, not the whole
group [13].

{{< callout type="warning" >}}
**Zero costs = budgets ignored.** A model whose two costs are 0 is subject to no
budget check [11]. The model is hosted locally, but it must still be given an
**internal cost**, calibrated in GPU-hours (see
[accounting]({{< relref "accounting.md" >}})). Otherwise `max_budget` protects
nothing. Costs are declared under `model_info` [22]. **Measured on LiteLLM
1.106.2**: without a cost, spend stays at 0; with a cost, it equals exactly
`3 c_in + 1 c_out` for 3 input tokens and 1 output token.
{{< /callout >}}

### Teams, keys and limits {#teams-keys}

```bash
# One team per institution (indicative values)
curl -X POST https://llm.example.org/team/new \
  -H "Authorization: Bearer $(cat ~/.config/litellm/master)" \
  -H 'Content-Type: application/json' \
  -d '{"team_alias": "instA", "max_budget": 1300, "budget_duration": "30d",
       "max_parallel_requests": 200, "metadata": {"priority": "instA"}}'

curl -X POST https://llm.example.org/team/new ... \
  -d '{"team_alias": "instB", "max_budget": 300, "budget_duration": "30d",
       "max_parallel_requests": 50, "metadata": {"priority": "instB"}}'

# One key per user, attached to their team
curl -X POST https://llm.example.org/key/generate ... \
  -d '{"team_id": "<instB-id>", "key_alias": "bob-vscode",
       "rpm_limit": 60, "max_parallel_requests": 8}'
```

A key that carries a `team_id` is governed by the team's budget, not by its
owner's personal budget. The limits `tpm_limit`, `rpm_limit` and
`max_parallel_requests` can be set on the team, the user or the key. For one
per-model limit, the key's wins over the team's [11]. The budget resets at each
`budget_duration` [11].

**Do not pass the secret on the command line.** The examples read the master key
from a file readable by its owner only, never from an environment variable or an
argument visible in `ps`.

### Fair-share for inference: reservation by priority {#fair-share-inference}

LiteLLM's `dynamic_rate_limiter_v3` shares a model's TPM/RPM capacity between
teams. **Below** a saturation threshold anyone may borrow the free capacity.
**Above** it, each priority is brought back to its reserved share [14]. It is a
direct transposition of the Slurm model: lending of surplus, guarantee at
saturation.

```yaml
litellm_settings:
  callbacks: ["dynamic_rate_limiter_v3"]
  priority_reservation:
    instA: 0.8125
    instB: 0.1875
  saturation_threshold: 0.8
```

A key's priority comes from its team's `metadata.priority`, which wins over the
key's. Keys with no priority fall into a common pool (`default_priority`, 0.25 by
default) [14]. The model's capacity (`rpm`/`tpm` of the deployments) is measured
in acceptance on your B300s, at the target latency.

{{< callout type="warning" >}}
(1) Priority reservation is marked **beta** and requires a **LiteLLM Enterprise
licence** (30-day free trial) [14]. (2) Borrowed capacity is not taken back when
strict mode engages, and TPM is checked at admission: a burst can exceed the
limit. (3) With several LiteLLM instances, strict mode may engage late on a node
that has not seen the traffic (60 s cache by default) [14].
{{< /callout >}}

## What LiteLLM forwards to vLLM: measurements {#what-litellm-forwards}

The documentation does not say what reaches vLLM. We **measured** it on
**23 October 2026**. A real **LiteLLM 1.106.2** was placed in front of a fake
vLLM written in Go that records every request it receives: path, headers, JSON
body. Control: a request sent directly to the fake vLLM with `priority: 99` and
the header `X-Vllm-Priority: 99` shows up intact. So the instrument does see what
it looks for.

| Sent by the client to LiteLLM | Received by vLLM |
| --- | --- |
| `"priority": 10` in the body | **nothing**: LiteLLM consumes this field for *its own* beta scheduler [15] (response header `x-litellm-request-prioritization-used`, +300 ms latency) |
| `"extra_body": {"priority": 3}` | `"priority": 3` in the body |
| deployment `extra_body: {priority: 5}` + client `extra_body` at 0 | `"priority": 0`: **the client overrides the deployment's value** |
| deployment `extra_headers: {X-Vllm-Priority: "6"}` | `X-Vllm-Priority: 6` |
| same + client `X-Vllm-Priority: 0`, client headers forwarded [23] | `X-Vllm-Priority: "0, 6"`: values merged, no longer an integer |
| same + client `X-Vllm-Priority: 0` and `x-pass-X-Vllm-Priority: 1`, **default settings** | `X-Vllm-Priority: 6`: the client's headers do not pass |

**Conclusion: the only safe configuration.**

1. Pin the priority **by header** in the deployment (`extra_headers`), never by
   `extra_body`.
2. Leave `forward_client_headers_to_llm_api` **disabled** (the default).
3. A client can still slip `priority` into the body through `extra_body`. But
   vLLM gives the header precedence over the body [10], so the pinned header wins.
4. Do not use LiteLLM's own `priority` field: it enables a beta scheduler that
   adds latency.

The class of service (interactive or batch) is then chosen by the **model name**
(`large-model` or `large-model-batch`). The keys of batch scripts are restricted
to the second.

## Split with lending, without a licence {#fairq}

Free-edition LiteLLM provides keys, teams, budgets and static limits (`tpm_limit`,
`rpm_limit`, `max_parallel_requests`) [11]. What is missing is the **lending**.
Static limits at 81.25 / 18.75 leave an absent institution's share unused.

The proposal is to reproduce the Slurm scheme (guaranteed share plus preemptible
borrowing) at request level: a small relay between the gateway and vLLM counts the
requests in flight per institution. Let `C` be the sustainable concurrency of an
instance, measured (see [accounting]({{< relref "accounting.md" >}}#calibration)).

- If the institution has fewer than `p x C` requests in flight (`p` = 0.8125 or
  0.1875), the request is **within its share**: `X-Vllm-Priority` is 0
  (interactive) or 1 (batch).
- Otherwise it is **borrowing**: priority 10 (interactive) or 11 (batch).
- The relay **overwrites** any `priority` and any `X-Vllm-Priority` coming from the
  client.

vLLM then does exactly what is expected (see [priority](#priority)): within-share
requests are admitted before borrowing ones; when KV memory runs short, borrowers
are preempted first. Without competition an institution takes everything; under
saturation each gets its share back.

This is the policy the [gateway]({{< relref "gateway.md" >}}#fair-share-with-borrowing)
(v0.8.0) implements in its `fairshare` package, with the two refinements the
simulator forced (the reservation test counts within-share requests, and usage is
charged at release). It replaces LiteLLM and the relay here: it holds the keys,
organisations, teams and budgets itself, and the shares belong to the
**organisations** (the institutions). For this cluster, its configuration (HCL2,
not YAML) would be as follows, with the 256-request capacity of the gateway's
example. This block was run through `llmflow-gateway check-config` v0.8.0, with
the key files present:

```hcl
master_key_file     = "/etc/llmflow/master.key"
cache_isolation     = "org"
cache_salt_key_file = "/etc/llmflow/cache-salt.key"

org "instA" {
  share = 0.8125
  team "a-physics" {}
  team "a-biology" {}
}
org "instB" {
  share = 0.1875
  team "b-lab" {}
}

pool "b300" { capacity = 256 }

model "large-model" {
  upstream_model = "meta-llama/Llama-3.3-70B-Instruct"
  pool           = "b300"
  class          = "interactive"
  key_file       = "/etc/llmflow/vllm.key"
  cost {
    input_per_mtok  = 0.02
    output_per_mtok = 0.20
    cached_per_mtok = 0.002
  }
  upstream "gpu01" { base_url = "http://gpu01.internal:8000" }
  upstream "gpu02" { base_url = "http://gpu02.internal:8000" }
}

# the same upstreams again, with class = "batch"
model "large-model-batch" {
  upstream_model = "meta-llama/Llama-3.3-70B-Instruct"
  pool           = "b300"
  class          = "batch"
  key_file       = "/etc/llmflow/vllm.key"
  cost {
    input_per_mtok  = 0.02
    output_per_mtok = 0.20
    cached_per_mtok = 0.002
  }
  upstream "gpu01" { base_url = "http://gpu01.internal:8000" }
  upstream "gpu02" { base_url = "http://gpu02.internal:8000" }
}
```

The `cost` values are in GPU-hours per million tokens; calibrate them as in the
[accounting]({{< relref "accounting.md" >}}#calibration) page. Note the gateway's
[limitations]({{< relref "gateway.md" >}}#limitations), in particular one replica
per pool.

{{< callout type="info" >}}
**To validate: how the relay knows the institution.** Two routes. (a) *One
deployment per institution* (`api_base: http://relay:9000/instA/v1`), reachable
only by the team's keys: safe, but one must check how LiteLLM exposes a single
model name to two teams. (b) The `X-Litellm-User_api_key_*` headers that LiteLLM
adds with `add_user_information_to_llm_headers`: we observed them (user id, key
fingerprint). The presence of the *team* id could not be verified, because our test
ran without a database, hence without teams.
{{< /callout >}}

## Alternatives without a licence {#alternatives}

Synthesis of each product's official documentation, consulted on 23 October 2026.
Y = native; P = partial; -- = absent; $ = paid offer. Columns: keys and teams,
rate limits, budgets, spreading between instances, reservation with lending.

| Gateway | Keys | Limits | Budgets | Spread | Lending | Remarks |
| --- | --- | --- | --- | --- | --- | --- |
| LiteLLM (MIT, Python) | Y | Y | Y | Y | $ | lending = Enterprise beta [14] |
| Bifrost (Apache-2.0, Go) [26] | ? | Y | Y | P | -- | teams described in the free docs, but "governance across teams and customers" is listed as Enterprise: **to test** |
| Apache APISIX (Apache-2.0) [27] | Y | Y | P | Y | -- | standalone YAML mode + Redis, no Kubernetes |
| Envoy AI Gateway [28] | -- | Y | P | P | -- | token-based limits; Kubernetes required |
| Gateway API Inference Ext. / llm-d [29] | -- | -- | -- | Y | P | routing on KV cache; Kubernetes required; weighted fairness announced as "planned" |
| vLLM production-stack [30] | -- | -- | -- | Y | -- | router only |
| Portkey (MIT) [31] | $ | ? | ? | Y | -- | key management is paid |
| Kong AI Gateway [32] | $ | $ | -- | $ | -- | advanced plugins are Enterprise |

**No free gateway natively provides lending with a guarantee.** The only project
that comes close (llm-d, weighted fairness) requires Kubernetes and still calls it
"planned". For a cluster without Kubernetes: **free LiteLLM plus a relay** covers
the five needs without a licence and keeps the keys, budgets and accounts
described here. Bifrost, written in Go, is an alternative to evaluate *if* its
teams turn out to be free; the relay would plug into it the same way.
