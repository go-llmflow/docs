---
title: "Deployment"
weight: 22
---

# Deployment {#deployment}

How the pieces fit on a co-owned GPU cluster, and which page says what. It states the
architecture as the gateway's README gives it and links to the pages that hold the
details; it adds no measurement and no recommendation of its own. What the READMEs do not
say is marked **open**, not invented.

Written from the README of gateway **v0.24.1** and of bench v0.7.0.

## The architecture {#architecture}

```text
 users, coding agents ──► N gateway replicas ──► vLLM instances
 (virtual keys, JWT,      (ha block, mutual TLS         ▲
  single sign-on)          on the internal port)        │ each replica follows the
                              │      │                  │ running jobs on its own
                              │      └── owner of each pool: a lease ─┐
                              ▼                                       ▼
                         PostgreSQL                      Slurm (slurmrestd, read only):
              (accounting, leases, sessions,             jobs of account svc_llm, QOS
               alert outbox, key revocations)            service, started and stopped
                              │                          by sbatch, scancel, scrontab
                              ▼
                   S3 with Object Lock (audit anchors)

 administrators ──► single sign-on through authn-bridge (SAML federations) or an OIDC provider
```

- **Clients** reach the gateway with a virtual key or a JWT; it maps them to an
  organisation, a team and a project and gives each request a vLLM priority from the
  institution's share ([Gateway]({{< relref "gateway.md" >}}),
  [Identity]({{< relref "identity.md" >}})).
- **vLLM instances run as Slurm jobs** of a service account, for example `svc_llm` with
  the QOS `service`. The gateway **follows** them and never starts or stops one.
- **Several gateway replicas** share the pools; PostgreSQL is the only shared component
  (no etcd, no Redis).
- **Batch and inference share the same GPUs.** How Slurm's Fair Tree, the QOS and
  preemption split them is the subject of the guide: see
  [Slurm: fair-share, QOS and preemption]({{< relref "guides/fairshare/slurm.md" >}}) and
  [Inference]({{< relref "guides/fairshare/inference.md" >}}).

## 1. PostgreSQL {#postgresql}

The one shared component. With the postgres store the gateway needs a **spool
directory** (`store.spool_dir`, mandatory since v0.10.0): every batch of accounting
records is written there and synced before it goes to the database, so records survive
an outage within stated bounds ([Upgrading to v0.10.0]({{< relref "gateway.md" >}}#upgrading-v0100)).
With an `ha` block PostgreSQL also holds the **pool leases** (expiry on PostgreSQL's own
clock), the **SSO sessions** (sealed, under hashed ids), the **soft-budget alert outbox**,
and carries the **key revocations** (`NOTIFY`). The version used by the chaos tests was
PostgreSQL 18.

- The spool is per process (its cap is per process too); a process that dies leaves its
  file, and the next process on the directory adopts and stores it.
- *Open: sizing, backup and restore of the database are not in the README.*

## 2. vLLM as Slurm jobs {#slurm-jobs}

A `slurm` block inside a `model`: the job-name convention
`llmflow:<model>:<port>[:<capacity>]`, the allowed accounts, partitions and users,
admission after `/health` and `/v1/models`, drain on job end or after three failed health
checks, the pool's capacity recomputed and audited, fail-closed after `stale_after`.

One instance, as a job (the key goes in `VLLM_API_KEY`, never `--api-key`):

```bash
#!/bin/bash
#SBATCH --job-name=llmflow:llama:8000:8
#SBATCH --account=svc_llm --qos=service --partition=gpu
#SBATCH --nodes=1 --gres=gpu:4 --exclusive --time=7-00:00:00
#SBATCH --signal=B:TERM@120
VLLM_API_KEY=$(cat /etc/llmflow/vllm.key); export VLLM_API_KEY
exec vllm serve meta/llama-3.3-70b --served-model-name meta/llama-3.3-70b \
  --host 0.0.0.0 --port 8000 --tensor-parallel-size 4
```

Day and night, with `scrontab` (needs `ScronParameters=enable`): the jobs end at 20:00,
the gateway drains them, the pool shrinks and the reservations follow, so the GPUs can be
lent to a borrowing QOS; at 07:00 the new jobs are admitted as soon as vLLM answers.

```text
#SCRON --account=svc_llm --qos=service --partition=service -t 0:05:00
0 7 * * 1-5 sbatch /opt/llmflow/vllm-llama.sbatch
#SCRON --account=svc_llm --qos=service --partition=service -t 0:05:00
0 20 * * 1-5 scancel --account=svc_llm --name=llmflow:llama:8000:8
```

- The node's vLLM port must be reachable by the gateway replicas only (firewall):
  `/metrics` is unauthenticated, and the key protects `/v1` only.
- slurmrestd needs `AuthAltTypes=auth/jwt`; the gateway's token is issued with
  `scontrol token username=svc_llm lifespan=...`, **read at start only**, so its lifespan
  must outlast your restart cycle.
- Full text, safety rules and the metrics: [Slurm discovery]({{< relref "gateway.md" >}}#slurm-discovery).
  What was tested on a real Slurm: [Benchmarking]({{< relref "benchmarking.md" >}}#verified-slurm).
- *Open: the README's tests used a small CPU model on one node; no GPU node and no
  multi-node tensor-parallel job were tested.*

## 3. N gateway replicas {#replicas}

The `ha` block, off by default. With **Slurm discovery** (works with `ha` since v0.15.2)
each replica discovers and health-checks on its own, for its own routing, and only the
owner's view sets the pool's capacity and reservations. This assembled configuration
(replicas differ by `replica_id`, `listen` and `advertise`) passes
`llmflow-gateway check-config` v0.24.1 with the key, DSN, secret and token files present:

```hcl
store {
  kind      = "postgres"
  dsn_file  = "/run/secrets/dsn"
  spool_dir = "/var/spool/llmflow"
}

ha {
  listen        = "10.0.0.5:9444"
  advertise     = "10.0.0.5:9444"
  peer_networks = ["10.0.0.0/24"]
  secret_file   = "/run/secrets/ha"
}

org "instA" {
  share = 1
  team "a-physics" {}
}

pool "gpu" { capacity = 0 }

model "llama" {
  upstream_model = "meta/llama-3.3-70b"
  pool           = "gpu"
  key_file       = "/etc/llmflow/vllm.key"
  slurm {
    url        = "https://slurmrestd.cluster:6820"
    token_file = "/etc/llmflow/slurm.jwt"
    accounts   = ["svc_llm"]
    partitions = ["gpu"]
  }
}
```

(The same file with the paths of the test machine; master and cache-salt keys omitted
above.)

**Mutual TLS between the replicas** is optional: `tls_cert_file`, `tls_key_file` and
`tls_ca_file` (all three or none), PEM files or `vault:` references, with
`tls_server_name` naming the DNS name in each replica's certificate. **Use v0.16.1 or
later**: v0.16.0 rejected every multi-line PEM file. With v0.16.1, `check-config`
accepts real PEM files and refuses a certificate whose key does not match. The key file
must not be readable by group or others. A live test (bench v0.6.0, 2026-10-27, two
gateway v0.16.1 processes, PostgreSQL 18.6) saw 8 of 8 admissions forwarded to the owner
over mTLS, and a replica with a certificate from another CA get no answer.
See [High availability]({{< relref "gateway.md" >}}#ha).

- Behaviour in a failover, the chaos results and the open items: [High availability]({{< relref "gateway.md" >}}#ha).
  A killed owner was replaced in 1.3 s with no request failed; a partitioned owner never
  overlapped the new one (805 samples).
- Without a reachable owner a request is admitted at the lowest borrowing priority, or
  refused with 503 under `on_owner_unreachable = "reject"`.
- Per-key rate limits are still counted per replica.
- Put the replicas behind a load balancer of your choice; the README does not prescribe one.
- *Open: the latency a forwarded admission adds, with and without mutual TLS, is not
  measured.*

## 4. Audit anchors on S3 {#anchors}

The admin audit log is hash-chained with signed checkpoints. Its guarantees end at its
newest **anchor**: without an exported anchor, truncation or rollback of the database goes
undetected. Anchors can be written to S3 (or Ceph RGW), under **Object Lock** retention
when the bucket has it enabled; create the bucket that way, since MinIO and Ceph require
it at creation. Keep the signing key in OpenBao or Vault. See
[Audit and logging]({{< relref "audit.md" >}}#anchors) and its
[threat model]({{< relref "audit.md" >}}#threat-model).

## 5. Identity: authn-bridge and single sign-on {#identity}

Administrators sign in through an OpenID provider. For a SAML federation (RENATER,
eduGAIN) the gateway is registered as a client of
[`go-authn/bridge`](https://github.com/go-authn/bridge) (`authn-bridge`), an OpenID
provider in front of the federation, and `sso.issuer` points at the bridge. SCIM
provisions users and groups. With an `ha` block the SSO sessions live in PostgreSQL, so
they outlive a replica. See [Identity]({{< relref "identity.md" >}}).

## 6. Reports and reconciliation {#reports}

What finance sees, and how it is set beside Slurm's own accounting (`sreport`):
[Finance reports]({{< relref "finance-reports.md" >}}). The PDF is written by
[go-pdfkit](https://go-pdfkit.github.io/docs/).

## 7. Checking a deployment {#checking}

- `llmflow-gateway check-config -c <file>` validates the configuration and every secret
  file it names. Run it on each replica's file.
- The acceptance tests of the guide:
  [Audit and acceptance]({{< relref "guides/fairshare/audit-and-acceptance.md" >}}).
- The tools that measure the gateway against a real vLLM, and a real Slurm:
  [Benchmarking]({{< relref "benchmarking.md" >}}).
- What was audited and attacked: [Security]({{< relref "security.md" >}}).

## What is still missing {#todo}

The repositories do not document, and this page does not invent: a network diagram with
the ports (the client listener, the internal replica port, slurmrestd, vLLM) and the
firewall rules, a rolling-upgrade procedure across replicas, sizing, and backup and restore
of PostgreSQL.
