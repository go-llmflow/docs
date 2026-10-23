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
| [Gateway]({{< relref "gateway.md" >}}) | the `gateway` project: an OpenAI-compatible gateway in front of vLLM. Its status, its architecture, how fair-share with borrowing works, and its security notes |
| [Fair-share on a co-owned GPU cluster]({{< relref "guides/fairshare/_index.md" >}}) | a guide: how two institutions can share a cluster of GPU servers with Slurm (batch), vLLM (inference) and a gateway (keys, quotas, priorities), reached from VS Code |

## Status {#status}

The organisation has one repository with code, [`gateway`](https://github.com/go-llmflow/gateway),
and it is **in development: v0.1.0 is not released yet**. These pages say, for
each part of it, whether it is merged, open, or only designed. Nothing here is
described as working that is not on the `main` branch of the repository.

The guide is different in kind: it is a design and operations document for a
cluster, written from the vendors' documentation and from measurements, and it
stands on its own. The gateway is the part of it that is being written in Go.

## Links {#links}

- Source: [github.com/go-llmflow](https://github.com/go-llmflow)
- Landing page: [go-llmflow.github.io](https://go-llmflow.github.io/)
