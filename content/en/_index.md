---
title: "Overview"
weight: 10
---

# go-llmflow documentation {#go-llmflow-documentation}

**go-llmflow** is high-performance LLM inference and serving in Go. It is
written for the case where one GPU cluster has several owners: each owner must
get the share it paid for, and a share that its owner leaves idle must not be
wasted.

Everything is **pure Go** (`CGO_ENABLED=0`) and BSD-3-Clause.

## What is here {#what-is-here}

| Page | What it says |
| --- | --- |
| [Gateway]({{< relref "gateway.md" >}}) | the `gateway` project (v0.16.1): an OpenAI-compatible gateway in front of vLLM. Its configuration, hierarchy and roles, admin API, how fair-share with borrowing works, prefix-cache isolation, overhead measurement, limitations and roadmap |
| [Identity]({{< relref "identity.md" >}}) | how the gateway takes people from your identity provider: OpenID Connect tokens, administrator single sign-on, SAML federations through a bridge, SCIM provisioning, key rotation |
| [Coding agents]({{< relref "agents.md" >}}) | using the gateway from VS Code and from agents that speak the OpenAI Responses or the Anthropic Messages API, and why Responses are stateless |
| [Audit and logging]({{< relref "audit.md" >}}) | the hash-chained admin audit log, its anchors and what they do not protect against, per-team request logs, retention, request policies |
| [Security]({{< relref "security.md" >}}) | the audit and chaos run of 2026-10-26: method, findings A1 to A9 and their fixes, chaos results and residual risks |
| [Deployment]({{< relref "deployment.md" >}}) | a skeleton: how the gateway, its replicas, vLLM as Slurm jobs and PostgreSQL fit together, and which page says what |
| [Finance reports]({{< relref "finance-reports.md" >}}) | what a finance director gets since gateway v0.13.0: project, lab and user attribution, the `/ui/reports` page, CSV, XLSX and PDF exports, reconciliation with Slurm's `sreport`, and the known gaps |
| [PII recognisers]({{< relref "pii.md" >}}) | the `pii` library (v0.1.0): e-mail, phone, IBAN, French NIR, payment cards and IP addresses found in text, checksum-validated, with the false positives measured |
| [Benchmarking]({{< relref "benchmarking.md" >}}) | the `bench` tools (v0.6.0): how `gwbench` measures the latency a gateway adds (interleaved, paired per round, load-gated, with a planted-delay control), the recording fake vLLM it measures against, and the checks of the gateway against a real vLLM |
| [Fair-share on a co-owned GPU cluster]({{< relref "guides/fairshare/_index.md" >}}) | a guide: how two institutions can share a cluster of GPU servers with Slurm (batch), vLLM (inference) and a gateway (keys, quotas, priorities), reached from VS Code |

## Status {#status}

The organisation has three repositories with code (the third, [`bench`](https://github.com/go-llmflow/bench), v0.4.0, holds the measuring tools described in [Benchmarking]({{< relref "benchmarking.md" >}})). [`gateway`](https://github.com/go-llmflow/gateway)
has its **current release at v0.16.1**: virtual keys, budgets on every scope of an
organisation, team, project and key hierarchy with delegated roles, HCL2
configuration, an admin API, fair-share with borrowing, per-tenant prefix-cache
isolation, identity (OpenID Connect, single sign-on, SCIM), an anchored audit log,
and guardrails (`hide_secrets`, `pii`, Llama Guard, Prompt Guard; the HTTP adapter
is off by default). [`pii`](https://github.com/go-llmflow/pii) is a library, at
v0.1.0, that recognises personal data in text; the gateway's PII guardrail uses it. The
[Gateway]({{< relref "gateway.md" >}}) page lists what it does, how it is
configured, what was measured, and its open limitations and roadmap.

The guide is different in kind: it is a design and operations document for a
cluster, written from the vendors' documentation and from measurements, and it
stands on its own. The gateway is the part of it that is being written in Go.

## Links {#links}

- Source: [github.com/go-llmflow](https://github.com/go-llmflow)
- Landing page: [go-llmflow.github.io](https://go-llmflow.github.io/)
