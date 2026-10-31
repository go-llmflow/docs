# docs

Documentation of [go-llmflow](https://github.com/go-llmflow), built with Hugo and
the [tannevaled/hextra](https://github.com/tannevaled/hextra) fork (a submodule
pinned at the release tag `v0.19.0`).

Published at <https://go-llmflow.github.io/docs/>:

- **Overview** of the organisation.
- **Identity**: JWT, single sign-on, SAML federations, SCIM, key rotation.
- **Audit and logging**: the anchored audit log, request logs, policies.
- **PII recognisers** (`pii` v0.1.0): the kinds recognised, their checksums, the
  measured false positives.
- **Coding agents**: Continue, the stateless Responses API, the Anthropic Messages API.
- **Deployment** (skeleton): replicas, vLLM as Slurm jobs, PostgreSQL.
- **Finance reports** (gateway v0.13.0): attribution by project, lab and user, the reports
  page, CSV/XLSX/PDF exports, reconciliation with Slurm.
- **Security**: the audit and chaos run of 2026-10-26 (gateway v0.9.0).
- **Workload profiles** (gateway v0.18.0): days per week, enforced by hard budgets.
- **Libraries**: `aws` v0.1.1 (SigV4 and event streams).
- **Amazon Bedrock** (gateway v0.23.0): Claude on Bedrock under hard budgets.
- **Decision reports** (gateway v0.23.1): fluids, lending, chargeback, adoption, the monthly report, energy reconciliation, quotas and profiles, FinOps variance, privacy, SMTP migration; soft quotas on the profiles page.
- **Gateway** (v0.23.1): HCL2 configuration, hierarchy and roles, admin API, admin web UI, MCP gateway,
  fair-share with borrowing, prefix-cache isolation, overhead A/B against LiteLLM,
  guardrails, limitations, roadmap.
- **Guide: fair-share on a co-owned GPU cluster**: Slurm, vLLM, the gateway, VS Code
  access, security audit, acceptance tests, governance and bibliography.

Each version of the documentation is a separate build under `docs/<version>/`
on the `gh-pages` branch; `main` is the version while no release is tagged.

```sh
git clone --recurse-submodules https://github.com/go-llmflow/docs
hugo server
```

Build as CI does: `hugo --gc --minify --panicOnWarning --printPathWarnings`.
