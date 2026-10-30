---
title: "Decision reports"
weight: 31
---

# Decision reports {#decision-reports}

Reports for the people who decide how a shared GPU cluster is paid for and shared:
**fluids** (the electricity and cooling each consumer really used) and **lending** (who used
the guaranteed share, who borrowed, who lent), from [gateway]({{< relref "gateway.md" >}})
**v0.19.0** (issue #97, parts 1 and 2); and, from **v0.20.0** (parts 3 and 4), a **finance
reader role**, **cost centres and a chargeback file**, **adoption** figures and a **monthly
decision report**. This page is written from the README at v0.20.0. None of this was measured
on a production cluster.

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

## The finance reader and cost centres (since v0.20.0) {#finance}

**`finance_viewer`** is a role without scope. It reads the reports of **every organisation**
(`/admin/spend`, `/admin/export`, `/admin/fluids`, `/admin/lending` with its matrix,
`/admin/chargeback`, `/admin/adoption`, `/admin/decision`) and nothing else: no listing of
teams, keys or users, no audit log, no write. **Only a proxy admin grants it**
(`PUT /admin/users/{name}/roles`); an `org_admin` never can, so an organisation's admin cannot
open another organisation's reports. It is also a valid `sso` role mapping. Every report route
has a row in the authorisation-matrix test with a `finance_viewer` column.

**Cost centres** are general-ledger codes on an organisation, a lab or a project; the most
specific wins (project, then lab, then organisation):

```hcl
org "etabA" {
  share       = 0.5
  cost_center = "GL-7001"
  team "t1" { lab = "genomics" }
  lab "genomics" { cost_center = "GL-7002" }
  project "t1" "p9" { cost_center = "GL-7003" }
}
```

A code is 1 to 32 characters of letters, digits, `_ . / -`, starting with a letter or digit
(no separator, quote or space: a finance system reads it). `check-config` v0.20.0 accepts the
block above in a complete file and refuses `"GL 7001"`.

## Chargeback {#chargeback}

`GET /admin/chargeback?from=&to=&format=csv|json` (csv by default): **one row per cost centre
and calendar month** of the period, with `gpu_hours`, `kwh`, `fluids_cost` and
`fluids_currency` from the [fluids](#fluids) block, and an external provider's spend **in its own
currency on a row of its own** (`provider_cost`, `provider_currency`).

- **Money is exact.** Each row's cost is rounded **once, half to even**, to the currency's minor
  unit (`fluids.currency_decimals`, default 2; provider spend: 2) and written as a decimal string
  without floating point; the JSON also carries the integer `*_cost_minor`. A file's total is the
  sum of its rows, in integers.
- **Instants, not day strings.** An interval belongs to the month its start falls in, as in
  `/admin/fluids`, whatever time of day the period ends (an earlier version dropped energy when
  `to` was not midnight). Idle energy under `by_usage` is allocated over the **whole calendar
  month**.
- **It adds up.** What has no code is the cost centre `UNMAPPED`, and energy no organisation is
  charged for is `UNALLOCATED`: the file always adds up to the reports.
- It is a chargeback in FinOps' sense, the file a finance system books (finops.org, Invoicing and
  Chargeback). **Estimate-against-actual variance is not implemented**: it needs actuals entered
  somewhere.
- Organisation admins and viewers get their own codes only.

## Adoption {#adoption}

`GET /admin/adoption?from=&to=&format=json|csv`: active people per day and per week (Monday),
requests, and per model the active people and requests. **Counts only**: no name, key or content
leaves it; a service key's request counts as a request and as no one. **Small cells are
suppressed**: a day, a week or a model used by fewer than *k* distinct people is returned without
counts (`suppressed`), models suppressed together are merged into `(other models)` with their
requests, and the overall figure is hidden below *k*. *k* is `adoption_min_users` (top level,
**default 5, at least 2**; `check-config` v0.20.0 refuses 1: "a figure over one person is that
person").

## The monthly decision report {#decision}

`GET /admin/decision?from=&to=&format=pdf|xlsx|json` gathers the figures above for a period (at
most 366 days) into the document a partnership's steering committee reads:

- **PDF**, typeset by go-pdfkit's `html2pdf` v0.3.0: A4, page numbers, a table of contents, then
  **one page per partner** (guaranteed capacity, use in share, borrowed and lent GPU-hours, net
  balance, inference GPU-hours against the previous period, fluids on measured energy, latency in
  share against borrowing, active people, cost centre), then the lending matrix, fluids, adoption,
  chargeback rows and charts.
- **XLSX**: sheets Partners, Lending matrix, Chargeback and Adoption; typed cells, one column per
  unit.
- **json**: the same data. Proxy roles and `finance_viewer` read every partner and the matrix;
  `org_admin` and org-level viewers their own partner only. **Every download is audited.**

**Scheduled delivery:**

```hcl
decision_report {
  destination = "reports"          # an export block (S3-compatible or a directory)
  formats     = ["pdf", "xlsx"]    # default both
  day         = 2                  # of the month, UTC, 1..28 (default 2)
  mail_to     = ["finance@partner-a.example", "finance@partner-b.example"]  # optional, max 50
}
```

This block, with an `export "reports"` block (`kind = "file"`, a `directory`) and an `alerts {
smtp { ... } }` relay (which `mail_to` requires), passes `check-config` v0.20.0. From that day on,
the previous calendar month is delivered twice over: **to the export** as
`decision/YYYY-MM/decision-YYYY-MM.pdf` and `.xlsx` (write-once under Object Lock where the export
has it), and **by mail** to `mail_to`: a MIME message with the PDF attached, the XLSX too when both
fit in 8 MiB, else a line saying where the export holds them.

The sink is the state: an object that exists is delivered, and the marker
`decision/YYYY-MM/mailed.json` says the mail went out, so a restart or a second replica does
nothing twice, and a report whose upload is still in flight is not typeset again. Every delivery
is audited **when the object is seen in the export**, not when it is queued, and so is every mail
(period, format, size, never the content).

## Bounds of the heavy reports (#111) {#bounds}

The heavy reports (lending, chargeback, decision) read the store under a timeout and **at most 2
run at once: the next caller gets 429 with `Retry-After`**. A period is capped at 366 days and
**500,000 requests (HTTP 413 beyond)**. The decision report is also refused (413) beyond 100
partners, 2,000 matrix cells, 3,000 chargeback rows, 400 adoption rows or 16 MiB; typesetting is
limited to 2 minutes and the build runs under a timeout.

## Mail transport: TLS is now required {#smtp}

**Migration note, read it before upgrading.** Alert and report mail used to go through Go's
`net/smtp.SendMail`, which upgrades to TLS only when the server offers it: a relay that did not
advertise STARTTLS got the message, in clear text. Since v0.20.0:

```hcl
alerts {
  smtp {
    server  = "mail.example.org:587"
    from    = "llmflow@example.org"
    to      = ["ops@example.org"]
    tls     = "starttls"   # default; "implicit" (port 465); "none"
    timeout = "30s"        # dial and whole session
  }
}
```

- `starttls` (the **default**): the session is refused, and nothing is sent, unless the server
  offers STARTTLS and the handshake succeeds (certificate verified against the host name, TLS 1.2
  or later).
- `implicit`: TLS from the first byte.
- `none`: clear text, the old behaviour. Accepted without more only for a relay on this host
  (`127.0.0.1`, `localhost`); for any other host it needs `allow_cleartext = true`. A password is
  never sent over a clear connection to another host.
- `timeout` (dial and whole session) is new; it defaults to 30 s.

**An existing `smtp` block that points at a relay without STARTTLS now fails every send** ("does
not offer STARTTLS"), soft-budget alerts and decision-report mail included. Enable STARTTLS on
the relay, or set `tls = "none"` (plus `allow_cleartext = true` off the local host) to keep the
old behaviour knowingly. v0.20.0 `check-config` was run on these cases: `tls = "none"` with
`mail.example.org` is refused ("it is for a relay on this host"); with `allow_cleartext = true`,
or with a `127.0.0.1` relay, it is accepted. The live behaviour of the session was not run here.

## The PDF costs binary size {#binary-size}

The PDF comes from go-pdfkit's `html2pdf`, which brings go-webengine (layout, a JavaScript engine,
esbuild) into the module graph: the stripped gateway binary grows from 28.8 MB to 43.8 MB
(**+52 %**), 60.2 MB unstripped. The README lists options under issue #97 (a build tag, or a
separate `llmflow-report` command that renders the PDF from `format=json`); none is implemented.

## Upgrading to v0.19.0 {#upgrade}

Start-up adds the tables of the soft quotas (`llmflow_warn_marks`, `llmflow_warn_levels`, and
`warn_at` on `llmflow_budgets_v2`) and those of the fluids and the pool capacity history.
**Upgrade every replica.** Soft quotas, which come in the same release, are on the
[Workload profiles]({{< relref "workload-profiles.md" >}}#soft-quotas) page.
