---
title: "Identity"
weight: 25
---

# Identity {#identity}

Since **v0.4.0** (roadmap milestone M2, identity and access) the
[gateway]({{< relref "gateway.md" >}}) can take people from your identity
provider instead of holding a secret for each of them. It does not implement the
protocols itself: it is built on the libraries of the
[go-authn](https://github.com/go-authn) organisation, and their own documentation
is the reference.

| Need | Configuration block | Library |
| --- | --- | --- |
| a person calls the **inference API** with an OpenID Connect access token | `jwt` | [go-authn/oidc](https://github.com/go-authn/oidc) |
| an administrator **logs in** with a browser | `sso` | [go-authn/rp](https://github.com/go-authn/rp) |
| the identity provider is a **SAML federation** (RENATER, eduGAIN) | `sso` and `jwt`, pointed at the bridge | [go-authn/bridge](https://github.com/go-authn/bridge) |
| the identity provider **provisions** users and groups | `scim` | [go-authn/scim](https://github.com/go-authn/scim) |
| a client's **key is rotated** on a schedule | `key_rotation`, `vault` | the gateway |

## A configuration {#configuration}

This block was run through `llmflow-gateway check-config` v0.16.1 (with the key
files present), together with the rest of a configuration it needs (organisations,
a pool and a model, left out here).

```hcl
master_key_file     = "/etc/llmflow/master.key"
cache_salt_key_file = "/etc/llmflow/cache-salt.key"

trusted_proxies = ["10.0.0.0/24"]

routes {
  admin { allow = ["10.0.2.0/24"] }
}

vault {
  address    = "https://vault.example.org:8200"
  token_file = "/etc/llmflow/vault.token"
}

key_rotation {
  vault_prefix   = "secret/llmflow/keys"
  check_interval = "1h"
}

jwt {
  issuer   = "https://sso.example.org/realms/research"
  audience = "llmflow"
  type     = "at+jwt"
  rpm      = 60
  map {
    value = "lab-physics"
    team  = "lab-nlp"
  }
  map {
    claim   = "department"
    value   = "chemistry"
    team    = "lab-chem"
    project = "eval"
  }
}

sso {
  issuer             = "https://login.example.org"
  client_id          = "llmflow-admin"
  client_secret_file = "/etc/llmflow/sso.secret"
  redirect_url       = "https://llm.example.org/admin/callback"
  scopes             = ["openid", "profile", "groups"]
  session_ttl        = "8h"
  role {
    value = "llm-admins"
    role  = "proxy_admin"
  }
  role {
    value = "lab-physics-leads"
    role  = "team_admin"
    team  = "lab-nlp"
  }
}

scim {
  token_file = "/etc/llmflow/scim.token"
  base_url   = "https://llm.example.org/scim/v2"
  group "Lab Physics" { team = "lab-nlp" }
  group "Lab Physics Leads" {
    team = "lab-nlp"
    role = "team_admin"
  }
}
```

## OpenID Connect tokens on the inference API {#jwt}

With a `jwt` block, a bearer token that has the JWS compact shape (three
dot-separated segments) is verified by go-authn/oidc instead of being looked up as
a virtual key. The library refuses `alg: none` and every HMAC algorithm (so the
issuer's public key cannot be used as an HS256 secret), compares `iss` whole,
requires `aud` and `exp`, bounds the clock skew, and fetches keys over https only,
at most once per `min_refresh`. `type = "at+jwt"` refuses ID tokens (RFC 9068).
Discovery happens at start-up: a gateway whose issuer does not answer does not
start.

The gateway only maps the verified identity: the `map` blocks are tried in order,
and the first whose `claim` (default `groups`) holds `value` gives the team and an
optional project. Each person is one synthetic key, so the limits of the block
(`rpm`, `tpm`, `max_inflight`), budgets, IP allowlists and spend reports apply as
to a key. Every refusal is the same 401; the reason is logged on the server only.

## Administrator single sign-on {#sso}

With an `sso` block, administrators log in with OpenID Connect in a browser
instead of holding an admin token (go-authn/rp: authorization code with PKCE,
`state`, `nonce`, ID-token verification).

- `GET /admin/login` redirects to the provider; `GET /admin/callback` ends the
  login and redirects to `GET /admin/session`, which returns the user, their roles
  and the session's `csrf_token`. `POST /admin/logout` ends the session.
- The roles are the `role` blocks the person's claims match, **plus** the roles of
  the gateway user of the same name, read from the store on every request.
  Somebody with neither is refused at the callback.
- The session is a random 256-bit value in a `__Host-llmflow-session` cookie
  (Secure, HttpOnly, SameSite=Lax), stored server-side as its SHA-256.
- **CSRF**: a state-changing request authenticated by the cookie must carry the
  session's token in `X-CSRF-Token` and must not be cross-site (`Sec-Fetch-Site`)
  or come from another `Origin` than the `redirect_url`'s. Bearer-token requests
  are unaffected.
- Sessions live in the memory of one replica: behind several, use sticky sessions.

## SAML federations: RENATER, eduGAIN {#saml}

A SAML federation goes through
[go-authn/bridge](https://github.com/go-authn/bridge) (`authn-bridge`), an OpenID
provider in front of the federation. Register the gateway as one of its clients and
point `sso.issuer` (and `jwt.issuer`) at the bridge. The bridge's configuration is
its own, documented in [go-authn's repositories](https://github.com/go-authn); the
gateway's README gives this client block:

```hcl
# authn-bridge configuration, not the gateway's
client "llmflow-admin" {
  secret_file     = "/etc/authn-bridge/llmflow-admin.secret"
  redirect_uris   = ["https://llm.example.org/admin/callback"]
  id_token_claims = true   # the bridge's ID tokens carry no claims about the person otherwise
}
```

Ask for `scopes = ["openid", "profile", "groups"]`: the bridge's `groups` are
entitlements held to the identity provider that asserted them, so a `role` block
can match `urn:mace:...` or `urn:geant:...` values. (This block belongs to
authn-bridge, not to `llmflow-gateway`, so `check-config` cannot check it.)
See also the [guide]({{< relref "guides/fairshare/identity.md" >}}).

## SCIM provisioning {#scim}

With a `scim` block, an identity provider (Entra ID, Okta, Keycloak) provisions
users and groups at `/scim/v2` (RFC 7643, RFC 7644); the protocol is
go-authn/scim, and the gateway stores what is pushed.

- A provisioned user is a gateway user of the same `userName` (letters, digits and
  `_ . - @ +`, at most 128), disabled while `active` is false or after deletion,
  which revokes their admin tokens.
- The members of a mapped group get the block's role on its team (`member` by
  default, `team_admin` or `viewer`); leaving the group, deactivation or deletion
  takes it away. Only those provisioned (role, team) pairs are managed: roles
  granted by hand stay.
- The state is reconciled after every change and at start-up.
- The bearer token is compared in constant time; `/scim/v2` obeys the
  `routes.admin` networks.

## Key rotation {#key-rotation}

- `POST /admin/keys/{id}/rotate` gives a key a new secret; the key keeps its ID,
  so limits, budgets, allowlists and reports continue. The replaced secret stays
  valid for `grace_period` (at most `720h`), or is revoked at once if absent.
- `PUT /admin/keys/{id}/rotation` schedules it: `every` (`1h` to `8784h`),
  `grace_period` and a `target`, a `vault:` reference under
  `vault_prefix/<the key's team>/`. This needs the `key_rotation` and `vault`
  blocks above. Every `check_interval`, a due key gets a new secret written into
  that KV v2 field, where the client reads it. If the write fails the rotation is
  undone and retried at the next check.

## OIDC verified against real identity providers (2026-10-26) {#verified}

The JWT path (go-authn/oidc), SSO (go-authn/rp) and SCIM had met only test keys.
The `e2e/oidc` suite of [bench]({{< relref "benchmarking.md" >}}) v0.3.0 runs the
same checks against two real providers, each behind a counting proxy that stands in
as the issuer, so every key-set fetch the gateway makes is counted.

- **dex v2.46.1**, the upstream container (the fleet's `claimward/dex` is an
  unmodified fork 232 commits behind it, and was not used).
- **go-authn/bridge v0.23.4** in front of a test SAML federation and identity
  provider written in Go; the provider's metadata, responses and (encrypted)
  assertions go through `xmlsec1` 1.3.12.
- **Gateway under test:** v0.8.0, with two organisations, `jwt` and `sso` role
  mappings and a `scim` group binding.

All checks **pass for both providers**:

| Check | Evidence |
| --- | --- |
| (a) a token is accepted and charged to its organisation and team | `llmflow_requests_total` and `/admin/spend` per team |
| (b) wrong audience; signed by another key; `alg: none`; HS256 keyed with the public key (PEM and DER); expired | 401 with a body identical to a garbage bearer's (an expired token is refused after `exp` plus the 5 s clock skew) |
| (b) 40 tokens with unknown `kid`s | exactly **1** key-set fetch at the provider |
| (c) signing-key rotation | the old (unexpired) and the new token both 200, 1 fetch; an SSO login right after works |
| (d) SSO: code, PKCE S256, state, nonce | `__Host-llmflow-session` cookie (Secure, HttpOnly, SameSite=Lax); role `org_admin` in `/admin/whoami` |
| (d) replayed callback; callback in another browser; no role binding; logout | 400; 400; 403; logout without the CSRF token 403, with it 200, then the old cookie 401 |
| (e) an `org_admin` of organisation A against organisation B | 13 admin listings without leaks; 6 writes on B answer 403 (control: the same write on A answers 200); 7 UI pages without B |
| (f) SCIM user and group to role | `team_admin` visible in `/admin/users/.../roles` and in the person's session |

**Findings** (the first two were fixed in gateway v0.10.0; the third was still
open when this was written):

- [gateway #60](https://github.com/go-llmflow/gateway/issues/60): SSO's ID-token key
  set refreshed at most once a minute (a fixed default), so a provider rotation right
  after start-up refused logins; observed with dex rotating every 30 s. **Fixed:**
  `sso.min_refresh` (default 1 minute, at least 5 s, as for `jwt`); lower it for a
  provider that rotates faster.
- [gateway #61](https://github.com/go-llmflow/gateway/issues/61): the README said
  keys are fetched over https only, but http to a **loopback** issuer is accepted.
  **Fixed in the README:** it now states this as go-authn/oidc's deliberate
  exemption (a loopback connection crosses no network link). Read "https only" above
  as "https, except loopback".
- [go-authn/bridge #79](https://github.com/go-authn/bridge/issues/79): the
  signing-key rotation procedure is not documented in the bridge's README; the one
  the test used (`authn-bridge keygen`, `retired_signing_key_files`, then a restart)
  works.

## Networks, proxies and secrets {#networks}

- `trusted_proxies` lists the reverse proxies whose `X-Forwarded-For` is believed;
  it is read **from the right**, skipping them. `routes { admin | metrics | healthz
  | inference { allow = [...] } }` restricts each route class to networks;
  `metrics` and `healthz` default to loopback.
- IP allowlists on an organisation, a team or a key (`GET`, `PUT /admin/allowlists`)
  must all match the client, else 403 with one generic message.
- Every `*_file` attribute is a secret reference: a path, or a field of an
  OpenBao or HashiCorp Vault KV v2 secret, `vault:<mount>/<path>#<field>`, which
  needs the `vault` block (https only, no redirect followed, reads cached).
- User names may contain `@` and `+`.

See the gateway's [limitations]({{< relref "gateway.md" >}}#limitations) 11 to 14
for what this does not do yet.
