---
title: "Workload profiles"
weight: 30
---

# Workload profiles {#workload-profiles}

An administrator, often in finance or HR rather than IT, gives each person a **workload
profile in working days per week** instead of a money budget: a quarter day, half a day, one
day ... seven days a week. The [gateway]({{< relref "gateway.md" >}}) turns the profile into
[hard budgets]({{< relref "gateway.md" >}}#hard-budgets), so **a profile is never exceeded**
(with the edges listed there). Since **v0.18.0**. This page is written from the repository's
README at that tag; it was not run on a production cluster.

## The unit: a workday {#workday}

A workday is what one person working with the assistant for a full day consumes, in each
cost unit: GPU-hours for the local pools and a currency for each external provider. It is
**measured, not guessed**:

- `/ui/profiles/calibrate` (or `GET /admin/profiles/calibrate?weeks=4`) shows the
  distribution of the cost of every *active person-day* (a person with any cost that day)
  over the last weeks, and **proposes its 75th percentile**.
- A **proxy admin** accepts a value (`POST /admin/profiles/calibrate
  {"unit": "USD", "amount": 13}`, audited), and every profile follows at once.
- For reference, Anthropic reports about $13 per developer per active day for Claude Code
  across enterprise deployments ([costs](https://code.claude.com/docs/en/costs)), but your own
  measurement is what counts.
- An accepted calibration takes precedence over a `workday` block in the configuration, which
  only gives a starting value.
- **A priced request in a unit without a workday is refused (503) for a person under a
  profile**: nothing could hold its cost against the profile.

## What a profile becomes {#budgets}

For a person with a profile of *d* days, in every unit that has a workday:

| Budget | Period | Limit |
| --- | --- | --- |
| `profile_week` | the ISO 8601 week, reset Monday 00:00 UTC | *d* x workday |
| `profile_session` | a rolling window, `session_window` (5 h by default) | `session_share` x workday |
| `profile_overflow` | the calendar month, only when overflow is allowed for the person | `overflow_cap` x workday |

**The session window** stops one person from burning the week in one sitting, as the
five-hour session limit of Anthropic's plans does
([Max plan](https://support.claude.com/en/articles/11049741-what-is-the-max-plan),
[Team plan](https://support.claude.com/en/articles/9266767-what-is-the-team-plan)). The
profiles **do not borrow those plans' names**: their ratios do not match a day scale. The
labels default to the day count (`"2 days / week"`) and can be changed.

**Overflow** is the analogue of Anthropic's capped usage credits
([manage usage credits](https://support.claude.com/en/articles/12429409-manage-usage-credits-for-paid-claude-plans)).
When the week or the session refuses a request and an administrator allowed overflow for that
person, the request is admitted on the profile's monthly `overflow_cap` instead. It is marked
`overflow` in its record, counts for the overflow budget only, and still counts for every
other budget of its scope (team, organisation, project). An organisation-wide cap is an
ordinary `org` budget. Without overflow, the request is refused.

**Refusals** are 429 `insufficient_quota` with `X-Llmflow-Budget-Reset`. The message is in
workdays and names only the caller's own allowance, for example:

```text
weekly allowance of 2 days used; resets Monday 2 Nov 00:00 UTC
session allowance of 0.5 day per 5h: 0.1 day left, and this request may cost up to 0.2 day (send a smaller max_tokens, or wait: room again from Wednesday 28 Oct 15:35 UTC)
```

For the session window the header gives when enough has left the window for the request, not
the end of a period.

**The rolling window** (`"period": "rolling:5h"`, a general budget period, 5 minutes to 168
hours in whole 5 minutes, no `soft_limit`) keeps its spend in 5-minute buckets, so its spent
counts at most 5 minutes more than the window, never less; a request counts in the bucket of
its end. Replica clocks must be in sync, as for calendar periods.

## Who assigns {#delegation}

- A **proxy admin** assigns anyone.
- An **org_admin** assigns a person all of whose role bindings are in their organisation.
- A **team_admin** assigns a person all of whose bindings are in teams they administer.
- A person with a binding in another organisation, a proxy role, or no binding at all (a JWT's
  user) is the proxy admins' only: a profile limits the person everywhere, so one organisation
  does not decide it for another.
- People read their own profile; administrators and viewers read the people of their scope.

Every assignment is stored with its history (who, when, which profile) and audited.

## Configuration {#configuration}

```hcl
workday {               # optional starting values; a calibration accepted in the UI wins
  gpu_hours = 0.8
  USD       = 13        # one attribute per unit, as the models spell it
}
profile "q" {           # id: [a-z0-9_-]{1,32}; at most 64 profiles
  days          = 0.25  # 0..7 days a week
  session_share = 0.25  # (0, 1] workday per session window
}
profile "2d" {
  label         = "2 days / week"  # default: the day count; at most 64 printable bytes
  days          = 2
  session_share = 0.5
  overflow_cap  = 2     # optional, 0..217 workdays a month beyond the profile
}
default_profile = "q"   # optional: everyone without an assignment
session_window  = "5h"  # 5m..24h in whole 5 minutes
```

This block, with `budgets`, models and organisations around it, passes
`llmflow-gateway check-config` v0.18.0. HCL allows one attribute only in a block written on
one line (`profile "q" { days = 0.25 }`): write one attribute per line. A person with
`default_profile` gets their budgets at their first request, which waits up to 3 s for them
to arm, once. The budgets API neither lists nor changes the profile budgets
(`PUT /admin/budgets` with a `profile_*` kind answers 404).

## Admin API {#api}

```text
PUT    /admin/users/{user}/profile   {"profile": "2d", "overflow": false}
DELETE /admin/users/{user}/profile   # back to default_profile, or none
GET    /admin/users/{user}/profile   # the person's week and session, in days, and the history
GET    /admin/profiles               # the profiles, the default, the workday in force per unit and its source
GET    /admin/profiles/people        # ?org= or ?team=: everyone the principal may read
GET    /admin/profiles/capacity      # see below
GET    /admin/profiles/calibrate     # ?weeks=1..26 (proxy roles)
POST   /admin/profiles/calibrate     # {"unit", "amount"} (proxy admins)
```

## The admin pages {#ui}

- `/ui/profiles` shows each person's week as a bar in days ("1.3 / 2 days") with when it
  resets, their session and their overflow, and offers the assignment form to those who may
  assign.
- `/ui/profiles/capacity` is the check a manager needs: it compares the days assigned each week
  to an organisation's people with what its share of the pools serves (share x C x the measured
  cost per request-slot-hour over the last 7 days of records x 168 h, in workdays), and shows
  the oversubscription ratio as a seat count would ("assigned 140 days a week for a capacity of
  about 90, x1.6"). Oversubscription is normal, since people do not all work at the same time
  and fair share arbitrates when they do. A pool without measured load says "not measured"; an
  external provider has no capacity, since it is billed per request.
- `/ui/profiles/calibrate` is the calibration above.

## Reports {#reports}

[Finance reports]({{< relref "finance-reports.md" >}}) gain a `profile` dimension
(`group_by=profile`, the reports page, and the CSV, XLSX and PDF exports): the profile in force
when each request was admitted, so a change of profile never rewrites history.

## Upgrading from v0.17 to v0.18 {#upgrade}

- Start-up adds the tables `llmflow_budget_buckets`, `llmflow_profile_assignments` and
  `llmflow_workdays`, and the `profile` and `overflow` columns of `llmflow_usage`. Existing
  records have no profile and are not overflow. Nothing changes without `profile` blocks.
- With profiles, **calibrate a workday for every unit a person under a profile uses before
  setting `default_profile`**; until then their priced requests in that unit are refused.
- **Upgrade every replica before assigning profiles.** A v0.17 replica does not know the
  profile kinds: it admits without holding on them, and the arming of each new profile budget
  covers its requests only until their records are stored.
- Fixed on the way: a budget deleted and created again now re-reads its period's spend from
  the records.

## What is not here {#not-yet}

Soft quotas and warnings at 75 % and 95 % of an allowance are **not shipped** (they are
tracked in gateway#96). No performance figure for profiles exists, and no live test of them
is cited on this page.
