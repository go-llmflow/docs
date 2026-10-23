---
title: "Slurm: fair-share, QOS and preemption"
weight: 20
---

# Slurm {#slurm}

Slurm carries the batch workload and the guarantee. Reference numbers are
`[n]` entries of the [bibliography]({{< relref "bibliography.md" >}}).

## Fair-share principles {#principles}

### Multifactor priority {#multifactor}

With `PriorityType=priority/multifactor`, a job's priority is an integer: a site
factor, plus the sum of each factor (age, association, fair-share, size,
partition, QOS, TRES) times its weight, minus the *nice* value. Each factor is
between 0 and 1 and weights default to 0, so only the configured factors count
[2].

### The Fair Tree algorithm {#fair-tree}

Fair Tree has been Slurm's default since 19.05 [2]. At each level of the
association tree it computes the *Level Fairshare* [1]:

```text
LF = S / U     S = S_self / S_(self+siblings)     U = U_self / U_(self+siblings)
```

`LF > 1` means under-served; `LF < 1`, over-served; `LF = infinity` with no
consumption. Siblings are sorted by decreasing `LF`, the tree is walked depth
first, and each user receives a rank. The user's fair-share factor is
*rank / number of users*: 1.0 for the first [1].

**The decisive property for us**: the first level compares **A against B** before
looking at users. If A is over-served, *all* of B's users come before *all* of
A's, whatever individual consumption is. This is exactly the wanted behaviour
between two institutions.

**A worked example.** Over the last 14 days (with decay), A consumed 70 % of the
billed GPU-hours and B 30 %:

```text
LF_A = 0.8125 / 0.70 = 1.16        LF_B = 0.1875 / 0.30 = 0.625
```

A is under-served relative to its share, so it goes first: its users get the
highest ranks. B's borrowing jobs will wait until A's usage is back to normal.
B's jobs in its owner QOS remain guaranteed (see [QOS](#qos)).

### Measuring consumption: GPU billing {#billing}

By default consumption is measured in CPU x seconds [2]. On a GPU cluster that
is wrong: a job taking 1 core and 8 GPUs would cost almost nothing. So
`TRESBillingWeights` is set on the partition [2], with
`PriorityFlags=MAX_TRES`. A job's billing then becomes *the per-node maximum* of
the weighted TRES (CPU, memory, GRES) [2], not their sum.

The weights are calibrated so that **a whole server is worth 8 units**, whichever
resource saturates it. For a server of `C` cores and `M` GiB of allocatable
memory:

```text
w_GPU = 1        w_CPU = 8 / C        w_Mem = 8 / M
```

With C = 128 and M = 3072 GiB: `CPU=0.0625,Mem=0.0026G,GRES/gpu=1.0`. A job
taking 1 GPU but 64 cores is billed 4 units, because it makes 4 GPUs unusable
for others. The unit of account becomes the **GPU-equivalent second**.

{{< callout type="info" >}}
**To validate**: replace `C` and `M` by the values of `slurmd -C` [5] on your
servers (`CPUs` and `RealMemory` minus the memory reserved for the system).
{{< /callout >}}

### Decay of consumption {#decay}

`PriorityDecayHalfLife` is 7 days by default and applies every
`PriorityCalcPeriod` (5 minutes by default) [2]. We recommend **14 days**:
training jobs on B300 often last several days, and a too-short half-life would
erase a consumption debt before the other institution could catch up.
`PriorityUsageResetPeriod` stays `NONE`: an abrupt reset creates edge effects at
the start of a period.

## Guarantee by QOS and preemption {#qos}

### The QOS {#the-qos}

| QOS | Cap | Role |
| --- | --- | --- |
| `instA` | `GrpTRES=gres/gpu=G_A` | A's share, never preempted |
| `instB` | `GrpTRES=gres/gpu=G_B` | B's share, never preempted |
| `service` | `GrpTRES=gres/gpu=G_S` | shared vLLM instances, never preempted |
| `borrow` | none | surplus; preemptible by the three QOS above; ordered by fair-share |

`GrpTRES` limits the total TRES allocated to all running jobs of the QOS [4].
Since `G_A + G_B + G_S = 32`, the sum of the guaranteed shares never exceeds the
cluster. The GPUs an owner does not occupy are either free or held by `borrow`
jobs, therefore recoverable by preemption.

### Splitting between batch and inference {#split}

The inference service is a common good: its GPUs are taken from each share in
proportion to ownership. With an inference pool of **16 GPUs (2 servers, 2 TP=8
instances)**:

| | Total share | of which inference (pro rata) | Guaranteed batch |
| --- | --- | --- | --- |
| A | 26 | 13 | `G_A = 13` |
| B | 6 | 3 | `G_B = 3` |
| Service | -- | -- | `G_S = 16` |

With a pool of 8 GPUs the pro rata gives 6.5 and 1.5, so it is rounded to
`G_A = 19` and `G_B = 5`: fair-share, which counts continuously, rebalances the
half GPU over time. The inference pool can change size with the time of day
([elastic pool]({{< relref "inference.md" >}}#elastic-pool)); the GPUs released
go to `borrow` jobs.

### B's big jobs: a weekly reservation {#weekly-reservation}

B's guaranteed share (`G_B` = 3 or 5) is below 8. Without a special measure, B
can run a job on a whole server (TP=8, multi-GPU training) only in the `borrow`
QOS, hence with no guaranteed delay. B is given a **recurring reservation** of
one whole server.

```bash
# A 48 h window every week, starting Monday 8:00, for institution B
scontrol create reservation ReservationName=instB_weekly \
    StartTime=2026-11-02T08:00:00 Duration=2-00:00:00 Flags=WEEKLY \
    Accounts=instB PartitionName=gpu NodeCnt=1

# On B's side, during the window
sbatch -A b_lab --reservation=instB_weekly --qos=borrow \
       --gres=gpu:b300:8 --time=2-00:00:00 training.sh
```

Slurm's behaviour, from the reservation documentation [21]:

- `WEEKLY` makes the reservation recurring: it activates and deactivates itself
  every week.
- During the window, the reserved resources serve *only* the reservation's jobs:
  "no other jobs can use those resources".
- **Unused reserved time is charged** to the reservation's users or accounts, in
  equal parts. Here all of it goes to `instB`. B therefore pays for its window in
  fair-share whether it uses it or not. That is fair (the server was taken from
  the others) and it encourages reserving only what will be used.
- B submits in the `borrow` QOS, not in `instB` whose 3-GPU cap would block an
  8-GPU job. The preemption risk is nil: no owner job of A can enter the
  reserved server.

**Sizing.** B's weekly share is `6 x 168 = 1,008` GPU-hours. A 48 h window costs
`8 x 48 = 384` GPU-hours, 38 % of its share. The rest is consumed in `instB` and
`borrow`. The agreement fixes the duration; a 126 h window would exhaust all of
B's share in a whole server.

{{< callout type="warning" >}}
Before each window, Slurm must start on the chosen server only jobs that finish
before it opens. `borrow` jobs that declare a short `--time` then keep filling
that server until the last minute. This behaviour is verified in the
[acceptance tests]({{< relref "audit-and-acceptance.md" >}}#acceptance) (test
9). For B, the job must also fit the window: `--time` less than or equal to the
remaining duration.
{{< /callout >}}

### Preemption {#preemption}

- `PreemptType=preempt/qos`: the rules come from the QOS in the database [3].
- `PreemptMode=REQUEUE`: the preempted job is put back in the queue. This
  requires `JobRequeue=1` or `sbatch --requeue` [3]. `SUSPEND` is rejected: a
  suspended job keeps its memory, hence its 288 GB of HBM per GPU, and the owner
  QOS would recover nothing [3].
- `PreemptExemptTime`: minimum run time before being preemptible [3]. We choose
  **30 min**, so that a borrowed job makes at least some progress.
- `GraceTime` on the `borrow` QOS: the delay between SIGTERM and SIGKILL [3],
  used to write a *checkpoint*. We choose **300 s**.

**Guaranteed delay.** An owner job that fits in its QOS starts after at most

```text
PreemptExemptTime + GraceTime + one scheduling cycle = 30 + 5 + 1 = 36 minutes
```

and in practice much sooner, since most borrowed jobs have already passed their
first 30 minutes. This figure goes into the agreement between institutions and
is measured in the acceptance tests.

## Reference configuration {#configuration}

### `slurm.conf` (excerpts) {#slurm-conf}

```ini
ClusterName=b300
SelectType=select/cons_tres
SelectTypeParameters=CR_Core_Memory
GresTypes=gpu
NodeName=gpu[01-04] Gres=gpu:b300:8 CPUs=<C> RealMemory=<M_MiB> State=UNKNOWN
PartitionName=gpu Nodes=gpu[01-04] Default=YES MaxTime=3-00:00:00 State=UP \
    TRESBillingWeights="CPU=0.0625,Mem=0.0026G,GRES/gpu=1.0"

# Accounting and enforcement of limits
AccountingStorageType=accounting_storage/slurmdbd
AccountingStorageTRES=gres/gpu,gres/gpu:b300
AccountingStorageEnforce=associations,limits,qos,safe

# Priority
PriorityType=priority/multifactor
PriorityFlags=MAX_TRES
PriorityDecayHalfLife=14-0
PriorityCalcPeriod=5
PriorityMaxAge=7-0
PriorityWeightFairshare=100000
PriorityWeightQOS=20000
PriorityWeightAge=5000
PriorityWeightJobSize=0
PriorityWeightPartition=0

# Preemption
PreemptType=preempt/qos
PreemptMode=REQUEUE
PreemptExemptTime=00:30:00
JobRequeue=1

# Isolation
ProctrackType=proctrack/cgroup
TaskPlugin=task/cgroup,task/affinity
```

Reading notes:

- `select/cons_tres` is required for sharing GRES (MPS, *shard*) [5] and for the
  preemption of heterogeneous jobs [3].
- `AccountingStorageTRES`: declare `gres/gpu` *and* its typed variant, as the
  documentation recommends [6]. Declaring `gres/gpu` also enables `gres/gpumem`
  and `gres/gpuutil` [6].
- `AccountingStorageEnforce=...,safe`: a job starts only if it can finish within
  its limits in TRES-minutes [4].
- The fair-share weight dominates (100,000); the QOS weight (20,000) breaks ties;
  age (5,000) prevents starvation of an old job.

### `gres.conf` and `cgroup.conf` {#gres-cgroup}

```ini
# gres.conf -- automatic detection through NVML
AutoDetect=nvml

# cgroup.conf
ConstrainCores=yes
ConstrainDevices=yes
ConstrainRAMSpace=yes
```

With `AutoDetect=nvml`, Slurm fills in the details of detected GPUs. The `Gres=`
line of `slurm.conf` stays mandatory, and a node exposing fewer GPUs than
announced goes to DRAIN [5].

`ConstrainDevices=yes` limits a job to the devices of its GRES allocation [7].
Without it a job could open another institution's GPUs.

**MIG (optional).** For small inference or TP jobs, the B300s can be cut into
MIG instances. Slurm manages them with `AutoDetect=nvml` on the nodes
concerned, each instance typed by its profile [5]. *Shards*, on the other hand,
do not partition processes on the GPU [5]: reserve them for uses inside one
institution.

### Accounts and QOS (`sacctmgr`) {#sacctmgr}

```bash
# Institutions: shares expressed in owned GPUs
sacctmgr add account instA Description="Institution A" Fairshare=26
sacctmgr add account instB Description="Institution B" Fairshare=6
sacctmgr add account svc_llm Description="Inference service" Fairshare=1 Parent=root

# Teams (example) under each institution
sacctmgr add account a_physics Parent=instA Fairshare=60
sacctmgr add account a_biology Parent=instA Fairshare=40
sacctmgr add account b_lab     Parent=instB Fairshare=parent

# QOS
sacctmgr add qos borrow
sacctmgr modify qos borrow set Priority=0 PreemptMode=REQUEUE GraceTime=300
sacctmgr add qos instA
sacctmgr modify qos instA set Priority=100 GrpTRES=gres/gpu=13 \
    Preempt=borrow Flags=DenyOnLimit
sacctmgr add qos instB
sacctmgr modify qos instB set Priority=100 GrpTRES=gres/gpu=3 \
    Preempt=borrow Flags=DenyOnLimit
sacctmgr add qos service
sacctmgr modify qos service set Priority=1000 GrpTRES=gres/gpu=16 Preempt=borrow

# Attachment: owner QOS by default + the right to borrow
sacctmgr modify account instA set QOS=instA,borrow DefaultQOS=instA
sacctmgr modify account instB set QOS=instB,borrow DefaultQOS=instB
sacctmgr modify account svc_llm set QOS=service DefaultQOS=service

# Users
sacctmgr add user alice Account=a_physics
sacctmgr add user bob   Account=b_lab
```

- `Fairshare=parent` on an account attaches its children to the parent for the
  computation [8]: B, which has a single laboratory, adds no useless level.
- `DenyOnLimit` rejects at submission a job that alone would exceed the QOS
  limits [8]. An 8-GPU job submitted in `instB` is refused immediately instead of
  waiting forever; the user is invited to use `borrow`.
- `svc_llm` receives a symbolic share: its guarantee comes from the `service`
  QOS, not from fair-share. Its consumption is re-charged to the institutions
  (see [accounting]({{< relref "accounting.md" >}})).

### User side {#user-side}

```bash
# Within one's guaranteed share (default QOS)
sbatch -A a_physics --gres=gpu:b300:4 train.sh

# Beyond one's share: borrowing, preemptible, must be able to resume
sbatch -A a_physics --qos=borrow --requeue --gres=gpu:b300:8 \
       --signal=B:TERM@300 train_checkpoint.sh
```

The script of a `borrow` job traps SIGTERM, writes a checkpoint within the 300 s
of `GraceTime`, and restarts from the last checkpoint when requeued.

### Monitoring {#monitoring}

| Command | What it shows |
| --- | --- |
| `sshare -l` | each association's Level FS [1]: are A and B over- or under-served? |
| `sprio -l` | decomposition of the priority of waiting jobs by factor |
| `sreport cluster AccountUtilizationByUser -T gres/gpu` | GPU-hours per account over a period |
| `scontrol show job <id>` | the `PreemptEligibleTime` [3] of a borrowed job |
