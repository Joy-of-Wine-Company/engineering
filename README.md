# Compliance Vine — Engineering

Public engineering pages for **Compliance Vine** (Joy of Wine Company Inc.), the compliance automation platform for direct-to-consumer wine shipping.

**Live site:** https://joy-of-wine-company.github.io/engineering/

| Page | What it is |
|---|---|
| [Distance to Five Nines](https://joy-of-wine-company.github.io/engineering/) | The data-quality program: measuring compliance rule data like SREs measure uptime, with the unedited climb log. |
| [Platform architecture](https://joy-of-wine-company.github.io/engineering/platform.html) | One intelligence engine behind every compliance decision — what the data powers. |
| [How we test a compliance engine](https://joy-of-wine-company.github.io/engineering/testing.html) | The QA discipline: external ground truth, adversarial bypass testing, parity, resilience, evidence. |
| [Fail-safe by design](https://joy-of-wine-company.github.io/engineering/failsafe.html) | How the system fails on purpose: legality fails closed into review, tax fails open to a floor that never over-collects. |
| [Before an AI recommends the bottle](https://joy-of-wine-company.github.io/engineering/agentic-commerce.html) | Pre-sale compliance for shopping agents: what inverts when there is no human backstop, and the read-only contract published before the endpoint was built. |
| [Case study (PDF)](https://joy-of-wine-company.github.io/engineering/compliance-savings-case-study.pdf) | ~$160K combined first-year labor + software savings for one multi-state shipper; 15+ months in production. |

## Machine-readable

The agentic-commerce memo is published for machines as well as people — a shopping agent
needs the contract, not the prose.

| Artifact | What it is |
|---|---|
| [`agentic-commerce.md`](agentic-commerce.md) | The memo as plain Markdown. |
| [`docs/presale-verification.md`](docs/presale-verification.md) | The pre-sale contract: request and response shapes, reason codes, errors, safe paraphrasing. |
| [`docs/verdicts-and-reason-codes.md`](docs/verdicts-and-reason-codes.md) | Every status and reason code, and how the pre-sale vocabulary maps onto the checkout-time one. |
| [`docs/authentication.md`](docs/authentication.md) | Credentials, the authorization model, and the opaque token format. |
| [`examples/`](examples/) | Five worked cases: eligible, ineligible, age-required, review, timeout. |
| [`openapi/cv-live-presale-v1.json`](openapi/cv-live-presale-v1.json) | OpenAPI 3.1 draft contract. Declares no `servers` — the endpoint is not deployed. |
| [`llms.txt`](llms.txt) | Curated index of this site for language models. |

The product source code is private. These pages are the public record of the engineering discipline behind it — internal identifiers, credentials, customers, and infrastructure details are redacted by design.

**Contact:** Christopher Anderson · curator@joyofwine.co · [joyofwine.co](https://joyofwine.co)
