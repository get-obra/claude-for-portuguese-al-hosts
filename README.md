# Claude for Portuguese AL hosts

> A reference pack for building Claude-powered automation for Portuguese vacation rental hosts ("Alojamento Local" operators) under EU and Portuguese regulatory obligations.

Maintained by [Obra](https://get-obra.com). Released under Apache 2.0.

> **Status:** alpha, in active development. This README describes the intended shape of the pack as it ships. Skill content, connector implementation, and example transcripts arrive over the coming weeks. See [Status](#status) for the timeline.

> **Start here.** The two depth pieces, in reading order:
> 1. *[The trust gap](docs/the-trust-gap.md)* — the narrative essay on why this work exists. Canonical version at [get-obra.com/the-trust-gap](https://get-obra.com/the-trust-gap). ~10 min read.
> 2. *[Architecture](docs/ARCHITECTURE.md)* — the engineer-readable mechanism behind the essay's claims. Defends the four architectural properties, documents the connector framework, the three-gate trust pillar, the pre-proposal extraction agreement gate, and the audit chain. ~20 min read.

---

## Why this exists

A vacation rental host in Portugal opens her phone at six in the morning. A booking arrived overnight from a guest who speaks French. Before the guest arrives in seventy-two hours, she has to:

1. Send a welcome email in French, written in her voice, asking for the guest's passport scan.
2. Read the passport when it arrives, extract the identity data, and submit it to the **SEF** (Serviço de Estrangeiros e Fronteiras, the Portuguese border authority) within twenty-four hours of arrival. This is **mandatory under Portuguese tourism law**. Missed submissions carry substantial fines under Decreto-Lei n.º 128/2014.
3. Answer the guest's questions during the stay, in French, including which restaurants nearby, where the keys go, and whether the wifi works after the third time you reset it.
4. After check-out, generate the invoice, hand a clean record to her accountant, and schedule the cleaning crew.

She manages most of this today on translated emails that read as cold, a one hundred and fifty euro a month accountant who only handles invoices, and the slow accumulation of guilt every time a deadline gets close.

This is the kind of work Claude does extremely well. The problem is not capability. The problem is that:

- The guest's passport is **GDPR Article 9 special-category data**. It cannot be casually sent to a hosted AI service.
- The host needs an **audit trail** she can hand to the tax authority or the data regulator on demand.
- The host has never written a line of code and is not going to wire up a multi-tool agentic workflow herself.

This repository is the open-source reference pack for that shape of work. It provides the skills, MCP connector contracts, example transcripts, and architectural guidance for building Claude-powered automation that meets Portuguese AL hosts where they are, with the compliance posture their regulators require.

---

## How this pack frames the work

**Obra reduces an operator's workload to verification.** The AI does the labor — drafting emails, reading scans, filling forms, generating records. The human keeps the decisions — approving the email before it sends, confirming the extracted passport numbers, signing off on the figures the accountant will file.

This is not a fire-and-forget automation pack. It is a labor-reduction-with-verification-kept-human pack. Every skill in this pack assumes a human reviewer is in the loop on anything that touches the regulator, the guest's money, or the operator's reputation. The skill files are explicit about which tier each action lives in (autonomous, gated, forbidden) and where the human's sign-off lands in the flow.

> *Routine steps just get done. Anything that needs you reaches you.*

---

## The skills

| # | Skill | What it does | Regulatory anchor |
|---|---|---|---|
| 01 | Pre-arrival welcome | Drafts the welcome email in the guest's native language, in the host's voice. Host approves before send. | Portuguese tourism law (Decreto-Lei n.º 128/2014) |
| 02 | Passport extraction | Reads a passport scan; a second independent read cross-checks the numbers; host signs off on the result. | GDPR Article 9 (special-category data handling) |
| 03 | SEF submission | Drives the SEF reporting portal to submit the host-approved identity report within the legally mandated twenty-four-hour window. | SEF reporting obligation (24h post-arrival) |
| 04 | In-stay support | Handles guest questions in their language during the stay, escalates judgment calls to the host, never invents policy. | Hospitality standard of care |
| 05 | Accountant handoff | Generates the structured monthly record; host approves the figures; accountant signs off. Each reservation carries an audit-chain pointer. | Portuguese tax authority (AT) reporting + GDPR data-minimization |

Each skill is designed to be **composed by an agentic runtime** (your own, or a managed-service provider like Obra) rather than to run standalone. The skills do not assume any particular orchestration framework. They assume Claude is the reasoning core, and that whatever runtime is calling them is responsible for credential handling, audit logging, and human-in-the-loop gating.

---

## Design principles

The pack is opinionated about a small number of architectural choices that we believe are non-negotiable for this customer shape.

**Local-first.** Customer data does not need to leave the customer's premises for these workflows to function. Claude is invoked from the customer's machine over an authenticated API connection. Passport scans, guest correspondence, and reservation history stay on the host's hardware.

**Typed actions over free-text steps.** Every action a skill can take is declared in a typed contract with explicit tier classification (autonomous, gated, forbidden). A workflow cannot smuggle a novel action past human review by phrasing it cleverly.

**Pre-proposal agreement on critical extractions.** On extraction-heavy actions like reading a passport scan or parsing an invoice, two independent Claude passes must agree on the extracted payload before the action enters the review pipeline. If they disagree, the action halts and lands in the human's hands for manual verification. This catches the class of hallucination that traditional adversarial review cannot reach — where both Claude invocations reasoning over the same proposed payload could silently agree on a misread.

**Three independent gates on risky actions.** Anything that touches the regulator, the guest's money, or external systems passes through three checks: structured reasoning by Claude, adversarial review by a second Claude invocation, and human confirmation. Hallucinations cannot act unilaterally.

**Audit by construction.** Every action emits an audit event whose payload is hash-chained to the prior event. The host owns the chain. She can hand it to the SEF, the AT, or the CNPD on demand without having to ask anyone's permission to export her own records.

**Redacted-by-construction telemetry.** Anything that flows to an operator, a vendor, or a remote observer is structurally limited to non-sensitive metadata (event types, counts, hashes). The architecture proves this rather than promises it.

These principles align with the broader responsible-deployment posture Anthropic articulates for Claude in production. See [docs/](docs/) for the long-form reasoning as it lands.

---

## Status

**Alpha, in active development.** This pack is being published in support of an active pilot launching summer 2026 with an independent Portuguese vacation rental host. Skill instructions, connector contracts, and example transcripts arrive over the coming weeks as the pilot validates them against real reservations.

The intended structure of the pack (each directory expands as content arrives):

```
claude-for-portuguese-al-hosts/
├── skills/         # SKILL.md files Claude reads to do specific tasks
├── connectors/     # MCP connector reference implementations (SEF portal first)
├── examples/       # End-to-end worked transcripts with synthetic data
├── docs/           # Architecture, threat model, compliance notes
├── CONTRIBUTING.md
├── CODE_OF_CONDUCT.md
├── LICENSE         # Apache 2.0
└── README.md       # This file
```

Each directory currently contains a README explaining what is planned and the order it will be built in.

Breaking changes are possible during alpha but will be called out in a CHANGELOG once skill files start landing. The intention is to reach a stable v1.0 release after the first ten paying customers, which we expect by the end of 2026.

---

## Quick start

This pack is reference material. It is not a runnable application by itself. To use it, you will need:

1. **An agentic runtime** that can read SKILL.md files and invoke MCP connectors. Examples: Claude Desktop with MCP, Anthropic Claude Code, or a managed-service provider such as Obra.
2. **An Anthropic API key** (or equivalent Claude access) for the runtime to use.
3. **Connector credentials** for whichever external systems the workflow needs to touch. Most relevantly: SEF portal access (the host already has this; her runtime needs scoped access through a secrets broker, not the raw credential).
4. **A machine the customer trusts** for the workflow to run on. This is typically the host's own laptop or a small home server.

A minimal happy-path setup walkthrough lands in [examples/](examples/) as the example transcripts are written.

---

## Contributing

Contributions from Portuguese AL hosts, accountants, developers familiar with the SEF portal, and anyone building Claude-powered workflows for regulated EU SMBs are welcome.

Areas where we especially want help:

- **More language coverage** for the pre-arrival welcome skill (French, Spanish, German, Italian, Dutch, and Portuguese covered; English is the default).
- **SEF portal robustness** — the portal occasionally changes its DOM. We want a community of contributors keeping the connector contract current.
- **Test fixtures** with synthetic passport scans for as many issuing countries as possible.
- **Regulatory clarifications** — if you are a Portuguese tax lawyer or compliance professional and you see something we have stated incorrectly, please open an issue.

See [CONTRIBUTING.md](CONTRIBUTING.md) for the contributor workflow.

---

## Acknowledgements

This pack exists because [Anthropic](https://www.anthropic.com) builds Claude. It is the most capable, careful AI model available, and it is the only model this pack is built against. Anthropic also publishes its own Apache 2.0 vertical reference packs, including [Claude for Legal](https://github.com/anthropics/claude-for-legal), which inspired the structure of this repository.

The pack also stands on the shoulders of every Portuguese AL host who has ever filled out a SEF form by hand at midnight, every accountant who has reconciled a missing invoice on a Sunday, and every regulator whose insistence on doing this carefully is the reason the data stays where it should.

---

## License

Apache License 2.0. See [LICENSE](LICENSE).

---

## Related

- [Obra](https://get-obra.com) — managed-service deployment of these skills for hosts who do not want to run the runtime themselves.
- [Claude](https://www.anthropic.com/claude) — the reasoning core every skill in this pack invokes.
