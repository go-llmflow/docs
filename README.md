# docs

Documentation of [go-llmflow](https://github.com/go-llmflow), built with Hugo and
the [tannevaled/hextra](https://github.com/tannevaled/hextra) fork (a submodule
pinned at the release tag `v0.19.0`).

Published at <https://go-llmflow.github.io/docs/>:

- **Overview** of the organisation.
- **Identity**: JWT, single sign-on, SAML federations, SCIM, key rotation.
- **Audit and logging**: the anchored audit log, request logs, policies.
- **Gateway** (v0.5.0): HCL2 configuration, hierarchy and roles, admin API,
  fair-share with borrowing, prefix-cache isolation, overhead A/B against LiteLLM,
  limitations, roadmap.
- **Guide: fair-share on a co-owned GPU cluster**: Slurm, vLLM, the gateway, VS Code
  access, security audit, acceptance tests, governance and bibliography.

Each version of the documentation is a separate build under `docs/<version>/`
on the `gh-pages` branch; `main` is the version while no release is tagged.

```sh
git clone --recurse-submodules https://github.com/go-llmflow/docs
hugo server
```

Build as CI does: `hugo --gc --minify --panicOnWarning --printPathWarnings`.
