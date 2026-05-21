# Skills

Each skill in this directory is a single Markdown file with structured frontmatter and task-specific instructions Claude reads to perform one operation. Skills are the building blocks an agentic runtime composes into workflows.

## Published

| # | File | What it does | Version |
|---|---|---|---|
| 01 | [`01-pre-arrival-welcome.md`](01-pre-arrival-welcome.md) | Drafts the welcome message in the guest's native language, in the host's voice. Produces both a chat body (no text URL, platform-policy-compliant) and a welcome PDF (with embedded QR encoding the secure-upload URL). Host approves both before send. | `v0.1.0` |
| 02 | [`02-passport-extraction.md`](02-passport-extraction.md) | Reads a guest identity document (passport / national ID / residence permit); a second independent MRZ read cross-checks the numbers via ICAO 9303 check digits; host signs off on the result. | `v0.2.0` |
| 03 | [`03-siba-submission.md`](03-siba-submission.md) | Drives the SIBA reporting portal to submit the host-approved boletim de alojamento within the legally mandated window. Idempotency, drift detection, host final review with dwell timer. | `v0.2.0` |

## Planned

In order of arrival:

| # | File | What it does |
|---|---|---|
| 04 | `04-in-stay-support.md` | Handles guest questions in their language during the stay, escalates judgment calls to the host, never invents policy. |
| 05 | `05-accountant-handoff.md` | Generates the structured monthly record; host approves the figures; accountant signs off. |

Plus a separate **guest-facing-channel skill** under [`/connectors/`](../connectors/) that provides the local-runtime secure upload page Skill 01 references via QR code.

## Skill file shape

Each skill file follows this structure:

- YAML frontmatter: `name`, `version`, `description`, `tier` (autonomous / gated / forbidden), `connector_dependencies`, `human_signoff` (true/false), `regulatory_anchor`, `audit_event_types`.
- Body: task description, input expectations, output schema, edge cases, what triggers escalation to the human, what gets written to the audit chain, worked examples.

`02-passport-extraction.md` is the canonical template. Other skills mirror its shape with skill-specific sections (the pre-proposal extraction agreement gate is in Skill 02; the host-voice model is in Skill 01; the idempotency + portal-drift handling is in Skill 03).

## Versioning

Each skill versions independently with semver. See [`CHANGELOG.md`](../CHANGELOG.md) at the repo root for the full history. Breaking changes during alpha are called out per skill and per version.

## Contributions

Welcome via pull request. The high-leverage areas right now:

- **More language baselines** for Skill 01 (Italian, Dutch, Polish, Mandarin queued for native-speaker review)
- **More document-type coverage** for Skill 02 (non-EU national IDs with different MRZ shapes)
- **Connector contributions** under [`/connectors/`](../connectors/). `siba-portal/` is the priority for Skill 03 to ship runnable
- **Edge cases** from anyone running this stack against real reservations
