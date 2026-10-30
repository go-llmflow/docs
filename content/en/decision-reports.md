---
title: "Decision reports"
weight: 31
---

# Decision reports {#decision-reports}

Two reports for the people who decide how a shared GPU cluster is paid for and shared:
**fluids** (the electricity and cooling each consumer really used) and **lending** (who used
the guaranteed share, who borrowed, who lent). Both come with
[gateway]({{< relref "gateway.md" >}}) **v0.19.0** (issue #97, parts 1 and 2). This page is
written from the README at that tag. Parts 3 and 4 of #97 (finance roles, cost centres and
chargeback, a monthly decision report) are **not in this release** and are not described here.
None of this was measured on a production cluster.

## Fluids: measured energy, attributed to consumers {#fluids}

Electricity and cooling are re-billed on **measured** consumption, not on a guess. Name the
NVIDIA DCGM exporter of each node on its upstreams and price the energy:

```hcl
fluids {
  pue           = 1.4        # facility energy / IT energy, 1..5 (default 1)
  price_per_kwh = 0.18
  currency      = "EUR"      # required with a price; never summed with GPU-hours
  idle_energy   = "by_share" # or "by_usage" (default by_share)
  w_prefill     = 1          # relative energy of a prefill and a decode token
  w_decode      = 1          #   (a modelling choice: calibrate on your models)
  idle_watts    = 0          # per GPU; energy up to it is idle (0: none)
  interval      = "15s"      # scrape period
  settle        = "1m"       # wait before attributing, so accounting has landed
  period "2026-07" { price_per_kwh = 0.21 }   # per-month override (pue too)
}
model "llama" {
  upstream "gpu01" {
    base_url = "http://10.0.0.1:8000"
    dcgm_url = "http://10.0.0.1:9400/metrics"
  }
}
```

This configuration, in a complete file, passes `llmflow-gateway check-config` v0.19.0.

- **Source.** `DCGM_FI_DEV_TOTAL_ENERGY_CONSUMPTION` is a counter "since the driver was last
  reloaded (in mJ)" (dcgm-exporter's `default-counters.csv`). The gateway turns it into
  per-interval deltas per GPU. A counter that goes down (a driver reload) counts its new value
  as the energy since the reload; a failed scrape loses nothing, since the next delta spans
  the gap; a GPU seen for the first time only sets a baseline. **Energy used while the gateway
  is down is not counted.** The URL follows the rules of `base_url` (no credentials, no query),
  is read with a timeout and a bounded body, and **follows no redirect**. Several upstreams on
  one node share its `dcgm_url`; **Slurm-discovered members are not metered yet.**
- **Exact attribution.** All arithmetic is in integer millijoules, so what is attributed adds
  up exactly to what was measured (property-tested). In an interval, *active* energy is split
  between the requests that finished in it by `prefill tokens x w_prefill + decode tokens x
  w_decode` (prefill tokens are the prompt minus cached tokens). The relative weights are **a
  modelling choice**: calibrate them on your models.
- **Idle energy** (an interval with no work, or the `idle_watts` floor) is allocated over a
  month by the agreement: `by_share`, pro rata to the organisations' shares, or `by_usage`, to
  their energy. Energy no one can be charged for is `(unallocated)`, visible to the proxy roles
  only. An organisation's idle energy is spread over its rows pro rata to their active energy,
  so every grouping adds up to the measured total.
- **Cost** is `kWh_IT x PUE x price_per_kWh`, priced per month (a `period` block overrides PUE
  and price for one month).
- **Report.** `GET /admin/fluids?from=&to=&group_by=&format=json|csv`, with `group_by` among
  org, lab, team, project, user, model, pool, day. Columns: `gpu_hours`, `kwh_active`,
  `kwh_idle`, `kwh`, `fluids_cost` and `currency`: three units in three columns, never added.
  Proxy roles read every organisation; `org_admin` and org-level `viewer` read their own;
  team-level roles get 403. Periods are capped at 366 days.
- **Batch jobs.** A parser for `sacct -P -o Account,ConsumedEnergyRaw` (joules) exists; Slurm
  notes the value is a job's real energy only for an exclusive allocation. **It is not yet
  wired into the reconciliation sheet.**

## Lending: guaranteed share against use {#lending}

`GET /admin/lending?from=&to=&format=json|csv` settles the first two rules of a shared-GPU
agreement from the period's finished requests:

- **Guaranteed capacity** is reserved slots x hours of the period. The reservation is the one
  the admission test uses, with **C as it was at each instant**: the gateway records every
  change of a pool's C (its configured capacity plus its Slurm-discovered instances) and the
  report integrates that history. A pool with `capacity = 0` fed by Slurm jobs is guaranteed
  what its instances made it. A period before the history started falls back to the configured
  capacity.
- **Used in share** and **borrowed** GPU-hours follow the request's `Borrowing` flag, decided
  at admission.
- **Lent.** Borrowing is served by capacity others reserved and did not use at that moment. At
  every instant a borrower's GPU-hour rate is split between the lenders in proportion to their
  idle reserved slots. `borrowed_from_others` came from other organisations;
  `borrowed_self_served` ran while the borrower's own reservation was idle again; borrowing
  that found no idle reservation anywhere is `unattributed_gpu_hours` (proxy roles), never
  dropped.
- **Net** is lent minus borrowed from others; the nets of all organisations sum to 0. The
  **lending matrix** (lender x borrower, GPU-hours) is returned to the proxy roles only.
- **Admission wait.** Each record carries its **admission wait** (`wait_ms`): the time between
  the request's arrival and the fair-share grant, **in the gateway's own queue**. Waiting there
  is not holding a reservation, so the lending interval starts at the grant. The report gives
  p50, p95 and p99 of the admission wait of in-share against borrowing requests
  (`wait_in_share_ms`, `wait_borrowing_ms`): the evidence that the guarantee held, next to the
  end-to-end latency. **The time a granted request spends in vLLM's own queue is not recorded,**
  and records from before v0.19.0 have `wait_ms = 0`.

A request belongs to the period it finished in; a period is capped at 366 days and 500,000
requests (HTTP 413 beyond). The heavy reports read the store under a timeout and **at most two
run at once**: the next caller gets 429 with `Retry-After`. `org_admin` and org-level `viewer`
read their own row, not the matrix; team roles get 403. The identities above are
property-tested in the repository against the real fair-share pool on random timelines.

## Upgrading to v0.19.0 {#upgrade}

Start-up adds the tables of the soft quotas (`llmflow_warn_marks`, `llmflow_warn_levels`, and
`warn_at` on `llmflow_budgets_v2`) and those of the fluids and the pool capacity history.
**Upgrade every replica.** Soft quotas, which come in the same release, are on the
[Workload profiles]({{< relref "workload-profiles.md" >}}#soft-quotas) page.
