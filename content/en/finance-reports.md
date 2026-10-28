---
title: "Finance reports"
weight: 29
---

# Finance reports {#finance-reports}

What a finance director, a lab manager or an organisation administrator gets from the
[gateway]({{< relref "gateway.md" >}}) since **v0.13.0**: who used how much, in which
unit, compared with the previous period, as a page, as a file, and next to what Slurm
says about the batch jobs on the same GPUs. This page is written from the repository's
README at tag v0.17.0 (the feature is from v0.13.0); nothing here was measured on a production cluster, and the charts
were checked in a browser under the page's security policy, not by users.

## Who is charged: organisation, lab, team, project, user {#dimensions}

The report groups by any of `org`, `lab`, `team`, `project`, `user`, `key`, `model`,
`pool`, `tag` and `day`. Two levels are new in v0.13.0 and one is now filled by clients:

- **`lab`** is optional, set on a team (`team "physics-1" { lab = "physics" }`). It is an
  accounting level only; access control does not use it. It is stamped on each record
  when the request is made, so moving a team to another lab later does not rewrite the
  past.
- **`user`** is the key's owner, or a JWT's username, or a stable synthetic id. Never a
  token.
- **`project`** is named by the client in a header, `X-Llmflow-Project`, because coding
  agents do not say which project a request belongs to. See
  [Project attribution]({{< relref "gateway.md" >}}#attribution) and, for the client
  side, [Coding agents]({{< relref "agents.md" >}}#claude-code-project). A project the
  key may not use gets a generic 403.

The drill-down follows the same order: organisation, lab, team, project, user.

```hcl
attribution {
  project_header  = "X-Llmflow-Project"
  default_project = "general"
}

org "etabA" {
  share = 0.5
  team "physics-1" { lab = "physics" }
  team "chem-1" { lab = "chemistry" }
}
```

This block passes `llmflow-gateway check-config` v0.17.0 (with the key files present,
and a pool and a model around it). Existing configurations stay valid: a lab is an
attribute of a team, not a new nesting.

## The page: `/ui/reports` {#page}

The page is part of the [admin web UI]({{< relref "gateway.md" >}}#admin-ui): it needs
an `sso` block and a signed-in administrator, it holds no privilege of its own, and an
organisation administrator sees their organisation only. It shows:

- **A period**, with date pickers and shortcuts (last 7 days, last 30 days, this month,
  last month, this year). Dates are UTC and the end is excluded. At most 366 days.
- **A grouping** over any dimension above, including `day`.
- **A drill-down.** In an `org` grouping each row links to its labs, then teams,
  projects, users and models. The filters in force are listed and each can be removed.
- **The previous period** of the same length, side by side: the amount and the change.
- **Charts**, as inline SVG: the share per group (a pie of the seven largest, the rest
  as "other"), this period against the previous one (bars) and the amount per day.
- **Units.** Every amount carries its unit, in one section per unit: **GPU-hours** for
  the local pools, the currency of an external provider (for example `USD` for the
  [Anthropic upstream]({{< relref "gateway.md" >}}#anthropic)) on its own. The two are
  **never added together**: there is no grand total across units.

A report that takes longer than 60 seconds in the store is stopped.

The charts add no script and no style attribute, so the page's
`Content-Security-Policy` is unchanged. **Since gateway v0.15.1 every chart** (pies, bars
and the daily series) **is drawn by [go-plot](https://github.com/go-plot/plot) v0.3.0**,
with `ParseMath(false)` on the figure and `PlainText()` on every call that draws a name
(tick labels, axis labels, titles). A team called `$\input{/etc/passwd}$`, `$5 and $6` or
`<script>` therefore stays a string and never reaches go-plot's TeX typesetter; a test
that spies on go-plot's text renderer proves it, with a control that does reach TeX. The
pie receives no name at all: its legend is the gateway's own escaped text. Before v0.15.1
(v0.13.0 to v0.15.0) the pie was drawn by go-plot v0.2.0 and the bars and the series by a
renderer of the gateway's own, since go-plot's tick labels had no plain-text option yet;
that engine is removed.

## Exports {#exports}

From the links on the page, or from the API:

```text
GET /admin/export?from=2026-10-01&to=2026-11-01&format=csv|xlsx|pdf[&group_by=org,lab,team,project,user]
```

| Format | What the file holds |
| --- | --- |
| `csv` | the spend report (default grouping `org,lab,team,project,user,model,pool`), with `cost` and `unit` columns |
| `xlsx` | a **Summary** sheet (the period, totals per unit against the previous period), a **Reconciliation** sheet (inference against Slurm batch GPU-hours per organisation), then one sheet per grouping: `org`, `lab`, `team`, `project`, `user`, `model`, `pool`, `tag`, `day`. Numbers are typed, each cost column names its unit (`Cost (GPU-hours)`, `Cost (USD)`) |
| `pdf` | a management summary: the period, totals per unit, totals per level (org, lab, team, project; the ten largest) with pie charts, the daily trend, and the reconciliation table |

An export shows what the report shows the same person. Each download is written to the
[audit log]({{< relref "audit.md" >}}) with kind `export` (who, their roles, the query,
the format, the size, the status), **refusals included**. Bounds: at most 366 days, at
most 100,000 rows at the finest grouping (HTTP 413 beyond), and the 60-second deadline.

What an export cannot carry:

- **CSV.** A cell that begins with `=`, `+`, `-`, `@`, a tab or a carriage return is
  prefixed with `'`, so a spreadsheet does not read a label as a formula.
- **XLSX.** The writer has no formula at all; such strings are marked as text.
- **PDF.** Every label is drawn as glyphs, in one embedded font. The file carries no
  action, link, script or attachment.
- Labels lose their control and bidi-format characters and are clipped; the file name is
  built from the dates only. The writers are fuzzed (`FuzzExportWriters`).

From the command line, `llmflow-gateway export` writes the same report as CSV or
parsable2 (see the [gateway page]({{< relref "gateway.md" >}}#status)). **Since v0.13.1
it has the API's bounds**: at most 366 days, 100,000 rows and 60 s in the store. An
operator raises them explicitly with `--max-days`, `--max-rows` and `--timeout`. Every run
is written to the audit log (kind `export`, actor `cli:<login>`) **before the store is
read**; a run with a raised bound does not start when that entry cannot be written, and
a run within the defaults prints a warning instead. (Take v0.14.1 or later, not v0.13.1:
see [which tag to take]({{< relref "gateway.md" >}}#status).)

## Reconciling with Slurm {#slurm}

The gateway knows the GPU-hours of the **inference** it served. Slurm knows the GPU-hours
of the **batch jobs** on the same cluster. Next to each other, per organisation, they say
how the shared GPUs were really used: on the Reports page and in the XLSX and PDF
exports. This is optional.

The input is the output of `sreport`:

```sh
sreport -P cluster AccountUtilizationByUser -T gres/gpu -t Hours start=2026-10-01 end=2026-11-01
```

Either a file the gateway reads whenever it builds a report, or a file uploaded with an
export (`POST /admin/export?...&format=xlsx|pdf`, the report as a `text/plain` body, or
the form on the Reports page):

```hcl
slurm {
  sreport_file = "/var/lib/llmflow/sreport.txt"   # optional; absolute path
  accounts     = { "phys-a" = "etabA" }            # optional; Slurm account -> organisation
}
```

This block passes `llmflow-gateway check-config` v0.17.0 (with the file present).

How the file is read:

- Delimited layouts (`-P`, `-p`, or a comma-separated file with the same header) and
  sreport's default fixed-width table. Prefer `-P`: the fixed width truncates long names
  with `+`.
- Only **account totals** count (rows without a login), for the organisation `accounts`
  names or, failing that, the organisation of the same name.
- The unit comes from the "Usage reported in ..." line (minutes by default, hours or
  seconds). A report in percent is refused; rows of another TRES are skipped.
- The **period sreport printed is shown** next to the table, so a mismatch with the
  report's period is visible. Look at it before trusting a comparison.
- The file is untrusted: at most 1 MiB, 50,000 lines of 4 KiB, 32 columns, finite
  non-negative numbers, UTF-8. Every refusal is an error: 400 for an upload; for a
  configured file that cannot be read, a generic 500 and a log line (the path is never
  shown). The parser is fuzzed (`FuzzSreport`).
- An organisation's batch usage is shown to whoever may view that organisation
  (organisation viewers and administrators, proxy roles). Team-level roles see none.

## What this does not do {#known-issues}

Known at v0.17.0:

- **The binary grew, then shrank.** In v0.13.0 to v0.15.1 go-pdfkit's root package pulled
  in a widget toolkit the export never used. Fixed in v0.15.2 by go-pdfkit v0.20.0: the
  binary went from 42,110,114 to 39,790,834 bytes (5.5 % smaller; see open defect 18 in the
  [Limitations]({{< relref "gateway.md" >}}#limitations)). The PDF writer is
  [go-pdfkit](https://go-pdfkit.github.io/docs/).
- **Not measured.** The cost of building a report on a large database, and how long an
  export takes at 100,000 rows, are not measured. The only bounds stated are the
  366 days, the 100,000 rows and the 60 seconds.
- **No price in a report unless a model has one.** A GPU-hour is the gateway's unit of
  account for local models, not a euro; converting it to money is the calibration of the
  [guide]({{< relref "guides/fairshare/_index.md" >}}), which is yours to decide.
