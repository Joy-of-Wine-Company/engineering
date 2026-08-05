# Verdicts and reason codes

There are **two vocabularies**. They are deliberately different, and conflating them is a
bug. This document is the mapping between them.

## Checkout-time (the enforcement API)

`POST /api/v1/compliance/check` — used by a merchant's storefront when an order exists.

| Verdict | Meaning |
|---|---|
| `PASS` | No blocking violation. |
| `BLOCK` | At least one `hard_block` violation. |
| `QUARANTINE` | At least one `soft_block` violation, **or** the service timed out. |

There is a second field, `effective_verdict`, which accounts for the merchant's
enforcement mode. In `shadow` or `disabled` mode it is always `PASS` — the engine logs
what it *would* have done without interrupting fulfillment.

> **Note on stale prose.** Some older material says `ALLOW` / `DENY` or `ALLOW` / `HOLD`.
> The illustrative demo on joyofwine.co's homepage uses `ALLOW` / `HOLD` / `BLOCK` and
> disclaims itself as a sample rule pack. The real enum is `PASS` / `BLOCK` / `QUARANTINE`.

### Violation types

`state_not_allowed`, `age_not_verified`, `volume_exceeded`, `license_expired`,
`license_missing`, `carrier_not_allowed`, `dry_county`, `service_timeout`

Severities: `hard_block`, `soft_block`, `warning`.

Worth knowing: **every rule violation today is `hard_block`.** `QUARANTINE` is currently
reachable only through the timeout and error paths, not through any rule.

## Pre-sale (the agent API)

`POST /api/v1/presale/verify` — used before an order exists. Five statuses, no
`effective_verdict`, no enforcement mode. An agent is not a merchant and has no shadow
mode to be in.

| Status | Terminal |
|---|---|
| `eligible` | Until `expires_at` |
| `ineligible` | Yes |
| `requires_buyer_input` | No |
| `requires_merchant_review` | No |
| `retry` | No — and not a verdict at all |

## The mapping

| Checkout violation | `details.reason` | Pre-sale status | Pre-sale `reason_code` |
|---|---|---|---|
| `state_not_allowed` | `dtc_prohibited` | `ineligible` | `destination_dtc_prohibited` |
| `state_not_allowed` | `merchant_not_configured` | `ineligible` | `seller_not_licensed_for_destination` |
| `license_missing` | `no_license` | `ineligible` | `seller_license_missing` |
| `license_expired` | `license_inactive` | `ineligible` | `seller_license_inactive` |
| `license_expired` | `license_expired` | `ineligible` | `seller_license_expired` |
| `dry_county` | `dry_county` | `ineligible` | `destination_dry_jurisdiction` |
| `carrier_not_allowed` | `no_carrier_service` | `ineligible` | `no_carrier_coverage` |
| `volume_exceeded` | — | `ineligible` | `volume_limit_exceeded` |
| `age_not_verified` | — | **`requires_buyer_input`** | `age_verification_required` |
| any `soft_block` | — | `requires_merchant_review` | `manual_review_required` |
| `service_timeout` | — | **`retry`** | `upstream_timeout` |
| (unexpected failure) | — | **`retry`** | `upstream_error` |

Two details that matter when implementing against this:

1. **`license_expired` covers two distinct answers.** An inactive licence and an expired
   licence share the same violation `type`, distinguished only by `details.reason`. Keying
   on `type` alone collapses them and tells the shopper the wrong thing.
2. **The two bolded rows are the inversions.** They are the reason the pre-sale endpoint
   exists as a separate contract rather than a filter over the checkout one.

## Reason-code naming

The prefix carries meaning. Read it before writing shopper-facing copy.

| Prefix | Scope | Safe generalisation |
|---|---|---|
| `seller_*` | This seller only | **None.** Another seller may be fine to the same address. |
| `destination_*` | The destination | Applies to the destination regardless of seller. |
| `volume_*` | This buyer at this destination | Depends on the buyer's history, not the law alone. |
| `age_*` | This buyer | Resolvable. Not a refusal. |
| `upstream_*` | Our service | Says nothing about the sale. |

Getting this wrong is the single most damaging integration bug available:
`seller_license_missing` rendered as "wine can't ship to your state" is false, defames the
jurisdiction, and hides every compliant seller in the results.

## `checks_performed`

Both APIs report the same six names, in the same fixed order, so a pre-sale decision and a
checkout-time decision can be compared directly:

`state_allowed`, `license_valid`, `age_verified`, `volume_limit`, `carrier_allowed`,
`dry_county`

A check absent from the list did not run, and its subject was not evaluated. On `retry`
the list is empty.

## Stability

The pre-sale status values and reason codes are a **published, frozen interface**. Codes
may be added; existing ones will not be renamed or repurposed. An integration should
handle an unknown `reason_code` by falling back to the generic message for its `status`
rather than failing.

Messages (`message`, `description`) are **not** stable and should never be parsed.
