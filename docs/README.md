# Documentation

Long-form architecture, threat-model, and compliance reasoning that backs the opinionated choices made elsewhere in this pack. Where the README states a principle in one line, the documents in this directory explain why.

## Published

| File | Topic |
|---|---|
| [`the-trust-gap.md`](the-trust-gap.md) | **Essay.** How Europe's short-term rental compliance silently breaks at the channel layer, and what kind of infrastructure is required to fix it. The narrative reasoning that motivates the architectural choices in this pack. Canonical version at [get-obra.com/the-trust-gap](https://get-obra.com/the-trust-gap). |
| [`ARCHITECTURE.md`](ARCHITECTURE.md) | **Architecture.** The engineer-readable mechanism document. Defends the four architectural properties from the essay with case-by-case analysis of why weaker alternatives fail. Covers the connector framework (typed actions, tier classification, drift detection, idempotency), the three-gate trust pillar (structured reasoning + adversarial review + human confirmation), the pre-proposal extraction agreement gate (visual vs. MRZ on passports), the audit chain (hash-chained, customer-owned, redacted-by-construction telemetry), and the action authorization model. |

## Planned documents

| File | Topic |
|---|---|
| `THREAT-MODEL.md` | The adversaries this pack assumes (operator-error, prompt injection, scan tampering, drift, supply-chain on the SEF portal). What the pack defends against natively versus what it relies on the runtime to defend. |
| `COMPLIANCE-NOTES.md` | GDPR Article 9 reasoning, SEF reporting obligations under Decreto-Lei n.º 128/2014, AT month-end requirements, CNPD posture. Written by a non-lawyer for non-lawyers; review by qualified counsel encouraged before relying on any of it. |
| `DRIFT-PLAYBOOK.md` | What to do when the SEF portal changes its DOM, when an issuing country's passport format changes, when a booking platform updates its API. The connector framework halts on drift; this playbook is the human's response. |

## Order of arrival

`ARCHITECTURE.md` lands first because it is the deepest dependency: every other document references the architectural decisions documented there.

## Status

Active development. `ARCHITECTURE.md` is the first document being written; the others follow.
