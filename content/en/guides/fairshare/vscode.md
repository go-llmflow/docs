---
title: "Access from VS Code"
weight: 50
---

# Access from VS Code {#vscode}

Two uses, two paths: the **code assistant** talks to the gateway over HTTPS;
**development and job submission** go over SSH to the login node.

## Code assistant: the Continue extension {#continue}

Continue accepts any OpenAI-compatible provider, LiteLLM and vLLM included:
`provider: openai` with a custom `apiBase` [16]. The configuration file
(`~/.continue/config.yaml` on macOS and Linux, `%USERPROFILE%\.continue\config.yaml`
on Windows; [Continue configuration](https://docs.continue.dev/customize/deep-dives/configuration)) requires the fields `name`, `version` and
`schema`. Each model declares `roles` among `chat`, `autocomplete`, `embed`,
`rerank`, `edit`, `apply` and `summarize` [17].

```yaml
name: B300 platform
version: 1.0.0
schema: v1

models:
  - name: Large model (B300)
    provider: openai
    model: large-model               # model_name exposed by the gateway
    apiBase: https://llm.example.org/v1
    apiKey: ${{ secrets.B300_GATEWAY_KEY }}
    roles: [chat, edit, apply, summarize]
    defaultCompletionOptions:
      contextLength: 131072
      maxTokens: 4096

  - name: Fast completion (B300)
    provider: openai
    model: small-code-model
    apiBase: https://llm.example.org/v1
    apiKey: ${{ secrets.B300_GATEWAY_KEY }}
    roles: [autocomplete]
```

- **The key** is the user's virtual key, attached to the team of their institution:
  their consumption lands in the right share. It is referenced as a secret
  (`${{ secrets.... }}` [17]), never written in clear in a versioned file.
- **Autocompletion** sends a request on every keystroke. Serve it with a *small*
  dedicated model (an instance of a few GPUs, or MIG) and set an `rpm_limit` on the
  key, or it will eat the institution's quota.
- `contextLength` must not exceed the vLLM instance's `--max-model-len`.

## Code assistant: Claude Code {#claude-code}

Claude Code can also talk to the gateway, through `ANTHROPIC_BASE_URL`, with the
virtual key as the credential. The full, sourced instructions are on the
[Coding agents]({{< relref "agents.md" >}}#claude-code) page; the short version:

- **Models.** Anthropic does not support routing Claude Code to non-Claude models
  through any gateway, so Claude Code against the local vLLM models of this cluster is
  possible but **unsupported by Anthropic**. The supported path is Claude models through
  the gateway's opt-in `anthropic` provider (gateway v0.12.0), billed to the
  organisation's API account. Pooling Pro or Max subscriptions is not allowed.
- **Credential.** Use `apiKeyHelper` in `~/.claude/settings.json`, a command that prints
  the key from a file of mode 0600. Never put the key in the project's committed
  `.claude/settings.json`.
- **VS Code.** The documented setup puts the gateway variables, key included, in
  `claudeCode.environmentVariables` in VS Code's user settings: keep that file out of
  any synced or committed location. A setup with the key file only, using
  `claudeCode.disableLoginPrompt`, is not verified against a gateway.
- **Project.** From gateway v0.13.0 (in progress), a project can be named with an
  `X-Llmflow-Project` header, set in the repository's `.claude/settings.json` as
  `ANTHROPIC_CUSTOM_HEADERS`, or with `requestOptions.headers` in Continue.

## Development and jobs: Remote-SSH {#remote-ssh}

Remote-SSH reads the user's `~/.ssh/config` [18]. Target the **login node**, never
a GPU node: GPU nodes are reached only through a Slurm job, otherwise the work
escapes fair-share and cgroup isolation.

```text
# ~/.ssh/config
Host b300-bastion
    HostName bastion.example.org
    User alice
    IdentityFile ~/.ssh/id_ed25519_b300

Host b300-login
    HostName login.b300.internal
    User alice
    IdentityFile ~/.ssh/id_ed25519_b300
    ProxyJump b300-bastion
```

```json
// settings.json (VS Code, user)
{
  "remote.SSH.remotePlatform": { "b300-login": "linux" }
}
```

`remote.SSH.remotePlatform` maps a host to its platform [18]. The login node is
shared by many users, so also enable **Remote.SSH: Remote Server Listen On
Socket** in the Settings UI (the page cited names it only by that label). The
VS Code server then listens on a Unix socket locked to the user instead of a
forwarded TCP port. This needs a Linux or macOS host, disables connection
multiplexing, and makes public-key authentication recommended
([Remote Development troubleshooting](https://code.visualstudio.com/docs/remote/troubleshooting)).

{{< callout type="warning" >}}
The VS Code server consumes CPU and memory on the login node. Set per-user limits
on that node (systemd cgroups) so that one session does not penalise the others.
Do not run computation on the login node: go through `srun`/`sbatch`.
{{< /callout >}}
