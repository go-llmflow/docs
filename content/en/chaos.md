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
and `llmchaos diff` between two releases must list exactly the controls.

| control (scenario) | fix it detects | v0.37.1 | v0.38.0 | v0.38.1 |
|---|---|---|---|---|
| `idp-schac-spoof` (v0.38.0) | a token's institution is its name's scope: a federation token named `eve@univ-e.example` that states the accepted `schac_home_organization` is refused | accepted | refused, 401 | refused |
| `control-admin-users` (v0.38.0) | `POST /admin/users` is bound to the institutions: the org admin of one institution creating a bare name, or a name in another institution's domain, is refused 400, the gateway restarts, and a name in its own domain is created (control) | created (201) | refused | refused |
| `control-spare-guard` (v0.38.0) | a request refused for lack of spare capacity costs no guard-model call, and a guard call never runs at priority 0: with a guarded spare-only borrower and the owners keeping the pool full, the guard model's calls are counted by a fake guard | about 1 000 to 2 000 guard calls per run, all at priority 0, for 90 to 100 admitted requests | 2 per admitted request, none at priority 0 | same |
| `control-fallback-spare` (v0.38.0) | the saturation fallback takes its slot through the same spare test: a borrower that holds a share of the main pool too small to reserve a slot, a never-full fallback pool whose vLLM queue is saturated | all 28 borrower requests of the run moved into the saturated fallback pool | none moved | same |
| `control-n-slots` (v0.38.0) | `n` and `best_of` are counted in fair share: requests with n = best_of = 4, max_tokens 64, and the gateway's in-flight token gauge per request in flight | about 113 tokens (one choice) | at least 256 (four) | same |
| `control-recorder-stall` (v0.38.1, gateway#165) | the accounting recorder no longer waits on the store: PostgreSQL hangs for 5.5 s (the relay accepts and answers nothing), requests are answered meanwhile, the only replica is SIGKILLed 4.5 s into the hang, before the 10 s insert timeout, and restarted: every answered request must be billed exactly once, but the last second before the kill | about 60 answers of 370 lost | same | none lost |

In the run of 2026-11-08 (185 runs per release, seeds 1 to 5, none broke an invariant outside the controls), `llmchaos
diff` v0.37.1 against v0.38.0 lists 30 differences and v0.38.0 against v0.38.1 lists 5: **exactly the controls of the
fixes between them**, the five audit controls (10 for the schac one, which also moves invariant 9, where the accepted
request is one the dispatcher counted) and the recorder control. A control token scoped to its institution is accepted
on both releases, so the schac probe cannot pass by refusing everything, and each control fails the campaign when it
passes on the old release.

The admin-users control covers the refusals and the restart; it does not cover the real person's SSO or SCIM role after
a squat, because the harness has no SCIM block. The guard and fallback controls run without a federation mount (the
gateway refuses a federation on a gateway that serves several models) and declare invariant 3 not applicable, since
there is a second upstream.

Each control had to be made to discriminate: a first version of the fallback control passed on v0.37.1 too, because a
spare-only borrower is never admitted into a saturated main pool on either release, so the move to a fallback is reached
only by a borrower that holds a share of the main pool; the recorder control first lost the documented last second on
both releases, and kills 4.5 s into the hang so that the stall dominates it. The scenario was corrected, not the judge.

Before the control, the first campaign had already shown the schac fix indirectly: v0.38.0 refuses a federation token
whose name is unscoped, which v0.37.1 accepted, and it appeared as 15 differing verdicts until the harness minted a
scoped name (`carol@univ-c.example`) as the release notes require.

Not a gateway verdict, the first runs also found defects of the harness, all fixed and tested: a fault lifted twice (a
replica started a second time, the first process left running), the request count of a client that left before the
upstream saw it (allowed only for a request the client abandoned and the fake upstream never received), a port taken
between its choice and the gateway's bind, and a run started across a UTC midnight (the per-day budgets rolled over
inside it, and the clients legitimately paid more than the limit).

## A gateway finding: records wait in memory while the store hangs {#chaos-recorder}

**Fixed in v0.38.1.** Spooling (fsync) and shipping to the store are now two goroutines, so spooling never waits
on the store and the one-second bound holds whatever PostgreSQL does; exactly-once is unchanged. What follows
describes v0.38.0.

One run of `ha-kill-during-pg-restart` (seed 2, gateway v0.38.0) left four answered requests unbilled. It did not recur,
but a test shows it can happen: the accounting recorder is a single goroutine, and while the store hangs (PostgreSQL
restarting) it is blocked inside an insert with a 10 s timeout, so records of requests answered meanwhile are in memory,
not in the spool, and a SIGKILL loses them. The documented bound is about one second of traffic; here it is the length of
the stall. Reproducer and details: [go-llmflow/gateway#165](https://github.com/go-llmflow/gateway/issues/165), fixed by
[#166](https://github.com/go-llmflow/gateway/pull/166).

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
