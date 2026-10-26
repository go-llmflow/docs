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
- **Security**: the audit and chaos run of 2026-10-26 (gateway v0.9.0).
- **Gateway** (v0.11.0): HCL2 configuration, hierarchy and roles, admin API, admin web UI, MCP gateway,
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
