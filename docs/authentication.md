# Authentication

**Status:** the pre-sale endpoint is not deployed and credentials for it are not issued
self-serve. This documents the model so integrators can review it before it exists.
Access requests: curator@joyofwine.co

## Credentials

Bearer token in the `Authorization` header.

```http
POST /api/v1/presale/verify
Authorization: Bearer cvl_sk_...
Content-Type: application/json
```

Two key classes:

| Prefix | Class | Use |
|---|---|---|
| `cvl_pub_` | Publishable | Safe in contexts where the key is visible. Read-only surfaces. |
| `cvl_sk_` | Secret | Server-side only. Never ship in client code or an agent prompt. |

Keys are stored hashed. A key is looked up by its prefix and then verified, so the
plaintext is never retrievable — a lost key is rotated, not recovered.

## The authorization model, and why the request has no account id

**The credential decides the account. The request body cannot.**

`seller_ref` is required in the request, but it may only *confirm* the account the
credential already identifies. If it names anything else, the call fails:

```json
{ "error": { "code": "SELLER_NOT_AUTHORIZED",
             "message": "seller_ref does not match the authenticated account." } }
```

This is deliberate and worth stating plainly, because the enforcement API does it
differently. `POST /api/v1/compliance/check` accepts a `merchant_id` in the body — a
historical shape that works because the value is cross-checked, but which is one forgotten
check away from being an authorization bypass. The pre-sale contract does not have the
field at all.

Today one credential maps to exactly one seller. `seller_ref` exists so that a future
marketplace holding one credential across many sellers has somewhere to name which one,
at which point the check becomes "is this seller inside the authorized account" rather
than "is this the account".

## Opaque tokens

Two optional fields. Neither carries a value the recipient can read back, and neither is
an identity.

| Field | Prefix | Carries |
|---|---|---|
| `buyer_token` | `bt_` | Continuity for rolling volume limits |
| `age_verification_token` | `avt_` | Proof that age verification was completed |

Format:

```
<prefix><payload-base64url>.<hmac-base64url>
```

The signature is HMAC-SHA256 over `<kind>:<payload>`. Verification is constant-time.

### How they degrade

This is the part to get right, because the failure modes are asymmetric.

| Token | Absent | Malformed or expired |
|---|---|---|
| `age_verification_token` | `requires_buyer_input` | `requires_buyer_input` — **never a silent pass** |
| `buyer_token` | Volume evaluated on this basket alone | `422 INVALID_BUYER_TOKEN` |

A bad age token is quiet: it simply means age is not verified, which the contract already
defines as a resolvable state rather than a refusal.

A bad buyer token is loud. Silently ignoring it would evaluate the volume limit against an
empty history and quietly bias the answer toward an optimistic `eligible`. A wrong answer
that looks right is worse than a rejection.

**Omitting `buyer_token` entirely is legitimate but lossy.** Without it there is no way to
count what the buyer has already received this period, so an `eligible` may be optimistic.
Send it when you have it.

### What is never accepted

- **A date of birth.** The token exists precisely so a DOB never crosses the wire.
- **An email address.** Identity is opaque or absent.
- **An account or merchant UUID.** See above.

## Errors

| Code | Meaning | Client should |
|---|---|---|
| `401 UNAUTHORIZED` | Missing or invalid credential | Stop. Do not retry. |
| `402 PAYMENT_REQUIRED` | Account past due or suspended | Stop. Surface to the integrator, never the shopper. |
| `403 SELLER_NOT_AUTHORIZED` | `seller_ref` is not the authenticated account | Fix the caller. |
| `422 UNPROCESSABLE_REQUEST` | Understood but not processable | Fix the request. Do not retry unchanged. |
| `422 INVALID_BUYER_TOKEN` | `buyer_token` failed verification | Re-mint or omit the token. |
| `429` | Rate limited; honours `Retry-After` | Back off. |
| `5xx` | Server error | Treat **exactly** as `retry`. Never as an outcome. |

Error bodies are `{ "error": { "code": "...", "message": "..." } }`. The `code` is stable
and safe to branch on. The `message` is not, and should not be parsed.

## Handling keys

- Secret keys server-side only. An agent's prompt context is not server-side.
- Rotate on any suspicion. There is no recovery path for a leaked key, by design.
- A `401` is not retryable. Retrying a bad credential in a loop is how you get rate
  limited on top of being unauthorized.
