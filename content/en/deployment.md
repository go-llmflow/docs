---
title: "Deployment"
weight: 22
---

# Deployment {#deployment}

How the pieces fit on a co-owned GPU cluster, and which page says what. **This page is
a skeleton**: it states the architecture as the gateway's README gives it and links to the
pages that hold the details. It adds no measurement and no recommendation of its own;
what is not in a README at the tags below is marked as open.

Written from the README of gateway **v0.15.1** and of bench v0.5.0.

## The architecture in one picture {#architecture}

```text
 users, coding agents ──► gateway (replicas) ──► vLLM instances
 (virtual keys, JWT)        │  ▲                   ▲
                            │  │ owner of each     │ follow the running jobs
                            ▼  │ pool: a lease     │ (slurmrestd, read only)
                        PostgreSQL              Slurm: jobs of account svc_llm,
                  (accounting, leases,           QOS service, started and stopped
                   sessions, alerts)             by sbatch, scancel and scrontab
```

- **Clients** reach the gateway with a virtual key or a JWT; it maps them to an
  organisation, a team and a project and gives each request a vLLM priority from the
  institution's share. See [Gateway]({{< relref "gateway.md" >}}) and
  [Identity]({{< relref "identity.md" >}}).
- **vLLM instances run as Slurm jobs** of a service account, for example `svc_llm` with
  the QOS `service`. The gateway **follows** them and never starts or stops one.
  See [Slurm discovery]({{< relref "gateway.md" >}}#slurm-discovery).
- **Several gateway replicas** can share the pools, with PostgreSQL as the only shared
  component. See [High availability]({{< relref "gateway.md" >}}#ha).
- **Batch and inference share the same GPUs.** How Slurm's Fair Tree, the QOS and
  preemption split them is the subject of the guide, in
  [Slurm: fair-share, QOS and preemption]({{< relref "guides/fairshare/slurm.md" >}}) and
  [Inference]({{< relref "guides/fairshare/inference.md" >}}).

## 1. One replica, static upstreams {#single}

The starting point, with nothing optional: one gateway, a PostgreSQL store with its
spool directory, `upstream` blocks that name each vLLM server.

- Configuration and secrets: [Gateway: configuration]({{< relref "gateway.md" >}}#configuration).
- The durable accounting spool: [Upgrading to v0.10.0]({{< relref "gateway.md" >}}#upgrading-v0100).
- Limits to know: [Limitations]({{< relref "gateway.md" >}}#limitations).
- *Open: a worked sizing example (replica CPU and memory, PostgreSQL) is not in the
  README, and no performance figure of the current release is measured.*

## 2. vLLM as Slurm jobs {#slurm-jobs}

The `slurm` block of a `model`: the job-name convention `llmflow:<model>:<port>[:<capacity>]`,
the allowed accounts and partitions, admission after `/health` and `/v1/models`, drain on
job end or after three failed health checks, the pool's capacity recomputed and audited,
fail-closed after `stale_after`.

- The full text, an `sbatch` template (the key in `VLLM_API_KEY`, never `--api-key`) and
  a day and night `scrontab`: [Slurm discovery]({{< relref "gateway.md" >}}#slurm-discovery).
- What was tested on a real Slurm, and what was not: [Benchmarking]({{< relref "benchmarking.md" >}}#verified-slurm).
- The accounts, QOS and preemption that the jobs run under:
  [the guide's Slurm page]({{< relref "guides/fairshare/slurm.md" >}}).
- *Open: the README's tests used a small CPU model on one node; no GPU node and no
  multi-node tensor-parallel job were tested.*

## 3. Several replicas {#replicas}

The `ha` block, off by default: a lease per pool with a fencing epoch, the owner's
decision forwarded between replicas, shared SSO sessions, an alert outbox and revocation
by `NOTIFY`.

- The full text, the chaos results (owner killed: takeover in 1.3 s; partition: never two
  owners in 805 samples; clock skew of 2 minutes harmless) and the open items:
  [High availability]({{< relref "gateway.md" >}}#ha).
- **Not yet combinable with Slurm discovery**: `check-config` refuses the pair.
- *Open: the latency a forwarded admission adds is not measured; per-key rate limits are
  per replica; the internal channel is not encrypted.*

## 4. Reports and reconciliation {#reports}

What finance sees, and how it is set beside Slurm's own accounting:
[Finance reports]({{< relref "finance-reports.md" >}}).

## 5. Checking a deployment {#checking}

- `llmflow-gateway check-config -c <file>` validates the configuration and every secret
  file it names.
- The acceptance tests of the guide:
  [Audit and acceptance]({{< relref "guides/fairshare/audit-and-acceptance.md" >}}).
- The tools that measure the gateway against a real vLLM, and now a real Slurm:
  [Benchmarking]({{< relref "benchmarking.md" >}}).
- What was audited and attacked: [Security]({{< relref "security.md" >}}).

## To be written {#todo}

This skeleton deliberately stops where the READMEs stop: a network diagram with the
ports (the client listener, the internal replica port, slurmrestd, vLLM), an upgrade
procedure across replicas, and backup and restore of PostgreSQL are not documented by
the repositories yet, and are not invented here.
