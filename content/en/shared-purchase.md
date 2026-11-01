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

In the example, etabB paid nothing for the machine and is refused with `shareholders`; with `all` it
could use idle slots. etabD has a share of 0 on the B300 servers (it is served there only as a
borrower) and 20 % of the RTX machine.

## The configuration {#example}

Two machines in one gateway: the B300 servers (apportioned by the organisations' own shares) and
the RTX machine. This is `examples/two-machines.hcl`; it passes `llmflow-gateway check-config`
v0.25.0 (with the three key files present).

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

## Upgrading under `ha` {#ha}

Under `ha` the pool's owner decides the apportionment. A replica from before v0.24.0 that owns a
pool with its own shares would silently use the organisations' shares. So the owner's published
reservations now carry a fingerprint of the shares they were computed from, and a replica whose
pool has shares **refuses to follow** an owner that publishes none, or another one: requests to that
pool answer **503** naming the cause until every replica runs the same version and configuration,
and the replica logs an error. **Upgrade all the replicas before adding `shares` to a pool.**
