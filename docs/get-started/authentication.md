# Authenticate with the API

Every Tessera request carries a bearer token. This page covers creating a token, choosing
its scopes, rotating it, and what to do when authentication fails.

## How authentication works

Tessera uses long-lived API tokens rather than an OAuth exchange. A token belongs to a
workspace, carries a fixed set of scopes, and is presented on every request:

```
Authorization: Bearer tsk_live_...
```

There is no refresh flow. A token is valid until it is revoked or reaches its expiry date.

!!! info "Why not OAuth"
    Tessera is written against by services, not by end users on behalf of themselves. A
    client-credentials exchange would add a round trip and a cached secret without
    changing the trust model, since the service would still hold a long-lived credential
    to obtain the short-lived one.

## Token types

| Prefix | Environment | Writes to |
|---|---|---|
| `tsk_live_` | Production | Your real audit log |
| `tsk_test_` | Sandbox | An isolated log, purged every 30 days |

A test token cannot read or write production data, and a live token cannot reach the
sandbox. Use the prefix to tell at a glance which environment a piece of configuration
points at.

## Scopes

Scopes are granted at creation and cannot be changed afterwards. To change a token's
scopes, create a new token and revoke the old one.

| Scope | Allows |
|---|---|
| `events:write` | Recording new events |
| `events:read` | Querying and retrieving events |
| `exports:create` | Creating and downloading export jobs |
| `grants:admin` | Creating, listing and revoking grants |

Grant the narrowest set that the service needs. A service that only emits events should
hold `events:write` and nothing else — it then cannot read the log back, which limits
what an attacker gains from stealing its credential.

## Create an API token

1. In the Tessera console, go to **Settings → API tokens**.
2. Select **Create token**.
3. Give the token a name that identifies the *service*, not the person — `billing-service`
   rather than `jo's token`. Tokens outlive the people who create them.
4. Select the scopes. The console warns if you select `grants:admin` alongside
   `events:write`, because that combination lets one credential both change permissions
   and record the access.
5. Optionally set an expiry. Tokens without an expiry appear in the quarterly credential
   review until they are given one.
6. Select **Create**, then copy the secret.

The secret is displayed once. Store it in your secret manager before leaving the page.

## Rotate a token

Tessera supports overlapping tokens so rotation needs no downtime.

1. Create a new token with the same scopes and a name marking the rotation date.
2. Deploy the new secret to the service.
3. Confirm traffic has moved: **Settings → API tokens** shows a last-used timestamp
   against each token.
4. Once the old token shows no use for a full deployment cycle, revoke it.

Revocation takes effect within 60 seconds across all regions.

## When authentication fails

| Status | Meaning | What to check |
|---|---|---|
| `401 unauthenticated` | Token missing, malformed, revoked or expired | The `Authorization` header is present and spelled `Bearer `, with one space. Check the token has not been revoked. |
| `403 insufficient_scope` | Token is valid but lacks the scope | The response body names the scope required. Create a token that has it. |
| `403 wrong_environment` | Test token against production, or the reverse | Check the token prefix matches the host you are calling. |

A `401` means Tessera does not know who you are. A `403` means it knows and is refusing.
Treating them the same is the most common integration bug we see — a retry loop on `403`
will never succeed.

## Related

- [Error reference](../reference/errors.md)
- [Tokens and sessions](../concepts/tokens-and-sessions.md)
