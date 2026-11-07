---
title: "Several machines, several contributors"
weight: 26
---

# Shared purchase: a machine with 8 RTX PRO 6000 {#shared-purchase}

This page follows a real case. A machine with **8 NVIDIA RTX PRO 6000 Blackwell GPUs** will be
bought by several institutions, each paying a part. The rule they want: **each institution gets at
least the share of the machine that it paid for, and may use the GPUs that are free.** It is not
installed yet, so everything about the machine below is the plan ("will"), not a measurement.

It uses [per-pool shares](#per-pool-shares), introduced in **gateway v0.24.0** (#125), and the
example `examples/two-machines.hcl` of the gateway repository.

## What the hardware says {#hardware}

Every figure below comes from a page opened on 2026-11-01. None is a measurement of ours.

- **NVIDIA product page**
  ([RTX PRO 6000 Blackwell Server Edition](https://www.nvidia.com/en-us/data-center/rtx-pro-6000-blackwell-server-edition/)):
  96 GB of GDDR7 per GPU, 1597 GB/s of memory bandwidth, PCIe Gen 5, MIG up to 4 instances, up to
  600 W, FP4 and FP8. **NVLink is not mentioned** on that page.
- **NVIDIA [CUDA GPUs](https://developer.nvidia.com/cuda-gpus) page:** compute capability **12.0**,
  for both the Server and the Workstation editions.
- **vLLM GPU installation page:** compute capability 7.5 or higher; Blackwell needs CUDA 12.8 or
  higher. That page does not name `sm_120`, so **check that the vLLM build you deploy supports
  compute capability 12.0** before the machine arrives.

### Consequence for the layout {#layout}

Without NVLink, tensor parallelism between the GPUs goes over PCIe. The plan is therefore **several
independent vLLM instances of 1 or 2 GPUs each**, all upstreams of **one** llmflow pool, rather than
one instance spread over eight GPUs. Then **measure C**, the sustainable concurrency of the pool
(the concurrency it serves without preemption), **per instance, and add them up**. This page gives no
throughput figure: none exists for this machine yet, and inventing one would put a made-up number
into every reservation.

## The apportionment {#per-pool-shares}

The machine is one pool. Its `shares` are what each institution paid, and apply **to this pool only**:

```hcl
pool "rtx6000" {
  capacity = 64 # 8 GPUs, 8 concurrent requests each (an assumption to replace by the measured C)
  shares   = { etabA = 0.5, etabC = 0.3, etabD = 0.2 }
  borrow   = "shareholders" # or "all" (the default)
}
```

The rules, from the gateway README at v0.24.0:

- A pool **without `shares`** keeps the organisations' `share`: a configuration written before
  v0.24.0 means what it meant.
- With `shares`, the checks are those of the organisations' shares: every name is a declared
  organisation, each share is between 0 and 1, and the sum is at most 1 (a machine not fully
  allocated is allowed).
- An organisation **absent** from a pool's `shares` has **0 slots** there.
- The slots are apportioned over `C` by the **largest remainder** (Hamilton) method. With C = 64 and
  50 / 30 / 20 %: the quotas are 32, 19.2 and 12.8, the floors give 32 + 19 + 12, and the last slot
  goes to the largest remainder: etabA 32, etabC 19, etabD 13, etabB 0.
- A request within its organisation's slots of the pool keeps priority; beyond them it borrows, at
  borrowing priority, exactly as in [fair-share with borrowing]({{< relref "gateway.md" >}}#fair-share-with-borrowing).

### Who may borrow {#borrow}

- **`borrow = "all"`** (default): idle slots are lent to anyone, behind every shareholder's
  within-share request. An organisation that holds no share is a pure borrower, preempted first.
- **`borrow = "shareholders"`**: only the pool's contributors (a share above 0) may use it. Anyone
  else gets **HTTP 403** `permission_error` before anything is held or spent, and a model of that
  pool is skipped as a fallback target for them. It is 403 and not 429 because retrying cannot help:
  the organisation is not entitled to the machine, whatever its load.

- **`borrow = "spare"`** (since v0.26.0): owners are unchanged, but an organisation that holds no
  share of the pool is served **only while the pool has room**. See
  [Lending to a third party](#ilaas).

In the example, etabB paid nothing for the machine and is refused with `shareholders`; with `all` it
could use idle slots. etabD has a share of 0 on the B300 servers (it is served there only as a
borrower) and 20 % of the RTX machine.

## The configuration {#example}

Two machines in one gateway: the B300 servers (apportioned by the organisations' own shares) and
the RTX machine. This is `examples/two-machines.hcl`; it passes `llmflow-gateway check-config`
v0.25.2 (with the three key files present).

```hcl
master_key_file     = "/etc/llmflow/master.key"
cache_salt_key_file = "/etc/llmflow/cache-salt.key"

org "etabA" {
  share = 0.5
  team "lab-nlp" {}
}
org "etabB" {
  share = 0.25
  team "lab-chem" {}
}
org "etabC" {
  share = 0.25
  team "lab-vision" {}
}
org "etabD" {
  share = 0 # holds no share of the B300: it only borrows there
  team "lab-bio" {}
}

pool "b300" {
  capacity = 256
  # no shares: the organisations' shares above apply to this pool
}

pool "rtx6000" {
  capacity = 64 # 8 GPUs, 8 concurrent requests each
  # What each institution paid for the machine; an organisation not listed
  # (etabB) has a reservation of 0 here.
  shares = { etabA = 0.5, etabC = 0.3, etabD = 0.2 }
  # "all" (the default): idle slots are lent to anyone, at borrowing priority.
  # "shareholders": only the contributors may use the machine.
  borrow = "shareholders"
}

model "llama-3.3-70b" {
  upstream_model = "meta-llama/Llama-3.3-70B-Instruct"
  pool           = "b300"
  key_file       = "/etc/llmflow/vllm.key"
  upstream "gpu01" { base_url = "http://10.0.0.11:8000" }
}

model "qwen-32b" {
  upstream_model = "Qwen/Qwen3-32B"
  pool           = "rtx6000"
  key_file       = "/etc/llmflow/vllm.key"
  upstream "rtx01" { base_url = "http://10.0.0.21:8000" }
}
```

For the real machine, give the model several `upstream` blocks (one per vLLM instance of 1 or 2
GPUs) and set `capacity` to the sum of the measured C of the instances.

## Reports {#reports}

Everything that read "the organisation's share" now reads the pool's: the integer reservations and
the order of borrowers, the owner's reservations under `ha`, the lending report and the decision
report (**a row per partner and pool**, the partner's own line being the totals), fluids, chargeback
and the profiles' oversubscription capacity. See [Decision reports]({{< relref "decision-reports.md" >}}).
`GET /admin/lending` has a `pools` array with a list and a matrix per pool, each guarantee measured
against the pool's own apportionment; `orgs` stays the total per organisation.

## Re-billing the electricity {#fluids}

[Fluids]({{< relref "gateway.md" >}}) re-bill electricity and cooling on **measured** energy, read from
an NVIDIA DCGM exporter. Two cautions for this machine:

- **Check the counter on the machine before relying on it.** The gateway reads the DCGM field
  `DCGM_FI_DEV_TOTAL_ENERGY_CONSUMPTION` (field 156). The
  [DCGM user guide](https://docs.nvidia.com/datacenter/dcgm/latest/user-guide/) (getting-started and
  feature-overview pages) says that non-datacenter GPUs have "limited functionality": field value
  watches are supported on all lines, but field 156 is not guaranteed by the documentation for this
  card. Run `dcgmi dmon -e 155,156` on the machine first.
- **What the gateway does with a scrape that fails:** it loses nothing, since the next delta spans
  the gap (README, fluids). The README does not describe an exporter that answers but never exports
  the field, so verify that it does before pricing energy.
- **One DCGM exporter serves one pool.** A node belongs to one pool: an exporter serving models of
  two pools, one of which has its own shares, is refused by `check-config`. The idle energy of an
  interval is allocated by the shares of the pool the node belongs to.

## Lending to a third party (ILaaS) and billing its fluids {#ilaas}

Since **gateway v0.26.0**, a multi-institution pool (the B300 servers, say) can lend its capacity to a
third party that holds no share of it, **only while the owners' total use is below 100 %**, and bill
that party's electricity and cooling. The example is `examples/b300-ilaas.hcl`.

```hcl
org "tiers" {
  share = 0 # the third party
  team "tiers-prod" {}
}

pool "b300" {
  capacity = 256
  borrow   = "spare" # owners as with "all"; a non-shareholder only while there is room
}

fluids {
  pue           = 1.3
  price_per_kwh = 0.18
  currency      = "EUR"
  idle_energy   = "by_usage" # fixed costs shared by all clients, the third party included
  fixed_cost "racks-b300" {
    monthly = 1800
    pool    = "b300" # omit for the whole platform
  }
}
```

### Admission {#ilaas-admission}

A non-shareholder of the pool (its share of **this** pool is 0; for a pool without `shares`, the
organisation's own share) is admitted only if fewer than C requests are in flight, decided and taken
atomically, and vLLM does not report its queues as saturated. Otherwise it gets **HTTP 429**
`rate_limit_exceeded` with `Retry-After: 5` (a fixed five seconds), a message naming the pool and
saying that the organisation holds no share and is served from spare capacity only, and nothing about
the owners' use. It is 429, not 403, because retrying later can succeed. The same applies when a
request falls back into such a pool.

Admitted, it always takes the **last priority band**: shareholders that borrow stop one band earlier,
so a zero-share organisation never ties with the most over-served owner (before v0.26.0 the ranks were
clamped at 9 for everyone, and with ten or more distinct usage levels the two tied).

**What this does not promise.** On vLLM's main branch (`scheduler.py` and `request_queue.py`, re-opened
2026-11-04), when a running request cannot get KV blocks the scheduler preempts the running request with
the largest `(priority, arrival_time)`, the waiting queue is a heap on `(priority, arrival_time)`, and a
waiting request does not preempt a running one. An owner arriving on a full pool therefore waits for a
slot or for a KV preemption of the third party: the guarantee is that the third party never takes a slot
while the pool is full, and always yields first.

### Billing its fluids {#ilaas-billing}

No extra configuration is needed: active energy is attributed by weighted tokens to whoever worked, so
the third party pays for its own requests. `GET /admin/fluids?group_by=org` gives its row; its
`org_admin` sees only its own. `GET /admin/lending` gives the GPU-hours lent and borrowed (owners to
third party).

**Fixed costs are idle energy, cooling through the PUE (which multiplies idle and active kWh alike) and
the racks.** We recommend `idle_energy = "by_usage"`: the fixed costs are spread over **all** clients,
the third party included, pro rata to each organisation's active energy in the month and pool. With the
default `by_share` the third party, holding no share, pays none of the idle energy and the owners bear it
all. `by_usage` falls back to the shares for a month and pool in which nobody worked (before v0.26.0
that idle energy was `(unallocated)` and went unbilled), and to `(unallocated)` only with neither usage
nor shares.

With `idle_watts > 0`, the idle baseline of a GPU during the third party's work is booked as idle and
shared like any idle energy; with `idle_watts = 0`, the third party pays all the energy of the intervals
it used.

The **racks** (space, power distribution, network) are money, not energy:
`fixed_cost "racks-b300" { monthly = 1800  pool = "b300" }`, with an optional `period "2026-12" { monthly = 1950 }`
override. Each month's amount is split between the organisations on the same basis as the idle energy,
then within an organisation pro rata to active energy, in integer minor units, so every grouping adds up
to the configured amount to the cent. A report over part of a month gets the part covered **by time**
(UTC), and the parts of a month add up exactly. The report has two new columns, `fixed_cost` and
`total_cost` (`fluids_cost + fixed_cost`, rounded to `currency_decimals`, half to even, once per row);
the existing columns are unchanged. The chargeback (`GET /admin/chargeback`, and the decision report's
chargeback table) carries a `fixed_cost` column after `fluids_cost` (in the CSV the columns after it move
by one), and a month's chargeback total is the fluids report's `total_cost` for that month.

### Being a node of a peer dispatcher {#ilaas-federation}

Since **gateway v0.37.0**. ILaaS routes its users with
[aristote-dispatcher](https://github.com/CentraleSupelec/aristote-dispatcher) (MIT, read at v1.5.0). It
lists each backend as `{url, token, organization, max_parallel_requests}`, forwards the client's path
under `url` with the token as a bearer, once, and, with its `least_busy` strategy, **polls `<url>/metrics`
without credentials**, scoring a node by the 95th percentile of `vllm:time_to_first_token_seconds` over a
24 s window. A node with no such histogram scores -1 ("never used") and is chosen first. Pointed at the
gateway as it was, the dispatcher sent it everything, and the 429 of a full pool reached the end user; the
gateway's own `/metrics` must not be opened, it carries per-organisation activity.

```hcl
federation "ilaas" {
  prefix           = "/fed/ilaas"
  org              = "tiers"
  metrics_networks = ["10.0.0.0/8"]
}
```

In the dispatcher the node is `url = https://llm.example.org/fed/ilaas` and a key of the organisation
`tiers`.

- Under the prefix the inference API answers as at the root, **for the keys of `org` only** (403 before
  any budget is held). The administration, the UI, SSO, SCIM, MCP, `/healthz` and the gateway's
  `/metrics` are not routes there; a path that is not in clean form is 404.
- `GET <prefix>/metrics` asks for no key and answers the listed networks only. It carries one histogram,
  `vllm:time_to_first_token_seconds` (`_bucket`, `_sum`, `_count`, label `model_name`) with vLLM's own
  bounds (`TIME_TO_FIRST_TOKEN_BUCKETS` in `vllm/v1/metrics/buckets.py`), and nothing of the owners.
- It observes, for the requests that came through the prefix, the time to the first body byte (the
  first event of a stream; the whole answer when not streamed). **Every refusal for lack of room** (429
  `pool_spare` or a rate limit, 502, 503, 504) is counted in the `+Inf` bucket only: once more than 5 %
  of a window was refused the dispatcher's p95 is +Inf and it sends the traffic elsewhere. After a
  window without traffic its diff is zero in every bucket and the node scores the first bound (0.001,
  not -1), so it is tried again. Before any request the exposition has no bucket line.
- **One model** (since v0.37.1). The dispatcher merges the lines of all models bucket by bucket, the last
  model by name winning, and a mount answers for every model of the gateway (a fallback included), so with
  several the node's score would be that of one of them. `check-config` therefore **refuses a `federation`
  block on a gateway that serves more than one `model`**, naming them. Run one gateway per model for a peer
  that needs several.
- `check-config` refuses a malformed, reserved or overlapping prefix, an unknown `org`, an `org` that
  holds a share of a pool a model uses, and an empty or `/0` `metrics_networks`. `borrow = "spare"` on the
  lent pool is **recommended, not required** (since v0.38.0): with `borrow = "all"` the peer's requests
  queue behind every owner at the lowest priority and the time to the first byte grows with the queue,
  a signal a least-busy strategy can read, only later than a +Inf refusal. A site may prefer queueing
  to refusing; what the share check enforces is that the peer holds no share, so it never gets reserved
  capacity.

The example is the end of `examples/b300-ilaas.hcl`.

## What a real bill holds {#bill}

Since **gateway v0.28.0** the fluids report carries more than electricity. The example is
`examples/b300-ilaas.hcl`.

```hcl
fluids {
  currency           = "EUR"
  idle_energy        = "by_usage"
  price_per_gb       = 0.02  # network
  price_per_gb_month = 0.05  # storage
  fixed_cost "racks" {
    monthly = 1800
  }
  fixed_cost "b300-depreciation" {
    monthly = 4200 # purchase price / months of life: straight line
    pool    = "b300"
    charge  = "non_shareholders"
  }
  fixed_cost "storage-array" {
    monthly = 900
    basis   = "storage_gb_month"
  }
}
org "tiers" {
  share = 0
  billing {
    markup = 0.15
  }
}
```

- **Fixed cost basis.** `energy` (the default, the idle-energy rule), `shares`, `network_bytes` (a switch
  by traffic) or `storage_gb_month` (a storage array by storage used). Use `fixed_cost` for maintenance,
  licences, storage and staff as well. Shared model weights are platform storage: a fixed cost, not
  metered per organisation.
- **Hardware the owners already paid for.** `charge = "non_shareholders"` bills the (straight-line)
  depreciation only to organisations with no share of the pool, on the same basis, and credits what they
  pay to the owners pro rata to the GPU-hours each lent them (the lending matrix). With no
  non-shareholder in use nothing is billed.
- **Network, metered.** Bytes of the request bodies forwarded and of the answers written to clients,
  recorded on every request (an aborted stream counts what it delivered; headers and TLS framing are not
  counted), per organisation and down to team, project and user. Column `network_gb`, priced by
  `price_per_gb`.
- **Storage, metered.** What the gateway keeps per organisation: its usage records in PostgreSQL and the
  request-log objects it writes to the export, sampled by day (the bytes stored at the end of each UTC
  day) and summed into GB-months. Column `storage_gb_month`, priced by `price_per_gb_month`.
- **Markup.** `billing { markup = 0.15 }` adds 15 % to the organisation's costs, in its own column.
- **Columns.** `fixed_cost`, `network_cost`, `storage_cost`, `markup`, `credit` and
  `total_cost = fluids + fixed + network + storage + markup - credit`, each rounded to the minor unit
  once per row. The chargeback CSV has the new columns and its later columns moved: read it by name.

### Closing a month {#close}

Reports are computed live, so a corrected PUE or a late record would rewrite a month that was already
invoiced. `POST /admin/billing/close?month=YYYY-MM` (proxy admin only, a month that has ended) freezes it:
an immutable snapshot with the parameters used (PUE, prices, fixed costs, markups, method and gateway
versions) whose SHA-256 is in the hash-chained audit log. A report over the whole month reads the
snapshot; a difference found later is booked once as an **adjustment** line on the next open month (a credit
note when negative), never a rewrite, and there is no reopening. `GET /admin/billing/closed` lists the
closed months.

### The method of a report {#method}

Every report states how it was computed: a `method` object in the JSON (method version, gateway version,
parameters, closed months read), the headers `X-Llmflow-Method-Version`, `X-Llmflow-Gateway-Version` and
`X-Llmflow-Method` on every response (CSV included), and a Method section and sheet in the decision report.

## The energy beyond the GPUs, a tariff, preemptions {#node-energy}

Since **gateway v0.31.0**:

- **Node energy.** The DCGM counter is the GPUs' energy only. On an upstream, `redfish_url` names the
  node's BMC (https; `redfish_user` and `redfish_password_file`, a secret reference; `redfish_ca_file` for
  the certificate's authority). The gateway reads the chassis' `EnvironmentMetrics` **`EnergykWh`** counter
  (DMTF EnvironmentMetrics v1.3.1), at the same instant as the GPUs: the node's energy minus the GPUs' is
  the **overhead** of the interval, split like the GPUs' energy (the active part by the same token weights,
  the idle part as idle). `node_overhead_factor` (1 to 5) is the fallback when no BMC can be read. Since
  **v0.33.0**, a BMC that reports only a power is metered by the **trapezoid** of its samples (exact for a
  constant power, an estimate between samples; two samples further apart than three rounds are not joined),
  and a counter that goes down (a BMC reboot) is never billed: those intervals, and a missed read, fall to the
  factor. The fluids report states, per month and pool, how much overhead came from the counter, the
  integrated power or the factor. The Redfish client is
  [go-redfish/redfish](https://github.com/go-redfish/redfish).
- **Sub-meter variance.** `meter "2026-10" { kwh = 41250.5 }` records the racks' sub-meter for a month; the
  fluids report and the decision report show it against the measured IT energy as a difference. Nothing is
  rescaled.
- **Time-of-use tariff.** `timezone = "Europe/Paris"` and `tariff` bands (`price_per_kwh`, `days`, `from`,
  `to`) price each hour at the band its local start falls in, daylight saving included (23 and 25 local
  hours on the two days; the repeated 02:00 hour is priced twice). The idle energy is priced at the bands it
  was used in. A closed month keeps its tariff.
- **Preemptions, measured.** The vLLM counter `vllm:num_preemptions` is read per interval; the fluids
  report gives, per month and pool, the energy of the intervals with preemptions and its share of the
  total, an upper bound of what recompute can account for. Nothing is billed on it.

## Upgrading under `ha` {#ha}

Under `ha` the pool's owner decides the apportionment. A replica from before v0.24.0 that owns a
pool with its own shares would silently use the organisations' shares. So the owner's published
reservations now carry a fingerprint of the shares they were computed from, and a replica whose
pool has shares **refuses to follow** an owner that publishes none, or another one: requests to that
pool answer **503** naming the cause until every replica runs the same version and configuration,
and the replica logs an error. **Upgrade all the replicas before adding `shares` to a pool.**
