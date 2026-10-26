---
title: "Coding agents"
weight: 28
---

# Coding agents {#agents}

Since **v0.7.0** the [gateway]({{< relref "gateway.md" >}}) serves the three
APIs coding tools speak, in front of the cluster's own models. Every request
authenticates with a **virtual key** as the bearer token and goes through the same
budgets, guardrails and fair-share priority, so an agent's consumption lands in its
team's share.

| Tool speaks | Route | Notes |
| --- | --- | --- |
| OpenAI chat completions | `POST /v1/chat/completions` | the default for editors such as VS Code with Continue |
| OpenAI Responses | `POST /v1/responses` | **stateless only**: no `previous_response_id`, no stored responses |
| Anthropic Messages | `POST /v1/messages` | vLLM's native Anthropic endpoint, no translation in the gateway |

## VS Code with Continue {#continue}

Continue is configured for the gateway exactly as for any OpenAI-compatible
server; the file and its keys are on the
[VS Code page]({{< relref "guides/fairshare/vscode.md" >}}), taken from Continue's
own documentation. Point `apiBase` at the gateway's `/v1` and use the user's
virtual key. Whether a given Continue version can call `/v1/responses` is a
question for Continue's documentation, which this site has not opened for that
route: the safe choice is `/v1/chat/completions`.

A request through `/v1/chat/completions` that the gateway accepts, whatever the
client:

```sh
curl https://llm.example.org/v1/chat/completions \
  -H "Authorization: Bearer $LLMFLOW_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model": "llama-3.3-70b",
       "messages": [{"role": "user", "content": "Hello"}]}'
```

## The Responses API, and why it is stateless {#responses}

A client that uses `/v1/responses` must send the whole conversation in `input` each
time. The gateway forces `"store": false` and answers 400 to `previous_response_id`,
`background`, `conversation` and a stored `prompt`:

```sh
curl https://llm.example.org/v1/responses \
  -H "Authorization: Bearer $LLMFLOW_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model": "llama-3.3-70b", "input": "Hello"}'
```

**Why.** vLLM can keep responses in the memory of the instance that produced them
(`VLLM_ENABLE_RESPONSES_API_STORE=1`), readable and continuable by anyone who holds
the id. The gateway forces `store: false` and refuses the stateful fields
regardless, so a deployment with the store enabled stays safe: on a shared cluster
no tenant can read or continue another's conversation. `GET /v1/responses/{id}` and the cancel route are not
routed.

## Agents that speak the Anthropic API {#anthropic}

`POST /v1/messages` lets an agent that speaks the Messages API use a local model.
The gateway's README says to point such an agent at the gateway "with a virtual key
as the bearer token". The request shape is the Messages API's:

```sh
curl https://llm.example.org/v1/messages \
  -H "Authorization: Bearer $LLMFLOW_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model": "llama-3.3-70b", "max_tokens": 256,
       "messages": [{"role": "user", "content": "Hello"}]}'
```

How a given agent is told its base URL and its key is that agent's own configuration.
[Claude Code](#claude-code) is documented below, from Anthropic's pages; for any other
agent, read its documentation, which this site has not opened.

What the gateway guarantees on this route: guardrails cover `system`, text and
`thinking` blocks, `tool_result` content and the strings inside `tool_use` inputs;
guarded streams pass through an Anthropic event filter; error bodies carry
`"type": "error"`; `/v1/messages/count_tokens` (since v0.12.1) goes to vLLM's own
endpoint and returns an **exact** count, which Claude Code uses instead of its estimate;
it is authenticated and rate limited, not charged and not guarded. Accounting counts
cached tokens as vLLM reports them.

## Claude Code {#claude-code}

Everything below is from Anthropic's pages
[LLM gateways](https://code.claude.com/docs/en/llm-gateway),
[connect Claude Code to an LLM gateway](https://code.claude.com/docs/en/llm-gateway-connect),
the [gateway compatibility guide](https://code.claude.com/docs/en/llm-gateway-protocol)
and [legal and compliance](https://code.claude.com/docs/en/legal-and-compliance),
read on 2026-10-26. Claude Code versions change; check them.

{{< callout type="warning" >}}
**Which models, said plainly.** Anthropic's page says it "doesn't support routing
Claude Code to non-Claude models through any gateway". Pointing Claude Code at this
gateway's `/v1/messages` and a **local vLLM model** is technically possible, since
the route exists, but it is **unsupported by Anthropic**. The supported path is
**Claude models**: Claude Code, then this gateway, then the Anthropic API through the
opt-in `anthropic` provider (gateway v0.12.0), billed to the organisation's own API
account.
{{< /callout >}}

### The supported path: Claude Code, the gateway, the Anthropic API {#claude-code-anthropic}

With v0.12.0 the gateway can send `/v1/messages` to Anthropic's API instead of to
vLLM. The configuration, with what `check-config` enforces (an explicit allow-list,
`allow_external`, one API key per institution, and `pii` and `hide_secrets`
guardrails), is on the [Gateway page]({{< relref "gateway.md" >}}#anthropic). For Claude
Code it means:

- Point `ANTHROPIC_BASE_URL` at the gateway and use the institution's **llmflow virtual
  key** as the credential (below). The gateway accepts it as `Authorization: Bearer`
  **or `x-api-key`**, so Claude Code's `ANTHROPIC_API_KEY` works too. The gateway
  replaces it with the **organisation's own Anthropic key**; the client's credential and
  the OAuth `anthropic-beta` capability never go out.
- The traffic is **billed to the organisation's Anthropic API account**, per token, in
  USD, and shown as USD in the gateway's budgets and reports, apart from GPU-hours.
- **Not a subscription.** The gateway never relays a Pro or Max login, and pooling
  subscriptions is forbidden (see [Subscriptions](#claude-code-subscriptions), the
  legal and DPO notes there, and the gateway's own statement that this is not legal
  advice).
- `/v1/messages/count_tokens` is routed since gateway v0.12.1, to Anthropic with the
  organisation's key; it is not charged, and the same masking applies, since a count also
  sends the prompt off the site.
- Prompts leave the site. That is why the guardrails are mandatory, and why the
  route needs your DPO's agreement.

### The base URL and the credential {#claude-code-credential}

`ANTHROPIC_BASE_URL` points Claude Code at the gateway; Claude Code then calls
`<base URL>/v1/messages`. The credential comes from one of three places, and each
sends it in a different header:

| Credential | Sent as |
| --- | --- |
| `ANTHROPIC_AUTH_TOKEN` | `Authorization: Bearer` |
| `ANTHROPIC_API_KEY` | `x-api-key` |
| `apiKeyHelper` | both headers |

The gateway reads the virtual key as a bearer token (above), so `ANTHROPIC_AUTH_TOKEN`
or `apiKeyHelper` fits it.

**Recommended: `apiKeyHelper`, reading the key from a file** of mode 0600, so that the
key is never in an environment variable or in a file that is committed. The helper is
a command that prints the credential and nothing else (Claude Code caches its output
for five minutes by default; `CLAUDE_CODE_API_KEY_HELPER_TTL_MS` changes that). A
one-line script:

```bash
#!/bin/bash
cat ~/.config/llmflow/key
```

with `chmod 600 ~/.config/llmflow/key`, referenced from the **user** settings file,
`~/.claude/settings.json` (`%USERPROFILE%\.claude\settings.json` on Windows):

```json
{
  "apiKeyHelper": "~/bin/llmflow-key.sh",
  "env": {
    "ANTHROPIC_BASE_URL": "https://llm.example.org",
    "CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC": "1"
  }
}
```

**Never put a credential in a project's `.claude/settings.json`**: it is committed and
shared with everyone who clones the repository. A per-project credential file is
`.claude/settings.local.json`, which Claude Code adds to your global gitignore when it
saves a setting there; if you create it by hand, add it to your gitignore first.

`CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC=1` keeps background traffic off other hosts
(version checks, telemetry, release notes). It also **disables auto-updates**, so plan
another update path.

For the **VS Code extension**, there are two setups. Anthropic's pages are
[connect to an LLM gateway](https://code.claude.com/docs/en/llm-gateway-connect#vs-code-extension)
and [Claude Code in VS Code](https://code.claude.com/docs/en/vs-code#use-third-party-providers).

**1. The documented, reliable setup.** `ANTHROPIC_BASE_URL` and the credential go in
`claudeCode.environmentVariables`, in VS Code's own user settings (the command
**Preferences: Open User Settings (JSON)**). Anthropic says the extension checks
credentials from this setting before launching, so it is the reliable place for the
credential, and that values in `~/.claude/settings.json` reach the spawned process but
not the extension's own login check:

```json
{
  "claudeCode.environmentVariables": [
    { "name": "ANTHROPIC_BASE_URL", "value": "https://llm.example.org" },
    { "name": "ANTHROPIC_AUTH_TOKEN", "value": "llmf-..." }
  ]
}
```

This **puts the key in VS Code's settings file** as plain text. Keep that file out of
any synced or committed location (Settings Sync, a dotfiles repository), and use a key
meant for this one purpose, so that it can be revoked on its own.

**2. A key file and no key in settings. Not verified.** The VS Code page says, for
third-party providers, to check **Disable Login Prompt** (the setting
`claudeCode.disableLoginPrompt`, "skip authentication prompts (for third-party provider
setups)") and to put the provider configuration in `~/.claude/settings.json`, which
"ensures your settings are shared between the VS Code extension and the CLI". By
analogy, the `apiKeyHelper` setup above, in `~/.claude/settings.json`, with
`claudeCode.disableLoginPrompt` checked, would keep the key in a 0600 file and out of
every settings file. **This page has not verified it against a gateway**, and the
connect page's statement that the extension's login check reads
`claudeCode.environmentVariables` is what makes it uncertain: try it, and confirm with
`/status`, before relying on it.

Run `/status` in Claude Code: an `Anthropic base URL` line, and an `apiKeyHelper` line
naming the credential, confirm that the gateway is in use.

**Pinning an organisation to the gateway.** Managed settings with
`allowedProviders: ["customEndpoint"]` and the gateway's `ANTHROPIC_BASE_URL` in the
file's `env` block make Claude Code refuse any other destination (Claude Code v2.1.285
or later). `CLAUDE_CODE_ENABLE_GATEWAY_MODEL_DISCOVERY=1` adds the gateway's models to
the `/model` picker.

### Subscriptions {#claude-code-subscriptions}

While a gateway credential or an `apiKeyHelper` is active, requests carry that
credential **instead of** a claude.ai subscription login, and the subscription's
limits do not apply; the traffic is billed per token to whoever owns the credential the
gateway forwards. **Pooling Pro or Max subscriptions behind a gateway is not allowed**:
Anthropic's [legal and compliance](https://code.claude.com/docs/en/legal-and-compliance)
page says it does not permit routing requests "through Free, Pro, or Max plan
credentials on behalf of their users", and customers "may not pay for, resell, or
intermediate Claude usage on their end users' behalf". Read that page, and the
consumer terms it cites, before planning any shared use.

### Attributing requests to a project {#claude-code-project}

What each client sends: Claude Code adds `x-claude-code-session-id`,
`x-claude-code-agent-id`, `x-claude-code-prompt-id` and `x-claude-code-request-class`
(among others), **but nothing that names a project**; Continue sends only the headers
you configure.

**Available from gateway v0.13.0.** The gateway reads the project from a header,
`X-Llmflow-Project` by default (the name is `attribution { project_header }`, and
`default_project` names the project used without the header; see
[Project attribution]({{< relref "gateway.md" >}}#attribution)). The value is a project
name looked up **in the key's own team**; a project that does not exist, is disabled,
belongs to another team, or is not allowed to the key's owner gets the same generic
403, "project not allowed for this key". The role must be stored in the gateway (or
come through SCIM): a role that exists only in an SSO token does not count.

The recommended setup keeps the secret and the project apart. The credential is
personal, in the user's settings (`apiKeyHelper` above). The project is not a secret:
it goes in the repository's committed `.claude/settings.json`:

```json
{
  "env": {
    "ANTHROPIC_CUSTOM_HEADERS": "X-Llmflow-Project: my-project"
  }
}
```

`ANTHROPIC_CUSTOM_HEADERS` takes one `Name: Value` pair per line, with `\n` between
pairs in a JSON string. Anthropic notes that headers like these count as
[headers that need approval](https://code.claude.com/docs/en/server-managed-settings.md#environment-variables-and-the-approval-dialog),
and that when they come from a project settings file Claude Code applies them under
[the rules for when it applies `env` values](https://code.claude.com/docs/en/settings-reference#when-claude-code-applies-env-values):
the user is asked to approve them.

For Continue, the header is set per model entry, with `requestOptions.headers`
([Continue's reference](https://docs.continue.dev/reference)):

```yaml
models:
  - name: Large model (B300)
    provider: openai
    model: large-model
    apiBase: https://llm.example.org/v1
    apiKey: ${{ secrets.B300_GATEWAY_KEY }}
    requestOptions:
      headers:
        X-Llmflow-Project: my-project
```

## Limits to keep in mind {#limits}

- A Responses **stream** under **post-call** guardrails is refused: ask with
  `"stream": false`.
- A guarded stream that carries tool-call arguments in the chat API is stopped; ask
  without `stream`.
- A client `chat_template` is stripped, and so are `priority`, `extra_body` and the
  other engine fields.
- The performance of the current release is not measured yet.
