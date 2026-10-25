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

**Why.** vLLM keeps stored responses in the memory of the instance that produced
them, readable and continuable by anyone who holds the id. On a shared cluster, a
tenant holding an id could read or continue another tenant's conversation, so the
gateway does not allow state. `GET /v1/responses/{id}` and the cancel route are not
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

How a given agent is told its base URL and its key (an environment variable, a
settings file) is that agent's own configuration, which this site does not
reproduce: read it in the agent's documentation.

What the gateway guarantees on this route: guardrails cover `system`, text and
`thinking` blocks, `tool_result` content and the strings inside `tool_use` inputs;
guarded streams pass through an Anthropic event filter; error bodies carry
`"type": "error"`; `/v1/messages/count_tokens` is not routed. Accounting counts
cached tokens as vLLM reports them.

## Limits to keep in mind {#limits}

- A Responses **stream** under **post-call** guardrails is refused: ask with
  `"stream": false`.
- A guarded stream that carries tool-call arguments in the chat API is stopped; ask
  without `stream`.
- A client `chat_template` is stripped, and so are `priority`, `extra_body` and the
  other engine fields.
- The performance of the current release is not measured yet.
