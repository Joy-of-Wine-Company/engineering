# Before an AI recommends the bottle

*An engineering memo — Compliance Vine, Joy of Wine Company · August 2026*

Machine-readable version of
https://joy-of-wine-company.github.io/engineering/agentic-commerce.html

The [platform memo](platform.html) says CV Live is the bouncer at checkout. This one is
about a question that arrives **earlier** — from a shopping agent, before a human ever
sees the cart — and what has to change when the thing asking is a machine with no human
behind it.

## The problem in one paragraph

Wine is one of the few categories where a **good recommendation can be unlawful to
fulfill**. A shopping agent that reads a catalogue cannot tell. The answer is not a
property of the bottle — it is a property of the seller, the destination below state
level, and the buyer's purchase history over a rolling period. All three are invisible to
the agent, and getting it wrong costs a winery its license, not a return.

## Three things an agent cannot infer

- **The seller.** Licenses are held per-winery, per-state. Two wineries listing the same
  bottle can have opposite answers for the same address.
- **The destination, below state level.** Dry counties and carrier coverage resolve at ZIP
  and FIPS, not at the state. A state-level answer is not precise enough to act on.
- **The buyer's history.** Most states cap what one household may receive in a rolling
  period — counting purchases the agent never saw.

A generic "does wine ship to Virginia" lookup answers none of them. An agent that
substitutes one for the other is wrong in a way that reaches a regulator.

## What changes when there is no human backstop

Our checkout-time engine has a documented, deliberate posture: [fail open](failsafe.html).
If the engine times out, the order proceeds flagged for review. That is correct — a
merchant should never lose revenue to *our* downtime, and a person reviews the order
before anything ships.

Move that same behaviour in front of an agent and it stops being safe. There is no review
queue. There is no person. A permissive answer during an outage becomes a recommendation
to buy a bottle that cannot lawfully be delivered, and nothing downstream catches it.

So two behaviours invert. Not because the old ones were wrong, but because the assumption
underneath them — that a human sees the result — no longer holds.

| | At checkout (order exists) | Pre-sale (no order yet) |
|---|---|---|
| **Missing age signal** | A hard block. The order exists; the safe move is to stop it. | `requires_buyer_input`. The honest answer is "not yet", not "no". |
| **Timeout** | A permissive verdict flagged for review. Revenue is never blocked by our downtime. | `retry` — the absence of a verdict. No path from a failed dependency to a positive answer. |

## One status. Five values. Nothing to interpret.

| Status | Meaning | What an agent may say |
|---|---|---|
| `eligible` | Every check passed for this seller, destination and basket. | "This seller can ship this to your address." |
| `ineligible` | A check failed in a way the buyer cannot resolve. | "This seller can't ship this to your address." |
| `requires_buyer_input` | Possible, but something is missing from the buyer. | "You'll need to verify your age before this can ship." |
| `requires_merchant_review` | The decision needs a human at the merchant. | "The seller needs to review this before it can proceed." |
| `retry` | **No decision was reached.** Transient condition upstream. | "I couldn't check that just now — let me try again." |

## The two rules that actually prevent harm

**Never present `retry` as a compliance outcome.** It carries no information about whether
the sale is permitted. `checks_performed` comes back empty because nothing was evaluated.
An agent that renders it as "this wine can't ship to you" has invented a legal conclusion
out of an outage.

**Never generalise a seller-specific answer.** `ineligible` for one winery says nothing
about another winery shipping to the same address. An agent that turns one seller's
missing license into "wine can't be shipped to Kansas" has misinformed the shopper,
misstated the state's law, and suppressed every compliant seller in the results. This is
the most likely integration bug, and it is why the reason codes are named `seller_*` and
`destination_*` rather than being flat.

## Four invariants

1. **An outage never returns `eligible`.** Timeouts and upstream failures return `retry`.
   There is no code path from a failed dependency to a positive answer, and the mapper
   refuses a timeout outright rather than letting one become a verdict.
2. **A missing age signal never returns `eligible`.** It returns `requires_buyer_input`
   with the action that would resolve it. An absent, malformed, or expired token all
   degrade the same way — never a silent pass.
3. **The check is read-only.** It never places an order, reserves inventory, takes
   payment, or becomes merchant of record.
4. **A decision is a snapshot.** Licenses lapse, states amend rules, and a buyer's rolling
   volume moves with every other purchase. `expires_at` is binding, and checkout
   re-verifies.

## Where it sits

```
agent builds cart
        │
        ▼
POST /api/v1/presale/verify        ← read-only, no order exists yet
        │
        ├── ineligible ──────────► do not offer this item from this seller
        ├── requires_buyer_input ─► collect the missing signal, verify again
        ├── retry ───────────────► back off; state nothing about eligibility
        └── eligible ────────────► proceed to the merchant's checkout
                                          │
                                          ▼
                          merchant creates the order, runs its own
                          checkout-time compliance check, takes payment,
                          and fulfills — as merchant of record
```

The pre-sale call informs the agent. It does not authorise anything, and it moves no
responsibility off the merchant — which is also how the major agentic-commerce protocols
allocate it. Joy of Wine Company is never a party to the transaction.

## Privacy: what the request refuses to carry

- **No email address.** A pre-sale query happens before a customer exists. Accepting an
  address would mean receiving personal data for a sale that may never occur, from a party
  that is not the merchant.
- **No date of birth.** Age verification travels as an opaque token. This mirrors what the
  service already stores at checkout — a verification result and its method, never the
  underlying date.
- **No account identifier.** The authorized account comes from the credential. A
  body-supplied account id is an authorization bypass waiting for the day someone forgets
  to cross-check it, so the field does not exist.

## Testing it

Two different things need testing, and they fail differently. The endpoint is
deterministic — fixture in, status out, five golden cases plus explicit tests for each
invariant above. The *agent* is not. An endpoint that is perfectly correct and an agent
that paraphrases `retry` as "this wine can't ship to you" still leaves a shopper holding
something false.

So the eval suite scores what the agent **says**, given a fixed response — including
adversarial cases: a multi-seller basket where one seller is eligible and one is not,
three retries in a row, and an `ineligible` with no remedy attached, where the failure mode
is an agent inventing one. The same discipline as
[how we test the compliance engine](testing.html), pointed at language instead of logic.

## Status

**The contract is published; the endpoint is not built.**
`POST /api/v1/presale/verify` is not deployed, and the OpenAPI document deliberately
declares no server — there is nothing to call, and a placeholder would only invite
requests that cannot be served.

Publishing in this order is the point: a contract is cheapest to correct before there is
an implementation defending it. If you are integrating agentic checkout for alcohol and
something here is wrong, that is the most useful thing you can tell us.

## The closing thought

The checkout question and the pre-sale question are the same question, asked at different
times: **"can this bottle legally ship here?"** What changes is who is listening. A person
can hold "I'm not sure yet". A machine will round it to yes or no — so the contract has to
make **"I don't know"** a first-class answer that cannot be mistaken for either.

---

**Machine-readable companions**

- [`docs/presale-verification.md`](docs/presale-verification.md) — the full contract
- [`docs/verdicts-and-reason-codes.md`](docs/verdicts-and-reason-codes.md) — every code
- [`docs/authentication.md`](docs/authentication.md) — credentials and tokens
- [`examples/`](examples/) — five worked cases
- [`openapi/cv-live-presale-v1.json`](openapi/cv-live-presale-v1.json) — OpenAPI 3.1 draft
- [`llms.txt`](llms.txt) — index for language models

**Companion memos:** [Distance to Five Nines](./) · [platform architecture](platform.html)
· [how we test](testing.html) · [fail-safe by design](failsafe.html)

**Contact:** curator@joyofwine.co · [joyofwine.co](https://joyofwine.co)
