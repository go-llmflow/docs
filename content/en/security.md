---
title: "Security"
weight: 26
---

# Security {#security}

On **2026-10-26** an independent, adversarial review of gateway v0.8.0 (commit
`b23064a`) was made by an agent that did not write the code, together with chaos
tests on the parts that keep state. Its fixes are released in **v0.9.0**. This page
summarises it; the gateway's README, section "Security audit 2026-10-26", at the
[v0.9.0 tag](https://github.com/go-llmflow/gateway/blob/v0.9.0/README.md#security-audit-2026-10-26)
is the source, and where the two differ the README wins.

## Method {#method}

- **Read, then attack.** The threat model and the code were read first. vLLM's own
  sources were opened wherever the gateway relies on how vLLM parses a request
  (vllm-project/vllm `main` at `256d8b4`: the scoring, embedding and Anthropic
  protocol files).
- **Scope.** Every package of the module, and the modules it pulls for
  authentication and storage: go-authn/oidc v0.4.2, go-authn/rp v0.1.2,
  go-authn/scim v0.1.0 and go-volumes/s3 v0.2.0. The fixes of earlier reviews were
  rechecked and hold, with their tests.
- **A route by role matrix.** `TestAuthzMatrix` calls every admin route of the
  router (a route without a row fails the test) as 12 principals on one
  organisation's objects: the master key, `proxy_admin`, `proxy_viewer`, the
  `org_admin` of the organisation and of another one (the cross-organisation
  attacker), `team_admin`, `member`, a team viewer and an organisation viewer, a user
  without roles, an inference key, and a JWT-shaped bearer. Listings are checked for
  leaks to outsiders, and the matrix was checked against planted mistakes.
- **Fuzzing.** The 29 existing targets ran for 10 minutes each on v0.8.0, three at a
  time with two workers each: none failed, and no new crasher was found. A new
  target, `FuzzScopeEveryRoute`, ran for 60 s with no failure; it caught a planted
  regression on its first seed and is part of the CI fuzz smoke.
- **`govulncheck`** (vulnerability database of 2026-10-21) on the module and on the
  four modules above.

## Findings {#findings}

| ID | Severity | Finding | Fix |
| --- | --- | --- | --- |
| A1 | medium | Multimodal `uuid`s were scoped per tenant on chat and Responses only. On rerank, tokenize and embeddings the client's uuid reached vLLM unchanged: one tenant could probe or poison another's cached media. | scoped on all routes, still reusable within a tenant (#48, #55) |
| A2 | medium | A client that never read its answer blocked its handler for ever, holding up to 64 MiB of buffered answer, its key's in-flight slot and its fair-share grant. A few connections could exhaust the gateway's memory. | every response has a write deadline of `max_request_duration` + 5 s (#49) |
| A3 | **high** | Any organisation admin could re-enable a user a proxy admin had disabled, with every role the user kept, `proxy_admin` included (`POST /admin/users` on an existing name). | `POST /admin/users` on a disabled user answers 403 unless the caller is a proxy admin (#50) |
| A4 | low | A key created while its team was being disabled stayed valid, and disabling a team did not clear the cache of verified JWTs. | disabling an organisation, team or project cuts its keys and cached JWTs at once (#51) |
| A5 | medium | Export queues were bounded in objects only: with S3 down and content logging on, up to 8 GiB of request logs could pile up in memory. | queues capped at 256 MiB; beyond that, objects are dropped, counted and alerted (#52) |
| A6 | medium | Store calls were unbounded: during a partition with PostgreSQL, requests with an uncached key hung (no answer after 20 s in the test), and a stuck budget refresh stopped every later one. | key lookups give up after 5 s (503), a budget refresh after 30 s, admin, UI and SCIM after 60 s (#54) |
| A7 | low | Two replicas started together on an empty database raced on the schema, and one failed to start. | schema creation is serialised (#54) |
| A8 | low | The admin API, SSO and SCIM answers had no `nosniff`, CSP or frame denial (the UI had them). | those answers carry `nosniff`, a CSP and `no-store` (#56) |
| A9 | **high** | go-llmflow/pii v0.1.0, used by the `pii` guardrail, is quadratic on long runs of digits: any key holder of a guarded team could pin a core per request. | fixed in pii v0.2.0, bumped in the gateway (#57) |

Each fix has a test that fails without it. A3 is the only one that crosses
organisations; the matrix found no other cross-organisation hole on the 39 admin
routes.

### The pii defect (A9) {#a9}

Its own README, at v0.2.0, says what was measured: 1 MiB of `1 1 1 ...` took
11 min 56 s with v0.1.0 (measured 2026-10-26, on a loaded machine). Since v0.2.0
card candidates are bounded at 19 digits, patterns of bounded length are searched in
512-byte windows, and `TestLongRunsStayLinear` checks the growth from 128 KiB to
2 MiB on twelve adversarial inputs (linear is x16, and the test fails above x64).
The gateway's README gives the same defect as 5 to 20 minutes; the two figures were
measured separately. Upgrade pii to v0.2.0 or later.

The `hide_secrets` guardrail uses gitsafe's `redact`. Its README at v0.17.1 says that
since that version it finds the issuers' prefixes with `strings.Index` instead of
stepping a regular-expression automaton through the whole text: identical results
(compared on 20,000 generated texts and by a fuzz target), and about 30 times faster
on the gateway's 2 KB benchmark prompt, **4 us instead of 130 us on an Apple M4 Max
on 2026-10-26, measured on a loaded machine, so only an order of magnitude**.

## Verified without a finding {#verified}

No cross-organisation IDOR on the 39 admin routes; CSRF protection (synchronizer
token, `Origin`, `Sec-Fetch-Site`) on every state change of the UI and of
cookie-authenticated API calls; a fresh session at login, `__Host-` cookies, no
open redirect; every outbound URL comes from the configuration and no client
redirect is followed; keys, tokens and the master key compared in constant time
over hashes and never logged; a fallback keeps the original organisation's cache
salt, uuids and priority; scheduled-rotation targets stay under their team's Vault
prefix; export keys validated against traversal; request bodies read only after
authentication and bounded.

Against a real vLLM, the end-to-end run of
[bench]({{< relref "benchmarking.md" >}}#verified-vllm) v0.2.0 (2026-10-26)
confirmed that a client's `priority`, `vllm_xargs`, `extra_body` and `cache_salt`
change neither the scheduling order nor the cache isolation, and that
prefix-cache isolation per organisation holds: vLLM's own counters showed 768 of
814 prefix tokens hit within one organisation and 0 of 814 from another.

## Chaos results {#chaos}

PostgreSQL ran behind a fault-injecting proxy (the README names a real
PostgreSQL) that could refuse connections or blackhole them.

| Fault | Behaviour |
| --- | --- |
| PostgreSQL down | An unknown key gets 503, not 401, and is not cached as invalid. A cached key is served until the budgets are 3 refresh periods old, then everything fails closed; everything recovers. |
| PostgreSQL partitioned | 503 within 5 s; fail-closed on time; recovery when the partition heals. |
| PostgreSQL restarted | A call that draws a dead pooled connection fails safe with a 503 (one call in the test); the pool recovers by itself. |
| **Accounting during an outage** | **Every record of the requests served during the outage is lost** and counted as failed. |
| Two replicas, one database | A concurrent first start works; keys are shared; a revocation reaches the other replica after `key_cache_ttl`; the audit chain stays one verifiable chain under two concurrent writers. |
| Upstream killed mid-stream, restarted | The client gets the partial stream; counters return to zero; the next requests are served. |
| Key rotated with 16 requests in flight | All 16 finish and are charged to the key; the old secret is refused at once and the new one is served. |
| Clock stepped back 2 h | The audit chain still verifies. |
| Export target down | The queue drops beyond 256 MiB and alerts. |
| Clients that stop reading | Released after `max_request_duration` + 5 s. |

**The accounting loss is being fixed.** A durable spool for usage records is
planned for **v0.10.0**; until it ships, usage served during a store outage is not
recorded, so budgets and reports undercount it, and fail-closed bounds the window
to 3 refresh periods.

## Dependencies {#dependencies}

`govulncheck` (database of 2026-10-21) finds nothing in the gateway built with Go
1.27.3. go-authn/oidc v0.4.2, go-authn/rp v0.1.2 and go-volumes/s3 v0.2.0 declare
`go 1.27.1`: built with that toolchain, nine standard-library vulnerabilities are
reachable from them (net/http, crypto/tls, crypto/x509; fixed in Go 1.27.2 and
1.27.3). The gateway's binary is not affected because it is built with 1.27.3, but
those modules should raise their `go` line. go-authn/scim v0.1.0 is clean.

## Residual risks {#residual}

1. Usage served during a store outage is not recorded (see above).
2. SCIM adopts an existing gateway user with the same `userName`, and re-enables it
   if a proxy admin had disabled it by hand. The identity provider is already
   trusted for usernames through SSO.
3. Roles that SSO maps from ID-token claims are fixed at login, for the session's
   lifetime. Stored roles and the disabled flag are read on every request.
4. A member may point their own key's scheduled rotation at any field under the
   team's Vault prefix, overwriting a secret delivered there for another key of the
   team: a denial of service inside the team, with no read access.
5. Secret files are read whatever their permissions (Kubernetes mounts secrets 0644
   by default). Keep them in a directory only the gateway's user can read.
6. A tool server's error message (at most 4 KiB) is returned to the MCP client: tool
   servers are configured, so they are trusted.

The gateway's [limitations]({{< relref "gateway.md" >}}#limitations) list the open
defects, re-checked in this audit.
