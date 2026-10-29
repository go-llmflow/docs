---
title: "Identity with your federation"
weight: 55
---

# Identity with your federation {#identity}

On a cluster co-owned by two institutions, people already have an account at their
own institution. The aim is that nobody needs a second one: A's and B's users log
in where they always do, and the gateway learns from that login which team they
belong to.

If the institutions are in a SAML federation such as **RENATER** (France) or
**eduGAIN**, the piece in the middle is
[authn-bridge](https://github.com/go-authn/bridge) (go-authn/bridge): an OpenID
Connect provider placed in front of the federation. People log in at their own
university; the gateway, which speaks OpenID Connect, gets a token. The bridge and
the libraries below are documented in
[go-authn's repositories](https://github.com/go-authn); this page only shows how
the gateway is pointed at them.

```text
user ──> their university (SAML, RENATER / eduGAIN)
              │
              v
         authn-bridge (OpenID Connect provider)
              │  token
              v
   llmflow gateway: sso (administrators), jwt (inference API) ──> vLLM
```

## 1. Register the gateway at the bridge {#bridge}

The gateway is a client of the bridge. This block is authn-bridge's configuration,
copied from the gateway's README (it is not `llmflow-gateway` configuration, so
`check-config` does not apply to it):

```hcl
client "llmflow-admin" {
  secret_file     = "/etc/authn-bridge/llmflow-admin.secret"
  redirect_uris   = ["https://llm.example.org/admin/callback"]
  id_token_claims = true   # the bridge's ID tokens carry no claims about the person otherwise
}
```

## 2. Point the gateway's `sso` and `jwt` blocks at the bridge {#gateway}

Administrators log in through `sso`; users of the inference API present the access
token through `jwt`. The `groups` the bridge returns are entitlements held to the
institution that asserted them, so the `role` and `map` blocks match them. This
block was run through `llmflow-gateway check-config` v0.18.2 (with the key files
present, and organisations, a pool and a model alongside):

```hcl
master_key_file     = "/etc/llmflow/master.key"
cache_salt_key_file = "/etc/llmflow/cache-salt.key"

sso {
  issuer             = "https://bridge.example.org"
  client_id          = "llmflow-admin"
  client_secret_file = "/etc/llmflow/sso.secret"
  redirect_url       = "https://llm.example.org/admin/callback"
  scopes             = ["openid", "profile", "groups"]
  role {
    value = "llm-admins"
    role  = "proxy_admin"
  }
  role {
    value = "b-lab-leads"
    role  = "team_admin"
    team  = "b-lab"
  }
}

jwt {
  issuer   = "https://bridge.example.org"
  audience = "llmflow"
  type     = "at+jwt"
  map {
    value = "b-lab"
    team  = "b-lab"
  }
}
```

With this, a member of B's laboratory who holds a token with the group `b-lab`
reaches the gateway as a person of team `b-lab`: their requests count in B's
share, their budget and limits are the team's, and the spend report names them.
The `value` strings stand for whatever entitlements your federation really issues
(for example `urn:mace:...` values); read them from a real token before writing
the blocks.

Behind it, the gateway refuses `alg: none` and HMAC tokens, requires `aud` and
`exp`, and answers every refusal with the same 401. See the
[Identity page]({{< relref "identity.md" >}}) for the details, for SCIM
provisioning and for key rotation, and keep in mind its
[limitations]({{< relref "gateway.md" >}}#limitations): a person holding a token
cannot be revoked individually before it expires, and SSO sessions live in one
replica.
