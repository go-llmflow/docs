---
title: "Fair-share on a co-owned GPU cluster"
weight: 10
---

# Fair-share on a co-owned GPU cluster {#fair-share}

How to share four HGX B300 servers (32 GPUs) between two institutions, **A** and
**B**, with Slurm for batch computing, vLLM for inference, a gateway for keys and
quotas, and VS Code to reach it all. The sharing must respect each institution's
share and leave no GPU idle.

This is an English adaptation of a design and operations document, version 1.1
of 23 October 2026. Every parameter cited refers to the vendors' documentation
consulted on that date (see the [bibliography]({{< relref "bibliography.md" >}})).
The measurements are real and dated: they were made on **LiteLLM 1.106.2 on
23 October 2026**.

The guide has three layers, one page each:

1. **Slurm** guarantees each institution its share (QOS caps, preemption of
   borrowing jobs) and divides the rest by fair-share (the Fair Tree
   algorithm): [Slurm]({{< relref "slurm.md" >}}).
2. **vLLM** serves the language models in long Slurm jobs, and a **gateway**
   in front of it authenticates users, applies per-institution quotas and
   counts consumption in tokens: [Inference]({{< relref "inference.md" >}}).
3. **One accounting**, in GPU-hours, ties the two:
   [Accounting]({{< relref "accounting.md" >}}).

Then [VS Code access]({{< relref "vscode.md" >}}), [identity with your
federation]({{< relref "identity.md" >}}) (RENATER, eduGAIN), the
[security audit, acceptance tests and governance]({{< relref "audit-and-acceptance.md" >}}),
and the [bibliography]({{< relref "bibliography.md" >}}).

## Context and goals {#context}

### The hardware {#hardware}

Each server is an HGX B300 with 8 Blackwell Ultra SXM GPUs linked by NVLink 5.
Figures are NVIDIA's (the HGX datasheet, the Blackwell Ultra technical blog and
the DGX B300 user guide).

| | per GPU | per server (8 GPUs) |
| --- | --- | --- |
| HBM3e memory | 288 GB | 2.3 TB (8 x 288); 2.1 TB on the HGX datasheet |
| HBM bandwidth | 8 TB/s | -- |
| NVLink 5, GPU to GPU (bidirectional) | 1.8 TB/s | 14.4 TB/s in total |
| NVFP4, dense / sparse | 15 / 20 PFLOPS | 108 / 144 PFLOPS |
| FP8, dense / sparse | 5 / 10 PFLOPS | 36 / 72 PFLOPS |
| FP64 | -- | 10 TFLOPS |
| INT8 (sparse) | -- | 3 POPS (72 on HGX B200) |
| MIG partitions | up to 7 instances (for example 7 x 34 GB) | -- |
| Power | TGP up to 1,400 W | DGX B300: 14.5 kW, 15 kW max |

Per-GPU and per-server figures are not always in a ratio of 8: 108/8 = 13.5
PFLOPS rather than 15, and 2.1 TB rather than 8 x 288 GB. NVIDIA states that
memory and SM count vary by part number. Use the values of *your* vendor's
datasheet. As an example, the DGX B300 has 2 Xeon Platinum 6776P and 2 TB of
RAM (up to 4).

{{< callout type="warning" >}}
**The B300 is an AI machine, not a double-precision scientific one.** 10 TFLOPS
of FP64 for 8 GPUs, and an INT8 rate divided by 24 relative to the B200. A
classic FP64 code (CFD, quantum chemistry, finite elements) wastes its
institution's share here. The agreement between the institutions should say so;
such codes belong on other facilities.
{{< /callout >}}

The cluster has **32 GPUs**. A practical consequence: a model served with tensor
parallelism 8 (`--tensor-parallel-size 8`) occupies **a whole server**, because
NVLink does not connect two servers.

### Ownership {#ownership}

Institution A bought 3 whole servers and the remaining quarter of the 4th;
institution B bought the other three quarters of the 4th. If a later purchase
changes the split, update this table and the values derived from it
(`Fairshare`, `GrpTRES`).

| Institution | Servers | GPUs | Share | Slurm Fairshare |
| --- | --- | --- | --- | --- |
| A | 3 + 1/4 | 26 | 81.25 % | 26 |
| B | 3/4 | 6 | 18.75 % | 6 |
| Total | 4 | 32 | 100 % | 32 |

Slurm shares are expressed in GPUs, which keeps them readable. Slurm normalises
against siblings (`S = S_self / S_(self+siblings)`), so only the ratio 26:6
counts.

### Objectives {#objectives}

1. **Guarantee**: each institution gets its share (26 or 6 GPUs) within a
   bounded, known delay, even when the cluster is full.
2. **No loss**: a GPU its owner does not use is lent to the other institution.
3. **Fairness over time**: over a sliding window, each one's consumption tends
   to its share. Whoever borrowed a lot goes behind the other.
4. **One accounting**: batch and inference are both read in GPU-hours, for
   re-invoicing and governance.
5. **Security**: GPU isolation between jobs, and no inference service reachable
   without authentication.

## Design choices {#design}

### Three possible models {#three-models}

| Model | Advantages | Drawbacks |
| --- | --- | --- |
| Fixed partitions | simple; immediate guarantee | GPUs lost when the owner is idle; B can never run a TP=8 job (8 GPUs) |
| Pure fair-share | no GPU lost; fairness in the long run | no guarantee on the *delay*: B may wait for A's long jobs to end |
| **Hybrid (chosen)** | bounded guarantee, lending of surplus, fairness | preemption: borrowed jobs must be able to resume (checkpoint) |

### The hybrid model in one sentence {#hybrid}

By default each institution submits to **its own owner QOS**, capped at its
share; those jobs are never preempted. Beyond its share it submits to the
**`borrow` QOS**, uncapped but **preemptible** by the owner QOS. Among waiting
jobs, the order depends on **fair-share**, which penalises the institution that
consumed more than its share over the last two weeks.

```text
VS Code ──HTTPS──> gateway (keys, teams, quotas, accounting) ──internal──> vLLM
(Continue,                                                              (Slurm jobs,
 Remote-SSH)                                                             QOS service)
    └──SSH──> login node (sbatch, srun) ──> slurmctld + slurmdbd ──> gpu01 ... gpu04
                                            (Fair Tree, QOS, preemption)   4 x 8 B300
```
