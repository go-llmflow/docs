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
| **several** identity providers, or a federation proxy filtered to some institutions | `sso { provider }`, `jwt "<name>"`, `accept` | the gateway ([below](#multi-idp)) |
| the identity provider **provisions** users and groups | `scim` | [go-authn/scim](https://github.com/go-authn/scim) |
| a client's **key is rotated** on a schedule | `key_rotation`, `vault` | the gateway |

## A configuration {#configuration}

This block was run through `llmflow-gateway check-config` v0.25.2 (with the key
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
  domain   = "example.org" # a person is local@example.org (see below)
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
  domain             = "example.org"
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
  domain     = "example.org"
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

## Several identity providers, and federations {#multi-idp}

Since **v0.27.0**, when there is no federation, each institution can keep its own
OpenID provider: the `sso` block takes one `provider "<name>"` block per provider,
and the `jwt` block becomes one `jwt "<name>"` block per issuer. Behind a federation
proxy (eduGAIN through MyAccessID, eduTEAMS, Renater, a Keycloak broker), an
`accept` list keeps the login to the institutions you serve. The single-issuer
forms above stay valid. This configuration was run through `check-config` v0.30.0
(it is `examples/federation.hcl` in the gateway repository):

```hcl
sso {
  redirect_url = "https://llm.example.org/admin/callback"

  provider "univ-a" {
    issuer             = "https://sso.univ-a.example"
    client_id          = "llmflow-admin"
    client_secret_file = "vault:secret/llmflow#sso-univ-a"
    orgs               = ["univ-a"]
    domain             = "univ-a.example" # the institution whose accounts predate scoped names
    role {
      value = "llm-admins"
      role  = "org_admin"
      org   = "univ-a"
    }
  }

  provider "partners" {
    issuer    = "https://proxy.federation.example"
    client_id = "llmflow-admin"
    orgs      = ["partners"]
    accept {
      claim  = "schac_home_organization"
      values = ["univ-c.example", "univ-d.example"]
    }
    role {
      claim = "schac_home_organization"
      value = "univ-c.example"
      role  = "member"
      team  = "lab-partners"
    }
  }
}

jwt "partners" {
  issuer   = "https://proxy.federation.example"
  audience = "llmflow-api"
  orgs     = ["partners"]
  accept {
    claim  = "schac_home_organization"
    values = ["univ-c.example", "univ-d.example"]
  }
  map {
    value = "llm-users"
    team  = "lab-partners"
  }
}
```

- **A callback per provider**, `/admin/callback/<name>`: register that redirect
  URI at that provider (RFC 9700 §4.4.2.2). `/admin/login` lists the providers on a
  page without JavaScript; `/admin/login/<name>` starts a login. A login belongs to
  the provider it was sent to: another provider's answer, or an `iss` response
  parameter that is not the provider's issuer (RFC 9207), is refused, and the ID
  token's `iss` is checked against that provider alone.
- **`orgs`** says what a provider may vouch for. A role on another organisation is
  refused (`check-config` rejects the file; at run time it is refused and logged),
  and a proxy-wide role needs `allow_proxy_roles = true`. An institution's provider
  never makes anybody administrator of another institution.
- **An identity is (provider, subject)**, because `sub` is unique only within its
  issuer (OpenID Connect Core §2). A stored user is linked to the provider and
  subject of the first login; the same name arriving through another provider, or
  with another subject, is refused, and a session opened before the user was
  stored, or by another identity, is dropped once the user exists. Users stored
  before v0.27.0 need no migration: a user without a link is adopted at their first
  login by the single `sso` issuer of earlier versions, or by a `provider` block only
  when its `orgs` cover every role the user holds (proxy-wide roles need
  `allow_proxy_roles`); a user with no role is adopted only by a provider with
  `adopt = true`.
- **`accept { claim, values }`**: all the `accept` blocks must hold, and any value of a
  multi-valued claim may match. The person is told the institution value that was
  refused and nothing else. `schac_home_organization` (a domain name, `caseIgnoreMatch`
  in the SCHAC 1.6.0 schema), `eduperson_scoped_affiliation` and
  `voperson_external_affiliation` are compared without case; every other claim exactly.
  The claim must be in the ID token (the access token for `jwt`): MyAccessID's
  documentation lists `schac_home_organization` for userinfo and introspection, not
  for the ID token, and the gateway does not call userinfo.
  Only ASCII letters are folded (the Kelvin sign U+212A is not a `k`). The matching
  and the routing are [go-authn/oidc](https://github.com/go-authn/oidc) v0.5.0 (`Set`,
  `Accept`); the mix-up defences are [go-authn/rp](https://github.com/go-authn/rp)
  v0.3.0 (`Set`, `CheckIssuer`).
- **In front of a SAML federation**, [go-authn/bridge](https://github.com/go-authn/bridge)
  can already restrict who logs in, by IdP entityID (`saml { idps = [...] }`), and
  disable one institution's IdP at run time. `accept` filters on the institution claim
  whatever the proxy; with the bridge, set `id_token_claims = true` on the gateway's
  client so the claim is in the ID token.
- **A person is `local@domain`** (v0.29.0; SCIM since v0.30.0). An account is relative to an
  institution's domain: there is no global "alice", and a username claim is not an identity (a token
  of one issuer saying `preferred_username` "alice" was once charged to the alice of another). The
  name of an administrator who logs in, of a JWT's holder and of a provisioned user is scoped, and its
  domain is **checked against what the issuer may assert**, as a Shibboleth service checks a scope: the
  issuer's `domain` and `domains`, or the institutions of its `accept` list. A name whose scope the issuer
  may not assert is refused (a login is 403; a token is attributed to the synthetic `jwt-…` key id of its
  issuer and subject). The claim carrying the name is the `username_claim`
  (`eduperson_principal_name`, eduPersonPrincipalName: single-valued and `caseIgnoreMatch` in the eduPerson
  schema, the scope being the domain that issued it; or `voperson_id`); names are compared in ASCII lower
  case, by go-authn/oidc `ScopedName` (v0.7.0 since gateway v0.36.1).
- **With an `accept` on `schac_home_organization`, a token's institution is its name's scope** (v0.38.0,
  **breaking**). The claim filtered the login and the name's scope was checked against the issuer's
  domains, but nothing tied the two: a token of univ-d.fr's IdP stating `schac_home_organization =
  univ-c.fr` passed the filter, and a role map on that claim gave it univ-c.fr's team. Now the scope of
  the person's name must **equal** a value of the claim, and that value must be one `accept` lists;
  otherwise the login is refused (403) or the token gets no key (JWT): no team, no role. This holds for
  `sso` providers and `jwt` issuers. **Upgrade note:** an unscoped name takes the issuer's default domain as
  its scope, so the tokens and logins of a **federation proxy** (eduGAIN, MyAccessID, Renater) must carry a
  scoped name for the person's institution (`eduperson_principal_name` or `voperson_id`); a proxy
  that releases a bare `preferred_username` now has every person refused. Check the claims your proxy
  releases before upgrading. The other half is on the IdP side: go-authn/saml v0.6.0 and
  [go-authn/bridge](https://github.com/go-authn/bridge) v0.23.9 no longer release a
  `schacHomeOrganization` that is not one of the scopes of the IdP that stated it. See
  [Fixes in v0.38.0]({{< relref "security.md" >}}#v0380).
- **Creating users by the admin API** (`POST /admin/users`, v0.38.0, **breaking**). Once a domain is
  configured a name is `local@domain`, and a bare name is refused, to the proxy admin too (unless
  `legacy_unscoped_names` is on): the gateway refuses to start with a bare stored user once accounts are
  migrated, so one organisation admin could otherwise stop the next restart for every institution. An
  **organisation admin** creates only names in the domains of the providers and issuers bound to its own
  organisations (and of the SCIM client, for the organisations of the teams its groups map to). A
  **proxy admin** is unrestricted. An automation that created bare names, or names of another institution
  with an organisation admin's token, now gets a refusal.
- **A person has one institution: the scope of their name** (v0.39.0, **breaking**). The v0.38.0 tie
  compared the name's scope with the values the token states, but a team or role map on
  `schac_home_organization` still matched any stated value (a token stating `[univ-d.fr, univ-c.fr]`
  got C's team), the tie read `schacHomeOrganization` where the map read `schac_home_organization`, and
  an issuer with `domains` and no `accept` had no tie. Now one function gives the person's institution,
  the scope of their name (checked against what the issuer may assert), and every mapping keys on it:
  a `map` or `role` on the institution claim holds only for that institution, never for a raw claim
  value, and an `accept` on it must list it. The claim itself, in either spelling (the configuration has
  one canonical name, `schac_home_organization`), is only cross-checked: a token that states
  institutions among which its name's scope is not is refused. The tie applies whenever an accept or a
  mapping uses the institution claim, with or without `accept`; a token with no scoped name is then
  refused.
- **A domain has one owner** (v0.39.0, **breaking**). `POST /admin/users` let an
  organisation admin create people in every domain of a SCIM client or provider that its organisation
  shared with others. An organisation now owns a domain only by `org { domains = ["univ-a.fr"] }`, or
  through an issuer or provider bound to it alone, or a SCIM client all of whose groups map to its teams.
  A **shared** SCIM client or `sso` provider owns nothing, the legacy unrestricted `sso` block owns
  nothing, and a domain claimed by two organisations is refused at load. **Upgrade note:** with a SCIM
  client or an `sso` provider shared by several organisations, claim each domain with
  `org { domains = [...] }`; until then their administrators cannot create the people of those domains
  (the proxy admin still can). See [Fixes in v0.39.0]({{< relref "security.md" >}}#v0390).
- **Migration: `migrate-users`** (v0.32.0). Accounts that predate scoped names are bare (`alice`). The
  institution they belong to sets `domain = "example.org"` on its `sso` provider, `jwt` block and `scim`
  block (only one issuer may carry it, since unscoped names belong to one institution; the others list
  theirs in `domains`). Then `llmflow-gateway migrate-users --config gateway.hcl` prints the plan and
  `--apply` renames every stored bare user to `alice@example.org` **and everything keyed by their name** in
  one transaction: roles, admin tokens, the identity link, the owners of keys, budgets and the budget
  ledger (user and workload-profile scopes), spend records, profile assignments and warning marks. It is
  idempotent, refuses to run when a new name is already in use (it lists the collisions and changes
  nothing), and is written to the audit log before it starts and after it ends. The audit entries (a hash
  chain) and the SCIM records are left as they were.

  **People who were never stored users (v0.34.0).** A caller with a JWT only has no stored user: their spend
  records, and perhaps a budget, carry the bare name. Without a rename, after the upgrade the same person is
  `alice` before and `alice@example.org` after, and per-person reports and `/v1/me/usage` split in two. Since
  v0.34.0 `migrate-users` plans those names too, with one grouped query (the records are not loaded), and
  renames their records, budgets, ledger, profile assignments and warning marks to `alice@example.org`, in the
  same transaction, with the domain used for stored users. A new name already in use (a stored user, a link, a
  budget, a profile or warning row) is a collision: it is listed and nothing is changed. Records already under
  `alice@example.org` are not a collision; the history joins them. A second run changes nothing, and the audit
  entry counts these people as `record_only_users`. Run it after v0.34.0, even if you migrated with v0.32.0.
  **The gateway refuses to start while stored users with bare names remain and a domain is configured**;
  `legacy_unscoped_names = true` starts it anyway with a read-only alias until you migrate. Without a domain
  an unscoped name is no identity, so **upgrading a gateway that uses `sso`, `jwt` or `scim` needs
  `domain` set** (`scim` without one is a configuration error).
- **SCIM acts only on the users it provisioned** (v0.30.0). A user SCIM creates has the origin `scim`; only
  such a user is enabled, disabled or retired by SCIM, so a SCIM client cannot disable a proxy admin by
  creating and deleting a user of that name, and a user an administrator disabled stays disabled whatever
  SCIM writes next. **Since v0.36.0 the roles follow the same rule**: SCIM manages the roles of a user of
  origin `scim` only, so posting the name of a hand-made user no longer revokes its hand-granted roles,
  and a mapped group grants no role to a user SCIM did not create.
- **Names compare without case, ASCII letters only** (go-authn/oidc `ASCIILower` and `EqualASCIIFold`, v0.36.1).
  `strings.EqualFold` folds Unicode: the Kelvin sign U+212A is a "k" and U+017F (long s) an "s", so
  `kate@example.org` and its Kelvin-sign spelling were one SCIM user in the in-memory store (the second was
  refused, a lookup of `kate` returned both), and a SCIM group `Lab Phyſics` received the roles mapped to
  `Lab Physics`. Group names in `scim` blocks and the `domain` values are folded the same way. With PostgreSQL
  the unique index on `lower(name)` is unchanged and still refuses the second, Kelvin-spelled user (it fails
  closed); a lookup returns only the ASCII-fold matches.
- **Inference tokens** are routed to their `jwt` block by `iss`; an unknown `iss` is a
  401 and no other issuer's keys are tried. Each block has its own audience, JWKS,
  maps, `orgs` and `accept`.

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
