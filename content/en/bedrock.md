---
title: "Amazon Bedrock"
weight: 24
---

# Amazon Bedrock as an upstream {#bedrock}

Since **gateway v0.23.0** (#89, #121) a model can be **Claude on Amazon Bedrock**, opt-in, in the
same position as [Anthropic as an upstream]({{< relref "gateway.md" >}}#anthropic) and under the
same hard-budget guarantee. The path is InvokeModel and InvokeModelWithResponseStream with the
Anthropic Messages body, signed with SigV4 by [go-llmflow/aws]({{< relref "libraries/aws.md" >}}).
**Claude model IDs only**: Converse, for other vendors' models, is a later change. This page is
written from the gateway's README and the v0.23.0 release notes. The provider's benchmarks are
written but **have not been run** (the machine's load was above the gate), so no performance figure
is given.

Since **v0.23.1** the gateway runs on go-llmflow/aws v0.1.1: **signing uses a cached derived
key** (keyed by the day, region, service and a SHA-256 of the secret, bounded, no secret stored),
so Bedrock signing is level with the AWS SDK's; behaviour is unchanged.

## Configuration {#config}

```hcl
model "claude-sonnet-bedrock" {
  provider          = "bedrock"
  region            = "eu-west-3"     # the endpoint is derived: bedrock-runtime.eu-west-3.amazonaws.com
  model_id          = "eu.anthropic.claude-sonnet-4-5-20250929-v1:0"   # the price is the price of THIS id
  pool              = "bedrock"
  allow_external    = true            # required: prompts leave the site
  orgs              = ["etabA", "etabB"]
  anthropic_betas   = ["context-management-2025-06-27"]   # same allow-list as the Anthropic provider
  context_window    = 200000          # bounds the input of a budgeted request
  max_output_tokens = 64000
  cost {
    unit                    = "USD"
    price_date              = "2026-10-31"
    input_per_mtok          = 3
    output_per_mtok         = 15
    cached_per_mtok         = 0.3
    cache_write_5m_per_mtok = 3.75
    cache_write_1h_per_mtok = 6
  }
  # Secret references (a file or a vault: reference), never the environment or the command line.
  org_credentials "etabA" {
    access_key_id_file     = "/etc/llmflow/bedrock-etabA.id"
    secret_access_key_file = "/etc/llmflow/bedrock-etabA.secret"
    session_token_file     = "/etc/llmflow/bedrock-etabA.token"   # optional
  }
  org_credentials "etabB" { bearer_token_file = "/etc/llmflow/bedrock-etabB.apikey" }   # a Bedrock API key
}
```

This configuration, with `pii` and `hide_secrets` guardrails, a pool and organisations around it
(and the credential files present, with `session_token_file` omitted), passes
`llmflow-gateway check-config` v0.24.1, which also loads every credential and prints no value.
The README's own example uses `vault:` references for etabA and needs a `vault` block.

- **`region`.** The endpoint is derived from it and is always https. `endpoint_override` is for a
  test's fake Bedrock (https, or http to a loopback address only). A client cannot choose a host.
- **`model_id`.** The exact model ID or inference profile ID. Geographic or global inference
  profiles are allowed; the price is set per configured `model_id`, not per base model.
- **Credentials.** SigV4 keys (`access_key_id_file`, `secret_access_key_file`, optional
  `session_token_file`) or a Bedrock API key sent as `Authorization: Bearer`. AWS recommends
  short-term keys (up to 12 hours) for production and long-term keys "only for exploration":
  prefer SigV4 with a session token that you rotate, or a short-term key refreshed through the
  secret reference.
- **`org_credentials`, `cache_isolation`.** The prompt-caching page does not say that caches are
  isolated per AWS account, so until a source says so `check-config` requires an
  `org_credentials` block, with an AWS account of its own, for every organisation that may use the
  model when more than one may and `cache_isolation` is not `"none"`. Two organisations naming the
  same credential reference are refused (checked), and so are two whose files hold the same key at
  start-up. A `credentials { ... }` block is the default for organisations without their own.
- **`allow_external = true`** stays mandatory, as do the pre-call `pii` and `hide_secrets`
  guardrails.
- **Routes.** A Bedrock model is reachable at `/v1/messages` only. **`/v1/chat/completions`
  answers 400**, as for the Anthropic provider. `/v1/messages/count_tokens` is not offered: it
  would send the prompt to a second operation the gateway does not meter.

## The guarantee {#guarantee}

**A budget is never exceeded by what it admits**, as for the Anthropic provider (see
[Hard budgets]({{< relref "gateway.md" >}}#hard-budgets)). The gateway holds **W**, the worst
case: the context window comes from `context_window`, `max_tokens` is always set, and the price is
the **highest input price, cache writes included**, because Bedrock's implicit prompt caching
"automatically attempts to reuse eligible prompt prefixes" and cross-region inference "may lead to
increased cache writes".

- **Usage and settlement.** `input_tokens`, `output_tokens`, `cache_read_input_tokens` and
  `cache_creation_input_tokens` are read from the answer (non-stream) or from `message_start` and
  `message_delta` inside the decoded chunk events (stream), and priced as for Anthropic.
- **An incomplete stream is billed at W**: a cut at any byte, an exception event
  (`throttlingException`, `modelStreamErrorException`, ...), a CRC failure, a missing
  `message_stop`, a `message_delta` without usage. So is a 200 answer without usage or with a body
  cut. W applies when a budget holds one; without a budget the usual estimate is recorded.
- **Streaming.** The binary event stream is untrusted input: the decoder's limits bound the message
  (1 MiB), the header block (4 KiB) and the header count (16), both CRCs are checked, and a chunk's
  base64 is bounded. It is re-emitted as Anthropic SSE to `/v1/messages` clients; the error cases
  become an `event: error` with a fixed text, never the provider's.

## What is refused, and why {#refusals}

The seven ways a budget could overflow (issue #89), and what the gateway does about each, at the
request or in `check-config`:

| Option | What happens | Why |
| --- | --- | --- |
| `guardrailConfig`, `amazon-bedrock-guardrailConfig`, `X-Amzn-Bedrock-Guardrail*` | refused (400) | guardrails are billed separately and not priced |
| `serviceTier`, `performanceConfig`, `X-Amzn-Bedrock-Service-Tier`, `-PerformanceConfig-Latency` | refused | the price depends on the tier; only the default tier is used |
| a prompt router ARN as `model_id` | refused by `check-config` | Bedrock picks the model, so the price |
| `additionalModelRequestFields` | refused | fields the gateway cannot bound |
| `anthropic_beta` in the body | joins the header betas and goes through the same allow-list and unbounded-beta rules | a context beta wider than `context_window`, or a compaction beta, would break W |
| Provisioned Throughput ARNs | refused by `check-config` | billed per hour, not per token: no per-request cost to bound |
| `promptVariables`, Prompt-management ARNs | refused | the prompt is server-side, its size unknown |

Also refused as `model_id`: custom, imported, Marketplace and SageMaker resources (price and vendor
unknown) and any non-Anthropic model. `X-Amzn-Bedrock-Trace` and `-Request-Metadata` are refused
too: headers are built from scratch and nothing the client sent is forwarded. `check-config`
v0.23.0 was run on three of these: it refused a non-Claude `model_id` ("only Anthropic Claude
models are served through InvokeModel here"), a Provisioned Throughput ARN ("billed per hour, not
per token") and two organisations sharing one credential.

## Data retention {#retention}

**Two AWS pages say different things, and both are quoted here.**

- The [Converse API reference](https://docs.aws.amazon.com/bedrock/latest/APIReference/API_runtime_Converse.html)
  (opened 2026-11-01) says: "Amazon Bedrock doesn't store any text, images, or documents that you
  provide as content. The data is only used to generate the response."
- The data retention page of the Bedrock user guide (opened 2026-10-31) describes retention as a
  per-region, per-account mode. As the README words it:

  - the **mode** is `none`, `default` or `aws_review`;
  - under `default`, "data may be retained for abuse detection purposes";
  - models requiring `aws_review` keep prompts and completions "within the AWS boundary for up to
    30 days";
  - the content is "not shared with the model provider";
  - with cross-region inference, "retained inputs and outputs are stored in destination Regions".

**Retention is therefore not "none" by default**, whatever the API reference's summary says. Set `none` where the model allows it, use an EU region and an EU inference profile (`eu.anthropic...`)
to keep inference in the EU regions the profile lists (a geographic profile may route to another
region of its geography), and **decide with your DPO**: prompts leave the site, which is why the
guardrails are mandatory.

## Security {#security}

- Credentials never appear in a log, an error, a metric, a record or the audit chain: a provider
  message that echoes one is redacted, a 401 or 403 says only that the credentials were refused,
  and a planted-secret test scans every output.
- The request is signed over the exact bytes sent, at the gateway's clock. The tests use a fake
  Bedrock that verifies every signature, and the independent check against aws-sdk-go-v2's signer,
  on these exact paths, runs in go-llmflow/aws CI (the `sdk-witness` job).
