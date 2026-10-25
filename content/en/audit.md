---
title: "Audit and logging"
weight: 27
---

# Audit and logging {#audit}

**v0.5.0** of the [gateway]({{< relref "gateway.md" >}}) (roadmap milestone M3)
answers three questions: who changed what in the administration, can that record
be trusted, and what may be kept about the requests themselves. This page is
written from the gateway's README at that tag; where the guarantees end is stated
as plainly as where they hold.

## A configuration {#configuration}

This was run through `llmflow-gateway check-config` v0.8.0, with the key files
present and a configuration (organisations, a pool and a model) around it.

```hcl
store {
  kind     = "postgres"
  dsn_file = "/etc/llmflow/pg.dsn"
}

audit {
  signing_key_file    = "/etc/llmflow/audit.seed"
  checkpoint_interval = "5m"
  anchor              = "archive"
  anchor_max_lag      = "15m"
  retention           = { admin = "61320h", auth = "8760h", scim = "8760h", system = "8760h" }
  prune_interval      = "24h"
}

export "archive" {
  kind            = "s3"
  endpoint        = "https://minio.example.org:9000"
  bucket          = "llmflow-audit"
  path_style      = true
  access_key_file = "/etc/llmflow/s3.access"
  secret_key_file = "/etc/llmflow/s3.secret"
  prefix          = "gateway-1/"
  lock_mode       = "COMPLIANCE"
  retention       = "8760h"
  queue_size      = 1024
}

logging "research" {
  teams             = ["lab-nlp", "lab-chem"]
  destination       = "archive"
  content           = false
  max_content_bytes = 1048576
  batch_records     = 500
  batch_interval    = "1m"
  lock              = false
}

policy "attributed" {
  teams        = ["lab-nlp"]
  require      = ["user", "metadata.project"]
  require_tags = true
}
policy "long-context" {
  max_body_bytes     = 33554432
  max_response_bytes = 134217728
}
```

The `signing_key_file` is an Ed25519 seed, made with
`llmflow-gateway audit keygen -o FILE`. Like every `*_file` attribute it may be a
`vault:` reference instead of a path.

## The admin audit log {#log}

Every state-changing admin request is recorded, accepted or refused: the actor and
their roles, the action, the target, the state **before** and **after**, the request
body, the status, the client address and a request id. Events the gateway raises
itself go to the same log: SSO logins and logouts, key rotations, SCIM
reconciliation. Key secrets and tokens never appear, and reads are not audited.

The log is append-only in PostgreSQL (a trigger refuses `UPDATE`) and
**hash-chained**: each entry stores a SHA-256 of the previous hash and itself, so
altering, inserting, reordering or deleting an entry breaks every hash after it.
With a signing key, **checkpoints** sign the newest hash at an interval and at
shutdown.

### Anchors {#anchors}

Every checkpoint, and the entries it covers, is also written to the `anchor` export
(`audit/checkpoints/<seq>.json`, `audit/entries/<from>-<to>.jsonl.gz`). On S3 the
objects go under **Object Lock** retention when the bucket has it enabled; create
the bucket that way, since MinIO and Ceph RGW require it at creation, with
versioning. A `file` export refuses to overwrite a file but does not stop its
owner. The age of the newest exported checkpoint is the metric
`llmflow_audit_anchor_age_seconds`, and a checkpoint still unexported after
`anchor_max_lag` raises the alert `audit_anchor_lag`.

### Verifying {#verifying}

```sh
llmflow-gateway audit keygen -o audit.seed       # once; keep the seed in Vault
llmflow-gateway audit pubkey -c gateway.hcl      # the public key, to keep elsewhere
llmflow-gateway audit verify -c gateway.hcl --public-key KEY --anchor export
llmflow-gateway audit verify -c gateway.hcl --public-key KEY --anchor file:PATH
```

`verify` recomputes the chain, checks every checkpoint and, with `--anchor`,
checks the database against the newest checkpoint of the export (or of a file you
kept yourself). Take the public key from somewhere the gateway's operator cannot
change. **Run without an anchor, `verify` prints a warning**, for the reason below.

### What this protects against {#threat-model}

| | Without an anchor | With an anchor kept outside |
| --- | --- | --- |
| an entry edited, inserted, reordered or deleted in the middle | detected | detected |
| the tail deleted together with the checkpoints that cover it | **not detected** | detected |
| an older consistent backup restored (rollback) | **not detected** | detected |
| everything deleted | **not detected** | detected |
| a log rewritten and re-signed with a stolen key, up to the anchor | not detected | detected |
| entries after the newest exported checkpoint deleted | not detected | **still open**: at most `checkpoint_interval` plus the export lag |
| a new history signed after the anchor by the holder of the signing key | not detected | **still open** |

Entries and checkpoints live in the same database, which is why an anchor outside
it is needed. To narrow what stays open, keep the signing key in OpenBao or Vault,
verify with a public key kept elsewhere, and anchor on write-once storage.

### Retention {#retention}

Entries are kept for ever unless a retention is set per kind (`admin`, `auth` for
SSO, `scim`, `system`). Every `prune_interval` the gateway deletes the longest
prefix of the log that ends at a signed checkpoint and whose every entry is past its
kind's retention: whole segments only, so what remains starts from a signed
checkpoint and still verifies. A kind without a retention stops the cut. It never
prunes past the newest checkpoint the anchor holds, and never a log that does not
verify (`audit_prune_refused` is raised instead). Each prune is itself an
`audit.pruned` entry.

## Per-team request logs {#request-logs}

By default **no request log is written for anyone**: the spend metadata goes to the
accounting store; prompts and completions are forwarded and forgotten, and the
gateway's own log never holds them.

A `logging` block sends the requests of the teams it names to an `export`. Each
request is one JSON line (time, org, team, project, key id, model, pool, upstream,
tokens, cost, latency, status, priority, borrowing, stream, estimated, tags, request
id). **Content is off**: only a block with `content = true`, an explicit opt-in,
adds the request and the answer, each cut at `max_content_bytes` and marked
`request_truncated` or `response_truncated`. A team is in at most one block. Lines
are batched into gzipped objects under `requests/<block>/YYYY/MM/DD/`.

## Export targets {#exports}

`kind` is `s3` or `file`. `gcs` and `azure` answer "not implemented". The S3 client
is [go-volumes/s3](https://github.com/go-volumes/s3). Exports go through a bounded
queue; a batch dropped or given up is counted and alerted (`export_dropped`,
`export_failed`), never silent.

## Request policies {#policies}

A `policy` refuses inference requests that lack given fields, and replaces the
global size limits, for the teams it names and for keys created with it
(`POST /admin/keys {"policy": "..."}`, by a team admin; a key's own policy wins over
its team's).

- `require` lists body fields (a name or a dotted path); `require_tags = true` asks
  for at least one request tag. A refused request gets a **400 naming what is
  missing**, and counts in `llmflow_denied_total{reason="policy"}`.
- `max_body_bytes` and `max_response_bytes` replace the global limits. The response
  limit is **not applied to streams**.

## Alerts {#alerts}

Soft-budget alerts are retried with exponential backoff, up to 6 attempts, and
every webhook attempt carries the same `Idempotency-Key` so that a receiver can
de-duplicate. Retries live in memory: a gateway stopped mid-retry does not resume.
