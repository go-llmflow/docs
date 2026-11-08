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
| **Accounting during an outage** | With v0.9.0, **every record of the requests served during the outage was lost** (21 of 21 in the test, 2026-10-26). With v0.10.0's durable spool, **21 of 21 are stored once, with 21 distinct ids**. |
| Two replicas, one database | A concurrent first start works; keys are shared; a revocation reaches the other replica after `key_cache_ttl`; the audit chain stays one verifiable chain under two concurrent writers. |
| Upstream killed mid-stream, restarted | The client gets the partial stream; counters return to zero; the next requests are served. |
| Key rotated with 16 requests in flight | All 16 finish and are charged to the key; the old secret is refused at once and the new one is served. |
| Clock stepped back 2 h | The audit chain still verifies. |
| Export target down | The queue drops beyond 256 MiB and alerts. |
| Clients that stop reading | Released after `max_request_duration` + 5 s. |

**The accounting loss is fixed in v0.10.0.** Every batch of usage records is
appended to a spool file and fsync'd before it is inserted into PostgreSQL, and a
record is stored at most once (its id is the request id). From the README of
v0.10.0, measured on 2026-10-26: `TestChaosAccountingOutage` stores 21 of 21 records
after a PostgreSQL outage, where v0.9.0 lost all 21; with the real binary, 20 records
were spooled while PostgreSQL was unreachable, the gateway was killed with SIGKILL
and restarted, and 21 of 21 records were stored, with 21 distinct ids and no double
billing. This is not a promise of zero loss: a record still in the in-memory queue
when the process dies (about the last second of traffic, whatever the store does
since v0.38.1; with v0.10.0 to v0.38.0 a store that hung made it the length of the
stall, [#165](#v0381)) is lost. When `spool_max_bytes` (256 MiB per
process by default) is reached or the disk refuses the write, records are held in memory
only, up to 100,000 (v0.38.1), alerted as `accounting_records_lost`, and a request a
budget governs is refused with 503 `spool_full`; past the 100,000 they are lost, counted
in `llmflow_accounting_failed_total`. `store.spool_dir` is **mandatory** with the postgres store
since v0.10.0; see [Upgrading to v0.10.0]({{< relref "gateway.md" >}}#upgrading-v0100).

## Dependencies {#dependencies}

`govulncheck` (database of 2026-10-21) finds nothing in the gateway built with Go
1.27.3. go-authn/oidc v0.4.2, go-authn/rp v0.1.2 and go-volumes/s3 v0.2.0 declare
`go 1.27.1`: built with that toolchain, nine standard-library vulnerabilities are
reachable from them (net/http, crypto/tls, crypto/x509; fixed in Go 1.27.2 and
1.27.3). The gateway's binary is not affected because it is built with 1.27.3, but
those modules should raise their `go` line. go-authn/scim v0.1.0 is clean. The
gateway v0.10.0 takes the new releases at the go 1.27.3 floor: oidc v0.4.3, rp
v0.2.0, scim v0.1.1, s3 v0.2.1, pii v0.2.1 (pii v0.2.2 since gateway v0.36.0).

## Residual risks {#residual}

1. ~~Usage served during a store outage is not recorded~~ **fixed in v0.10.0**
   (see above). What remains is the last second of in-memory queue on a crash
   (since v0.38.1 whatever the store does), and a full spool with more than 100,000
   records pending.
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

## Fixes in v0.35.0: an independent audit of v0.34.0 {#v0350}

Three findings, each with a failing proof test that is now a regression test.

1. **MEDIUM: `finance_viewer` still saw people and keys.** The v0.30.0 fix covered `/admin/spend`
   only. `/admin/export`, `/admin/fluids?group_by=user` and `GET /admin/billing/closed?month=`
   (the raw snapshot, with per-person rows and `gateway_version`) still showed them. One shared
   guard now covers every report that groups, the closed snapshot is summed over the person for
   that role, and a test asks every GET admin route as a `finance_viewer`.
2. **MEDIUM: the JWT inference path ignored the stored (provider, subject) link.** A person of one
   institution behind a federation issuer could be charged as another institution's linked user.
   The scoped name is now the person only when no link exists for it or the link is this issuer's
   and this subject's; otherwise the person is the synthetic `jwt-<hash>`. Tying the scope to the
   institution claim of the token is not done: it needs a setting that says which claim carries it.
3. **LOW: `migrate-users` renamed synthetic `jwt-<hash>` people** into the legacy domain, at every
   run. It no longer plans them. If v0.34.0 `migrate-users --apply` ran while such records
   existed, rename them back (the statements are in the gateway README, "v0.35.0 fixes").

Also: a user name has one `@` at most, with something on each side (`a@b@c` was accepted).

## Fixes in v0.36.0: a second audit of v0.35.0 {#v0360}

Five findings; each proof test fails before its fix and is now a regression test.

1. **HIGH: billing bypass on `/v1/messages` streaming.** The usage of `message_start`
   (`output_tokens` 0 or 1) was taken as final, so a client that left after the text and before
   `message_delta` was charged one output token, `estimated:false`, and the delta-count fallback never
   ran: unlimited output under a hard quota. Usage is now final only once `message_delta` has been seen;
   a stream cut before it is charged the larger of what `message_start` reported and the deltas
   forwarded (`text_delta`, `thinking_delta`, `input_json_delta`), marked estimated. The same pattern was
   audited on chat, Responses, Bedrock invoke and converse streams and transcription: Responses took a
   usage object from any event (`response.in_progress`), and now reads only `response.completed`,
   `.incomplete` and `.failed`; the others were already final-only or charge the worst case.
2. **MEDIUM: a non-streaming request cut before its answer** was charged the prompt estimate and no
   output (free GPU time, fair-share priority kept). It is now charged the worst case under a hard
   budget, and `max_tokens` of output otherwise.
3. **MEDIUM: SCIM acted on roles of users it did not create.** Posting a SCIM user with the name of a
   hand-made user revoked its hand-granted `team_admin`, and group membership granted `team_admin` to
   non-SCIM users. A user is now provisioned, and its roles managed, only when its origin is `scim`.
4. **MEDIUM and LOW, pii v0.2.2:** a neighbouring character hid an e-mail (`...john.doe@x.org`) or an IPv6
   address (`IPv6:2001:db8::1`, `2001:db8::1:`), and a lower-case IBAN was missed. Fixed; the gateway takes pii v0.2.2.
5. **LOW, bench v0.9.2:** `fakevllm` did not tighten the mode of an existing recording file holding
   Authorization headers; it now sets 0600 and refuses a file that is not regular.

## Fix in v0.36.1: names compare without case, ASCII letters only {#v0361}

A patch release. Go's `strings.EqualFold` and `ToLower` fold Unicode: the Kelvin sign U+212A is a "k" and
U+017F (long s) an "s". The gateway now compares names with go-authn/oidc v0.7.0 `ASCIILower` and
`EqualASCIIFold` (the fold that produces the scoped names), so:

- the in-memory SCIM store no longer treats `kate@example.org` and its Kelvin-sign spelling as one user (the
  second was refused and a lookup returned both);
- a SCIM group `Lab Phyſics` is no longer mapped to the team role of `Lab Physics`;
- group names in `scim` blocks and the `domain` values are folded ASCII-only too;
- with PostgreSQL, the unique index on `lower(name)` is unchanged (it fails closed) and lookups return only
  ASCII-fold matches.

The dependency go-authn/oidc moves from v0.6.0 to v0.7.0.

## Fix in v0.37.1: a Bedrock stream that was complete but whose usage was not read (#157) {#v0371}

A patch release. The fuzz smoke of `FuzzBedrockStream` failed with "complete, but the usage was not read".
The stream converter took a `message_delta` as the final usage when each count was non-negative, and the
usage tracker then refused the same delta when the counts, each valid, added up past an int64
(`input_tokens` 9223372036854775807 and a `cache_read_input_tokens` of 1): a stream reported complete with
no usage read. The charge was already safe (the worst case is held), but the two disagreed. A first fix made
them share a bound, `openai.ValidMessagesCounts` (each count in [0, 2^60], so the sums cannot overflow; seed
`usage_sum_overflow`). The next fuzz smoke found a **second disagreement**, with its input kept by the new
artifact upload (the fuzz-smoke job now uploads the failing corpus entry): a usage key spelled `usAge`, which
Go's JSON decoder matches case-insensitively and the tracker's byte search for `"usage"` does not (seed
`usage_key_case`). Two copies of one rule had disagreed twice, so there is no second copy any more: the
converter feeds each event it writes to the tracker that `CopySSEFor` runs (`openai.MessagesStream`), and a
stream is complete only when the reader finds the final usage in it.

Also in v0.37.1, `check-config` refuses a `federation` block on a gateway that serves more than one model
(see [Being a node of a peer dispatcher]({{< relref "shared-purchase.md" >}}#ilaas-federation)).

## Fixes in v0.38.0: a third-party audit of v0.37.1 (breaking) {#v0380}

An audit by a third party of v0.37.1, all fixed in v0.38.0, each with a test that failed before.

1. **HIGH: a token's institution is its name's scope.** An issuer or provider with an `accept` on
   `schac_home_organization` filtered on the claim and took the name scopes from the same values, but
   nothing tied the two: a token of univ-d.fr's IdP stating `schac_home_organization = univ-c.fr` (or
   eve@univ-e.fr stating it) passed the filter, and a role map on that claim gave it univ-c.fr's team.
   Now the scope of the person's name must equal a value of the claim, and that value must be one `accept`
   lists; otherwise the login is refused (403 at SSO) or the token gets no key (JWT): no team, no role.
   go-authn/saml v0.6.0 and authn-bridge v0.23.9 close the other half: the bridge no longer releases a
   `schacHomeOrganization` that is not one of its IdP's scopes.
2. **MEDIUM: `POST /admin/users` refuses a bare name** once a domain is configured, to the proxy admin too
   (unless `legacy_unscoped_names` is on). Start refuses to run with a bare stored user once accounts are
   migrated, so one organisation admin could stop the next restart for every institution.
3. **MEDIUM: an organisation admin creates only names in its institutions' domains.** A person
   pre-created in another institution's domain was never managed by that institution's SCIM and was
   refused by its restricted SSO. A proxy admin is unrestricted. A user with no role and no link is also
   refused without `adopt`, now tested.
4. **MEDIUM: a refused request costs no guard-model call.** The pool slot, the spare check included, is
   taken before the pre-call guardrails, so a spare-only borrower on a full pool is refused 429 without
   a guard call; the slot is held while the guard judges. A guard call also runs at the priority of the
   request it judges (its band), no longer at 0 ahead of the owners' work.
5. **LOW.** The saturation fallback takes its slot through the same test as the first one. Fair share
   counts every choice: the in-flight estimate is the prompt plus `max_tokens` times the larger of `n`
   and `best_of` (47 tokens for `n` = 4 where the estimate of one choice plus three more was 76).
   `federation` no longer requires `borrow = "spare"`
   (see [Being a node of a peer dispatcher]({{< relref "shared-purchase.md" >}}#ilaas-federation)).
   Global policies (guardrails, budgets, rate limits applying to the whole gateway) are documented
   by design and unchanged.

CI: GitHub Actions are pinned by commit SHA, govulncheck is pinned, and the Renovate preset no longer
auto-merges v0 minors.

**Upgrade notes (this is a MINOR below v1, so breaking).**

- **Federation proxies.** Tokens and logins that carry an unscoped name, or a name whose scope is not the
  institution claim, are refused once the provider or issuer has an `accept` on
  `schac_home_organization`. Make the proxy release `eduperson_principal_name` or `voperson_id` with the
  institution's scope. See [Identity]({{< relref "identity.md" >}}#multi-idp).
- **Admin user creation.** `POST /admin/users` with a bare name, or by an organisation admin for another
  institution's domain, is refused. Use scoped names (`migrate-users` for the stored ones), or a proxy
  admin.

## Fix in v0.38.1: spooling no longer waits on the store (#165) {#v0381}

A patch release. The chaos campaign found (see [Chaos]({{< relref "chaos.md" >}}#chaos-recorder)) that the
accounting recorder was one goroutine that spooled a batch and then inserted it into PostgreSQL with a 10 s
timeout. While PostgreSQL hung, the records of requests answered meanwhile waited in memory, outside the
spool, and a `SIGKILL` lost them: the documented bound of about one second of traffic was in fact the length of
the stall.

Now one goroutine appends to the spool and fsyncs it, and **never waits on the store**; another ships what is
spooled. The bound holds whatever the store does. Exactly-once is unchanged: a record is stored at most once,
keyed by its id (the request id). When the spool is full or cannot be written, the recorder is degraded: records
are held in memory only, up to 100,000, and `accounting_records_lost` is alerted at once; a request that a
budget governs is refused with `503 spool_full` (fail closed) rather than answered unrecorded; the spool accepts
records again as soon as it is emptied. See [Durable accounting]({{< relref "gateway.md" >}}#upgrading-v0100).

## Security fixes in v0.18.1 {#v0181}

Not part of the 2026-10-26 audit above: two defects of v0.18.0 found by a **post-merge
review** and fixed in v0.18.1 (gateway#100). **Upgrade if you use
[workload profiles]({{< relref "workload-profiles.md" >}}).**

- **#98, a hard-budget hole.** A rolling window kept only twice its current length of
  spend, so widening `session_window` (for example 1 h to 5 h) forgot spend that was inside
  the new window and a session budget could be exceeded. A rolling budget now keeps its
  buckets for the longest window that can apply (`MaxRollingWindow`, 7 days: at most 2016
  buckets of 5 minutes per budget), on the memory and the PostgreSQL store. **Spend lost by
  a v0.18.0 prune before the upgrade is not recoverable from the buckets**; it is still in
  the usage records.
- **#99, an authorization bypass.** An organisation admin could grant their organisation's
  `viewer` role to any existing user, which made that person theirs to assign a profile,
  read and delete. Who a person belongs to is now decided by their **team** bindings (which
  SSO and SCIM also create). An organisation-level binding (`viewer`, `org_admin`) is
  **authority, not membership**: it gives no access to the person, whoever granted it. A
  person with only such bindings, or none, or a proxy role, belongs to the proxy admins. The
  same rule applies to the user budgets API. **Behaviour change:** an organisation admin no
  longer assigns the profile of another organisation admin who is in no team; ask a proxy
  admin.

## Security fix in v0.18.2: who may place a person in an organisation (#104) {#v0182}

v0.18.1 still let an organisation admin pull a user who had no binding, or who belonged to
another organisation, into their organisation by granting them a team role, and then act on
them. **Fixed in v0.18.2** (gateway#106). Every role binding now records its **provenance**:
`scim`, `sso_claim`, `proxy_admin` or `org_admin` (empty for unknown or legacy bindings),
shown as `provenance` in `GET /admin/users/{name}/roles`.

- **The first grant to a binding-less user, or to a user of another organisation, needs a
  proxy admin.** An organisation or team admin may grant a role only to a user already
  provisioned into that organisation: one holding a team binding of that organisation created
  by SCIM, an SSO claim or a proxy admin. (A user created by an org admin needs a proxy
  admin's first grant.)
- **A team binding that an org admin granted does not make the person theirs**: it is ignored
  by the profile and user-budget access rules. SCIM re-creates a binding it also wants under
  its own provenance.
- **Migration.** The column is added automatically (`provenance text NOT NULL DEFAULT ''`).
  Bindings stored before v0.18.2 have an unknown provenance, which is treated as legitimate:
  nothing says they were not, so no existing access changes at the upgrade. Only grants made
  after it are classified. Whether an old binding was in fact illegitimate is therefore not
  something the upgrade can tell you: review them.
