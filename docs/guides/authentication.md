---
page_title: "Authenticating the Chainguard provider"
subcategory: ""
description: |-
  Configure authentication for the Chainguard provider, including assumable
  identities for CI and other automation.
---

# Authenticating the Chainguard provider

Every Chainguard API call the provider makes is authenticated. The provider
obtains a Chainguard token in one of two ways:

- **Ambient credentials** — the token that `chainctl auth login` caches on
  disk. Best for local, interactive use.
- **An assumable identity** — the provider exchanges an OIDC token from your
  environment for a Chainguard token. Best for CI and other automation, where
  no human is present to run a browser login.

You configure both through the optional `login_options` block. This guide
covers each path and then shows how to authenticate as an assumable identity.

## Interactive authentication

Log in once with `chainctl`:

```shell
chainctl auth login
```

The provider reuses that cached token, so no provider configuration is
required:

```terraform
provider "chainguard" {}
```

To have the provider open a browser and refresh the token itself when it
expires, configure the login flow. Set `auth0_connection` to the social
connection you sign in with:

```terraform
provider "chainguard" {
  login_options {
    # One of: google-oauth2, github, gitlab.
    auth0_connection      = "google-oauth2"
    enable_refresh_tokens = true
  }
}
```

If your organization uses a custom identity provider, select it with
`organization_name` (your verified organization) or `identity_provider_id`
instead of `auth0_connection`.

## Authenticating as an assumable identity

An assumable identity is a Chainguard identity that a workload authenticates
as by presenting an OIDC token, rather than by a human logging in. The identity
defines which token claims it accepts; any environment that can mint a matching
OIDC token can then assume it. This guide assumes the identity already exists.
Create one with the [`chainguard_identity`](../resources/identity.md) resource
or with `chainctl iam identities create`, and list existing identities and
their UIDPs with `chainctl iam identities list`.

To authenticate as the identity, give the provider two things:

- `identity_id` — the UIDP of the identity to assume.
- `identity_token` — the OIDC token to exchange, as either a file path or the
  token string itself.

```terraform
provider "chainguard" {
  login_options {
    identity_id    = "<identity-uidp>"
    identity_token = "/path/to/oidc/token"
  }
}
```

Most CI systems that support OIDC write a token to a file or expose one through
an environment variable. Point `identity_token` at that token. For example,
when the token is available as an environment variable, pass it through a
Terraform variable rather than hardcoding it:

```terraform
variable "oidc_token" {
  type      = string
  sensitive = true
}

provider "chainguard" {
  login_options {
    identity_id    = "<identity-uidp>"
    identity_token = var.oidc_token
  }
}
```

```shell
export TF_VAR_oidc_token="$(get-oidc-token)"
terraform apply
```

The token's audience (`aud`) claim must match the audience the identity
expects. By default the provider uses the `console_api` URL as the audience;
override it with the `CHAINGUARD_AUDIENCE` environment variable when your token
is minted for a different value.

### Constraints

- `identity_token` is exclusive. It cannot be combined with `auth0_connection`,
  `identity_provider_id`, `organization_name`, or `enable_refresh_tokens` —
  those configure the interactive browser flow, which the token exchange
  replaces. Refresh tokens are a browser-flow feature and do not apply.
- `identity_token` is a sensitive value. Source it from a CI secret or a
  short-lived OIDC token; never commit it.
- If you omit `identity_id`, token refreshes reuse the identity the current
  session was minted for. Set it to pin a specific identity, or to keep token
  caches separate when several identities share one environment.

## Verify authentication

A provider block on its own authenticates nothing. Terraform configures a
provider only when a resource or data source needs it, so a configuration that
contains just a `provider` block plans as "No changes" without ever logging in.

To confirm your credentials work, add a read-only data source and an output:

```terraform
data "chainguard_group" "group" {
  name = "YOUR.ORG"
}

output "group_id" {
  value = data.chainguard_group.group.id
}
```

Then run:

```shell
terraform plan
```

`plan` reads data sources, which forces the provider to authenticate — you
don't need to apply. If your credentials work, the plan resolves `group_id` to
the group's UIDP. If they don't, the provider returns an authentication error
instead of a silent no-op.

Two commands help when a login fails:

- `TF_LOG=INFO terraform plan` prints the provider's login logs, including the
  `login_options` it parsed, so you can see which flow ran.
- `chainctl auth status` reports which identity your ambient credentials belong
  to.

## Disabling automatic login

In non-interactive environments, disable the browser flow so a missing or
expired token fails fast instead of blocking on a prompt:

```terraform
provider "chainguard" {
  login_options {
    disabled = true
  }
}
```

## Next steps

- Bind an identity to a role with
  [`chainguard_rolebinding`](../resources/rolebinding.md).
- Look up an existing identity with the
  [`chainguard_identity`](../data-sources/identity.md) data source.
