# Pre-sale verification — agent contract

**Status:** Draft contract. Published for review.
**Endpoint:** `POST /api/v1/presale/verify`
**Availability:** Private beta. Not yet deployed. No base URL is published, and no
credentials are issued self-serve. Request access at info@joyofwine.co.
**Version:** v1 (draft)
**Last updated:** 2026-08-04
**Canonical page:** https://joyofwine.co/developers/agentic-commerce
**OpenAPI:** https://joyofwine.co/openapi/cv-live-presale-v1.json

---

## What this answers

Given a **specific seller**, a **destination**, and a **basket**, may this alcohol sale
proceed — and if not, why, and what would fix it.

This is not a shipping-availability lookup. The answer changes per seller, because
alcohol licenses are held per-winery and per-state; per destination, because some
counties are dry and some carriers will not deliver alcohol to them; and per buyer,
because most states cap how much one household may receive in a rolling period.

A generic "does wine ship to Virginia" answer is not this answer, and an agent that
substitutes one for the other will be wrong in a way that costs the merchant a license.

## What this is not

The endpoint is **read-only**. It does not create an order, reserve inventory, take
payment, calculate tax, or become the merchant of record. Order creation, payment,
fulfillment, tax, risk, and acceptance all remain with the merchant. Joy of Wine Company
is never a party to the transaction.

---

## Request

```http
POST /api/v1/presale/verify
Authorization: Bearer <api_key>
Content-Type: application/json
```

```json
{
  "seller_ref": "sel_9fbc2a41",
  "destination": {
    "country": "US",
    "state": "VA",
    "postal_code": "23454"
  },
  "items": [
    {
      "sku": "JOW-CAB-2021",
      "quantity": 2,
      "volume_ml": 750,
      "abv": 13.5,
      "category": "still"
    }
  ],
  "buyer_token": "bt_opaque_value",
  "age_verification_token": "avt_opaque_value"
}
```

| Field | Required | Notes |
|---|---|---|
| `seller_ref` | yes | Opaque identifier for a seller **within the authorized account**. The account itself is derived from the credential — see below. |
| `destination.country` | yes | `US` only in v1. |
| `destination.state` | yes | Two-letter code. 50 states plus `DC`. |
| `destination.postal_code` | yes | 5-digit or ZIP+4. Required because dry-jurisdiction and carrier coverage resolve below the state level. |
| `items[].sku` | yes | Seller's own identifier. |
| `items[].quantity` | yes | Positive integer. |
| `items[].volume_ml` | no | Defaults to `750`. Drives volume-limit arithmetic. |
| `items[].abv` | no | Percent by volume. |
| `items[].category` | no | `still`, `sparkling`, `dessert`, `fortified`, `cider`. |
| `buyer_token` | no | Opaque. Lets the rolling volume limit account for this buyer's prior purchases. Omitting it means volume is evaluated on this basket alone, which can produce an optimistic `eligible`. A **malformed** token is rejected with `422 INVALID_BUYER_TOKEN` rather than ignored — see [Errors](#errors). |
| `age_verification_token` | no | Opaque proof that age verification has been completed. See [Age verification](#age-verification). |

### What the request must not contain

- **No `customer_email`.** Identity is carried by opaque tokens.
- **No raw date of birth.** Only a verification token, never the underlying value.
- **No merchant or account UUID.** The authorized account is derived from the bearer
  credential and never from the request body. A body-supplied account identifier is an
  authorization bypass waiting to happen; the field does not exist.

---

## Response

Always `200` when a decision was reached. Transport and authorization failures use the
status codes in [Errors](#errors).

```json
{
  "decision_id": "dec_01J8XQ4M2N7VZ",
  "status": "requires_buyer_input",
  "reason_codes": ["age_verification_required"],
  "next_actions": [
    {
      "type": "collect_age_verification",
      "description": "Complete age verification for the buyer, then re-verify with the resulting token."
    }
  ],
  "checks_performed": [
    "state_allowed",
    "license_valid",
    "age_verified",
    "volume_limit",
    "carrier_allowed",
    "dry_county"
  ],
  "rule_version": {
    "destination": "VA",
    "version": "2026-07-01"
  },
  "checked_at": "2026-08-04T14:22:03Z",
  "expires_at": "2026-08-04T14:37:03Z"
}
```

| Field | Notes |
|---|---|
| `decision_id` | Stable identifier. Quote it in support requests; it resolves to the retained decision record. |
| `status` | Exactly one of the five values below. This is the only field an agent should branch on. |
| `reason_codes` | Zero or more stable codes. Never localised, never reworded. Present for every non-`eligible` status. |
| `next_actions` | What would change the outcome, when anything can. Empty for terminal `ineligible`. |
| `checks_performed` | Which checks actually ran. Names match the enforcement engine exactly. A check absent from this list did not run and its subject was not evaluated. |
| `rule_version` | Which version of the destination's rule set produced this decision. |
| `checked_at` | When the decision was made. |
| `expires_at` | When it stops being usable. Binding — see [Decisions expire](#decisions-expire). |

### `checks_performed` values

These are the six checks the enforcement engine runs, named identically so that a
pre-sale decision and a checkout-time decision can be compared directly:

`state_allowed`, `license_valid`, `age_verified`, `volume_limit`, `carrier_allowed`,
`dry_county`

---

## The five statuses

| `status` | Meaning | Terminal? |
|---|---|---|
| `eligible` | Every check passed for this seller, destination, and basket, as of `checked_at`. | Until `expires_at`. |
| `ineligible` | A check failed in a way the buyer cannot resolve. | Yes. |
| `requires_buyer_input` | The sale may be possible, but something is missing from the buyer. | No — retry after `next_actions`. |
| `requires_merchant_review` | The decision needs a human at the merchant. | No — out of band. |
| `retry` | **No decision was reached.** Transient condition upstream. | No — retry per `next_actions`. |

`retry` is not a verdict. It is the absence of one.

---

## Reason codes

Every code maps onto a specific check in the enforcement engine, so a pre-sale answer and
the eventual checkout answer share one vocabulary.

| `reason_code` | Status | Check | Meaning |
|---|---|---|---|
| `destination_dtc_prohibited` | `ineligible` | `state_allowed` | The destination state does not permit direct-to-consumer wine shipment at all. |
| `seller_not_licensed_for_destination` | `ineligible` | `state_allowed` | The destination is not among the seller's configured shipping states. |
| `seller_license_missing` | `ineligible` | `license_valid` | No license on file for the destination state. |
| `seller_license_expired` | `ineligible` | `license_valid` | The seller's license for that state has lapsed. |
| `seller_license_inactive` | `ineligible` | `license_valid` | The license exists but is not currently active. |
| `destination_dry_jurisdiction` | `ineligible` | `dry_county` | The destination is in a dry or partially dry jurisdiction where alcohol delivery is prohibited. |
| `no_carrier_coverage` | `ineligible` | `carrier_allowed` | No carrier will lawfully deliver alcohol to that destination. |
| `volume_limit_exceeded` | `ineligible` | `volume_limit` | The basket would exceed the destination's rolling volume limit for this buyer. |
| `age_verification_required` | `requires_buyer_input` | `age_verified` | Age verification has not been completed for this buyer. |
| `manual_review_required` | `requires_merchant_review` | any | The decision was referred to the merchant. Reserved; not emitted in v1. |
| `upstream_timeout` | `retry` | — | A dependency did not answer in time. **No checks were completed.** |
| `upstream_error` | `retry` | — | An unexpected failure occurred. **No checks were completed.** |

### `next_actions` types

| `type` | Paired with | Meaning |
|---|---|---|
| `collect_age_verification` | `age_verification_required` | Run age verification, then re-verify with the resulting `age_verification_token`. |
| `reduce_quantity` | `volume_limit_exceeded` | Carries a `max_quantity` when a smaller basket would pass. Absent when no quantity would. |
| `retry_after` | `upstream_timeout`, `upstream_error` | Carries `retry_after_seconds`. |
| `contact_merchant` | `manual_review_required` | Out-of-band resolution. |

---

## Two deliberate inversions from the enforcement API

The existing checkout-time API (`POST /api/v1/compliance/check`) is an order-integration
contract. It is correct for its purpose and wrong for an agent's, in two specific ways.
Both are inverted here, and this is the main reason a separate endpoint exists.

### 1. A missing age signal is not a block — it is a question

At enforcement time, unverified age is a hard block, because by then the order exists and
the safe action is to stop it. An agent is earlier in the flow, where the honest answer is
not "no" but "not yet".

- Enforcement API: unverified age → `BLOCK`.
- Pre-sale: unverified age → **`requires_buyer_input`** with `age_verification_required`.

An agent that renders a hard "no" here has told the shopper a sale is impossible when it
is one step away from possible.

### 2. A timeout is never an approval

At enforcement time the service fails open: a timeout returns an internal verdict of
`QUARANTINE` but an `effective_verdict` of `PASS`, so the order flows and a human reviews
it before fulfillment. There is a human backstop, so failing open is safe.

An agent has no such backstop. If an outage returned something an agent paraphrases as
"eligible", the failure mode is a recommendation to buy a bottle that cannot lawfully be
delivered — with no one downstream to catch it.

- Enforcement API: timeout → `effective_verdict: PASS`.
- Pre-sale: timeout → **`retry`**, which is not a verdict.

**An outage never produces `eligible` on this endpoint.** There is no code path from a
failed dependency to a positive answer.

---

## Invariants

These hold for every response. An integration may rely on them.

1. **An outage never returns `eligible`.** Timeouts and upstream failures return `retry`.
2. **A missing age signal never returns `eligible`.** It returns `requires_buyer_input`.
3. **The endpoint is read-only.** It never places an order, takes payment, or becomes
   merchant of record.
4. **A decision is a snapshot.** It is valid until `expires_at` and no longer.

---

## What an agent may say

`status` is the only field to branch on. The wording below is safe; the "must not" column
is what produces a false statement.

| `status` | Safe to tell a shopper | Must not say |
|---|---|---|
| `eligible` | "This seller can ship this to your address." | "Your order is confirmed." Nothing has been ordered. |
| `ineligible` | "This seller can't ship this to your address," plus the reason if useful. | "Wine can't be shipped to your state." The limit is usually this seller's, not the state's. |
| `requires_buyer_input` | "You'll need to verify your age before this can ship." | "This can't be shipped." It can, once the step is completed. |
| `requires_merchant_review` | "The seller needs to review this order before it can proceed." | "This is approved" or "this is rejected." Neither has happened. |
| `retry` | "I couldn't check that just now — let me try again." | Anything about eligibility. No check completed. |

Two rules underneath that table:

- **Never present `retry` as a compliance outcome.** It carries no information about
  whether the sale is permitted.
- **Never generalise a seller-specific answer to the whole category.** `ineligible` for
  one winery says nothing about another winery shipping to the same address, and saying
  otherwise misinforms the shopper and defames the destination's law.

---

## Age verification

The endpoint accepts an opaque `age_verification_token`. It never accepts a date of birth.

This mirrors what the service stores: a verification boolean and the method used, never
the underlying date. A token that is absent, malformed, or expired produces
`requires_buyer_input`, never a silent pass.

---

## Decisions expire

`expires_at` exists because the inputs move. Licenses lapse, states amend their rules, and
a buyer's rolling volume changes with every other purchase they make.

- Treat a decision as stale after `expires_at`, without exception.
- Re-verify at checkout even inside the window. A pre-sale `eligible` is a good-faith
  signal for building a cart, not an authorisation to fulfill.
- The binding decision is the one made at checkout, by the merchant, through the
  enforcement API.

---

## Errors

| HTTP | `error.code` | Meaning | Agent should |
|---|---|---|---|
| `401` | `UNAUTHORIZED` | Missing or invalid credential. | Stop. Do not retry. |
| `402` | `PAYMENT_REQUIRED` | Account past due or suspended. | Stop. Surface to the integrator, not the shopper. |
| `403` | `SELLER_NOT_AUTHORIZED` | `seller_ref` is not the account the credential identifies. | Fix the caller. Never surface to the shopper. |
| `422` | `UNPROCESSABLE_REQUEST` | Understood but not processable — unsupported country, malformed postal code, unknown field. | Fix the request. Do not retry unchanged. |
| `422` | `INVALID_BUYER_TOKEN` | `buyer_token` failed verification. | Re-mint the token, or omit it and accept a basket-only volume evaluation. |
| `429` | — | Rate limited. Honours `Retry-After`. | Back off. |
| `5xx` | — | Server error. | Treat exactly as `retry`. Never as an outcome. |

Error bodies carry `{ "error": { "code": "...", "message": "..." } }`. The `code` is
stable; the `message` is not, and should not be parsed.

### `details` on a 422

A `422` adds a `details` array naming the fields that failed. It is diagnostic —
useful in a log, not something to branch on.

```json
{
  "error": {
    "code": "UNPROCESSABLE_REQUEST",
    "message": "Invalid request",
    "details": [
      { "field": "destination.country", "message": "Invalid input: expected \"US\"" },
      { "field": "items.0.quantity", "message": "Too small: expected number to be >0" }
    ]
  }
}
```

`field` is a dotted path into the request. A rejected **unknown top-level key** —
the case where you sent something the contract does not accept, such as an email
address — has no path of its own and is reported against `(request)`.

### Why a `seller_ref` mismatch is 403 and not 404

The account is resolved from the credential. `seller_ref` may only *confirm* it,
and a value naming anything else is refused rather than looked up. That is
deliberate: an endpoint that answered questions about accounts other than the
caller's own would leak which sellers exist and which states they are licensed
in. There is no enumeration surface here, so there is nothing to hide behind a
404.

### Why a bad `buyer_token` is loud and a bad age token is quiet

The two tokens fail differently on purpose.

A malformed `age_verification_token` degrades to `requires_buyer_input`. That is a real,
actionable answer: age is not verified, and the buyer can fix it. Nothing is lost by
treating a broken token the same as an absent one.

A malformed `buyer_token` is rejected outright. Silently ignoring it would evaluate the
rolling volume limit against an empty purchase history and bias the answer toward an
optimistic `eligible` — a wrong answer that looks exactly like a right one, with nothing
in the response to indicate the degradation happened. An error you can see beats a
number you cannot check.

Omitting `buyer_token` entirely remains legitimate. The contract already warns that doing
so evaluates volume on the basket alone. The difference is that omission is a choice you
made, and a malformed token is a bug you have not noticed yet.

---

## Where this sits in an agentic checkout

```
agent builds cart
        │
        ▼
POST /api/v1/presale/verify        ← read-only, no order exists yet
        │
        ├── ineligible ──────────► do not offer this item from this seller
        ├── requires_buyer_input ─► collect the missing signal, verify again
        ├── retry ───────────────► back off and retry; state nothing about eligibility
        └── eligible ────────────► proceed to the merchant's checkout
                                          │
                                          ▼
                          merchant creates the order, runs its own
                          checkout-time compliance check, takes payment,
                          and fulfills — as merchant of record
```

The pre-sale call informs the agent. It does not authorise anything, and it does not move
any responsibility off the merchant.

---

## Getting access

The endpoint is not deployed. This document is published so the contract can be reviewed
before it is built, and so integrators can tell us where it is wrong.

Request beta access, or send contract feedback, to **info@joyofwine.co**.
