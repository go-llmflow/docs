---
title: "Chaos"
weight: 32
---

# Chaos: faults in, invariants out {#chaos}

`llmchaos` (in [`bench`](https://github.com/go-llmflow/bench), `cmd/llmchaos`) attacks a **llmflow-gateway
binary** from the outside. It writes the configuration of a multi-institution deployment, starts the gateway
as a process, injects faults while clients send traffic, and judges nine invariants after every scenario. A
scenario is a row of a Go table and is replayed from its name and a seed.

```sh
go install github.com/go-llmflow/bench/cmd/llmchaos@latest
llmchaos list
llmchaos run baseline upstream-cut-sse --seed 1 --seed 2 \
  --gateway ./llmflow-gateway --workdir ~/chaos-runs
```

## The deployment {#chaos-deployment}

Three owner organisations with shares 50/30/20 of one pool of 8 slots; an ILaaS third party (`tiers`, share 0,
`borrow = "spare"`, a 15 % markup) reached through a `federation` mount; two identity providers (one with a legacy
`domain`, one with `domains`) and a federation issuer accepting a list of `schac_home_organization`; hard and soft
budgets on a key, a team, two organisations and a person; fluids with idle energy, fixed costs, a
`charge = "non_shareholders"` credit and a time-of-use tariff; a fake DCGM exporter and a BMC over TLS (go-redfish
`mockbmc` for the Redfish service, plus a controllable `EnvironmentMetrics` counter); two HA replicas on one
PostgreSQL in CI. The upstream is the recording fake vLLM of `bench` behind a fault injector.

## The invariants {#chaos-invariants}

| # | invariant | how it is judged |
|---|---|---|
| 1 | no hard budget exceeded beyond the documented worst case, in every scope | the clients' own sum of answers read in full against the limit, plus the gateway's `committed` and `held`; the excess allowed is what answers cost above their W |
| 2 | no request billed to another person or org | per key and per org, billed requests and cost must lie between what the caller received in full and what it could have been charged; a failed request bills nothing |
| 3 | the spare borrower never goes over the owners; the owners' share converges | borrower concurrency at the upstream never above the pool; `X-Llmflow-Priority` of the borrower behind every owner's; after the owners ramp up (3 s grace), the borrower gets at most 10 % of the answers and each owner's share of completed work is within 15 points of 50/30/20. Judged only in a scenario whose upstream models vLLM's priority scheduling (a waiting queue ordered by priority then arrival, a waiting request never preempting a running one); elsewhere the shares are not judged and the verdict says so |
| 4 | fluids conservation | active + idle = kWh per row; rows = GPU energy + stated overhead; fixed costs minus credits = the racks, credits = the depreciation when a non-shareholder used energy |
| 5 | a closed month never changes | PostgreSQL scenarios only (CI): the month that has just ended is closed during traffic; its fluids and spend reports must be byte-identical afterwards and a second close must answer 409. The closed month holds no traffic of the campaign (the gateway's clock cannot be moved from outside), so this judges immutability of the snapshot, not the figures in it. Everywhere else: N/A |
| 6 | refusals | 429 has a positive `Retry-After`; no 200 with an empty body; a token the gateway must refuse (clock skew beyond `clock_skew`, key removed from the JWKS) gets 401 |
| 7 | no leak | the harness's goroutines and descriptors, and the gateway's descriptors, return to their start |
| 8 | HA spend | spend neither lost nor doubled across replica kills, partitions and reloads |
| 9 | federation metrics | no other organisation or key in the exposition; the `+Inf` bucket equals the count of answered and refused requests; refusals are in `+Inf` only |

## Faults {#chaos-faults}

Upstream: 5xx, hang, slow tokens, a stream cut mid-way, wrong or missing usage, `/v1/models` down at start,
preemption bursts. Clients: disconnects at every stream phase, retry storms, `n` and `best_of`, bursts at the edge of
a budget. Identity: JWKS down, rotated, clocks four minutes ahead or seven behind. BMC: offline, counter reset,
power only. HA: a killed replica, a partition between replicas, a rolling reload. PostgreSQL: a restart and slow
queries, through a relay (there is no Docker on the workstation). Faults combine in one scenario.

## Reading a failure {#chaos-failures}

A run leaves `<workdir>/<scenario>/evidence`: the clients' ledger, the upstream's witness of every hit, the
spend, budget and fluids reports at the end, and the gateway log. A broken invariant is reduced to a deterministic
test in the gateway before anything is claimed: first say whether the failure value is wrong or the check is.
Several early failures of the campaign were the check's: a Retry-After capped at an hour when a budget refusal
rightly names the time to the period's end; a power-only BMC the fake made inconsistent; a client that cancelled
while the answer arrived; a fake upstream that ignored vLLM's priority and so served the owners equally; and a
token "signed with a removed key" answered 200, which was the verifier's documented refetch bound
(`min_refresh`, default one minute: until the key set is refetched, a removed key still verifies what it signed, and
tokens of the new key are refused). With `min_refresh = "5s"` a deterministic test, at the library level and against
the gateway binary, shows the removed key refused after the refetch.

## The campaign in CI {#chaos-ci}

The repository's CI runs all 31 scenarios with seeds 1 to 5 against **gateway v0.37.1 and v0.38.0**, as six shards per
release on GitHub runners (each with its own PostgreSQL 17), then merges the shards and fails on a failed invariant, a
run that broke, was skipped or sent nothing, an N/A a scenario is not entitled to, and any scenario and seed that no
shard ran. Result of the run of 2026-11-07 (155 runs per release, both releases identical):

| invariant | PASS | FAIL | N/A |
|---|---|---|---|
| 1 budgets | 145 | 0 | 10 |
| 2 attribution | 155 | 0 | 0 |
| 3 spare borrower and shares | 155 | 0 | 0 |
| 4 fluids | 120 | 0 | 35 |
| 5 closed month | 10 | 0 | 145 |
| 6 refusals | 155 | 0 | 0 |
| 7 leaks | 155 | 0 | 0 |
| 8 HA spend | 30 | 0 | 125 |
| 9 federation metrics | 110 | 0 | 45 |

Without a control, "no difference between two releases" would be a statement about what the scenarios reach, not about
the releases. So a scenario can be a **positive control**: it must fail on the releases before a fix and pass from it on,
and `llmchaos diff` between the two releases must list exactly the controls.

| control (scenario) | v0.38.0 fix it detects | v0.37.1 | v0.38.0 |
|---|---|---|---|
| `idp-schac-spoof` | a token's institution is its name's scope: a federation token named `eve@univ-e.example` that states the accepted `schac_home_organization` is refused | accepted (200, or 429 when the budget was spent) | refused, 401 |

`diff` of the run of 2026-11-08 lists 10 differences, all of this scenario (invariant 6, the verdict, and invariant 9,
where the accepted request is one the dispatcher counted). A control token scoped to its institution is accepted on both
releases, so the probe cannot pass by refusing everything. **The other four fixes still have no control**: an
organisation administrator creating a bare or foreign-domain user, the spare check before the guards (a refused spare
request must cost no guard call), the saturation fallback's spare check, and `n` and `best_of` in the borrower's slots.
The campaign cannot see them, and a release that regressed on one would not fail it.

Before the control, the first campaign had already shown the schac fix indirectly: v0.38.0 refuses a federation token
whose name is unscoped, which v0.37.1 accepted, and it appeared as 15 differing verdicts until the harness minted a
scoped name (`carol@univ-c.example`) as the release notes require.

Not a gateway verdict, the first runs also found defects of the harness, all fixed and tested: a fault lifted twice (a
replica started a second time, the first process left running), the request count of a client that left before the
upstream saw it (allowed only for a request the client abandoned and the fake upstream never received), a port taken
between its choice and the gateway's bind, and a run started across a UTC midnight (the per-day budgets rolled over
inside it, and the clients legitimately paid more than the limit).

## A gateway finding: records wait in memory while the store hangs {#chaos-recorder}

One run of `ha-kill-during-pg-restart` (seed 2, gateway v0.38.0) left four answered requests unbilled. It did not recur,
but a test shows it can happen: the accounting recorder is a single goroutine, and while the store hangs (PostgreSQL
restarting) it is blocked inside an insert with a 10 s timeout, so records of requests answered meanwhile are in memory,
not in the spool, and a SIGKILL loses them. The documented bound is about one second of traffic; here it is the length of
the stall. Reproducer and details: [go-llmflow/gateway#165](https://github.com/go-llmflow/gateway/issues/165). Not fixed
in this release.

## Behaviours the campaign confirmed {#chaos-observed}

These are documented gateway behaviours that the invariants account for, and the verdicts name them when they occur.

- **A replica that cannot reach the pool owner admits at the lowest priority band** (`ha/node.go`, `lowest()`), the band
  the spare borrower always takes. An owner's request can therefore tie with the borrower's while ownership changes
  (killed replica, reload, slow PostgreSQL). Invariant 3 counts these requests and does not judge them.
- **A killed replica loses at most the last second of its accounting queue** (gateway README, durable accounting). The
  answers a replica served in the 2 seconds before a `SIGKILL` may go unbilled; invariants 2 and 8 allow exactly those,
  and nothing else.
- **A stream cut after its `200` ends as a clean EOF**, with no error event and no `[DONE]`; the client keeps what it
  received. Reported as an observation under invariant 6.
- **A request that failed upstream is counted in `requests` with zero tokens and zero cost.**
- **An upstream's own 503 is passed through without `Retry-After`.**

## Key rotation: an exposure to know {#chaos-keys}

The gateway refetches an issuer's key set only when a token its held keys cannot verify arrives, and at most once per
`min_refresh` (default **1 minute**, minimum **5 seconds**; `jwt` and `sso` blocks). A key **removed** from the
issuer's JWKS is therefore still accepted for the tokens it signed until the next refetch, and tokens of the new key are
refused (401) in the meantime. `llmchaos` showed it (scenario `idp-jwks-rotated`) and two deterministic tests confirm that
the removed key is refused once the refetch has happened. **An operator handling a compromised signing key should
restart the gateway or lower `min_refresh` (to 5 s) before rotating**, and treat tokens already signed as valid until
they expire or the refetch happens.

## What it does not exercise {#chaos-limits}

What invariant 5 judges is **immutability only**: the figures of a closed month (the gateway's clock cannot be moved from outside, so the closed month is empty, and a month that holds the campaign's traffic cannot be closed),
soft-budget alert delivery, the gateway's goroutines (no pprof is exposed), and a PostgreSQL restart by the server
itself (the relay resets connections). The PostgreSQL and HA scenarios, `migrate-users` and the month close run only in
the repository's CI, against both pinned gateway releases, and fail the job when an invariant breaks, when a
scenario sent nothing, or when one was skipped.
