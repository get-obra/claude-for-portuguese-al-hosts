---
name: 03-sef-submission
version: 0.2.0
description: Consumes the verified GuestIdentityPayload from Skill 02 + the home address collected by the guest-facing channel, and submits the boletim de alojamento to the SIBA portal within the legally mandated window. Host signs off on the filled form before the submission action executes. Captures the SIBA confirmation number in the audit chain.
tier: gated
requires_extraction_agreement: false
connector_dependencies:
  - sef-portal
human_signoff: true
idempotency_key: "reservation_id + arrival_date + document_number_hash"
regulatory_anchor:
  - Decreto-Lei n.º 128/2014 (Portuguese tourism law — SIBA reporting obligation)
  - Decreto-Regulamentar n.º 14/2019 (boletim de alojamento implementation)
audit_event_types:
  - submission_attempted
  - submission_form_filled
  - host_final_review_required
  - host_final_approved
  - submission_submitted
  - submission_acknowledged
  - submission_failed
  - submission_retry_required
  - drift_detected
  - idempotency_replay_rejected
---

# Skill: SIBA submission

> **Version 0.2.0** (2026-05-19) — schema and field mapping updated to match the actual SIBA portal field set as documented by an active Portuguese AL operator. The 8-field set: guest name, nationality, date of birth, guest home address, document type, document number, check-in date, check-out date. Removed `country_of_issue` and `document_expiry` from the field mapping (not required by SIBA). Added `home_address` as a required input from the guest-facing channel. Updated authentication model: SIBA portal uses a single string code issued at AL registration, not email/password. See CHANGELOG.md for full version history.

## Purpose

The host has a verified identity payload for a newly-arrived guest, produced by [Skill 02 (`02-passport-extraction.md`)](02-passport-extraction.md). She also has the reservation context: the property where the guest is staying, the arrival date, and the planned check-out date. Portuguese tourism law obliges her to submit a *boletim de alojamento* — a guest registration record — to the SIBA portal operated by the border authority within the legally mandated window after the guest's arrival. Missing this submission carries substantial fines.

This skill drives the submission. It does not propose new identity data; it consumes what Skill 02 already verified and the host already approved. Its job is to fill the SIBA form correctly, surface the filled form for one final human review before the submit action executes, and then submit, capture, and audit-log the receipt.

The skill does **not** decide whether to submit. Submission is the regulatory baseline; it always happens for an arrived guest who is not exempt. The host's role at this stage is verification of the form, not approval of the decision to submit.

## Regulatory note

> **Authority status (2026 alpha):** SEF (Serviço de Estrangeiros e Fronteiras) was reorganized in 2023; the boletim-de-alojamento reporting obligation now sits with AIMA (Agência para a Integração, Migrações e Asilo). The SIBA portal endpoint may rename or migrate during this transition. This skill's connector binding (`sef-portal`) is the canonical name in this pack for historical continuity; the connector itself is responsible for following the live endpoint. Any change is flagged via `drift_detected` events and surfaced to the host as a runtime alert.

> **Authentication model:** AL operators authenticate to the SIBA portal using a single string code issued by SEF at AL registration. There is no email/password login. The connector framework's auth model for `sef-portal` treats this as a single-secret credential stored in the operating system's secrets store, scoped to this connector, never logged.

> **Submission window (2026 alpha):** the current practical guidance is *within the legally mandated window after arrival*. This skill reads the configured window from the host's locale settings rather than hard-coding "24 hours" or "3 working days," so a regulatory update is a configuration change rather than a code change. Founders deploying this skill should verify the current window with their compliance counsel before going live.

## Input

The verified identity payload from Skill 02, plus the home address collected via the guest-facing channel, plus the reservation context the runtime already has.

```
SibaSubmissionInput {
  identity: GuestIdentityPayload         // verified output of Skill 02 (v0.2.0)
  home_address: string                    // guest's home-country address; collected as typed input
                                          // via the guest-facing channel alongside the document image
                                          // (not present on the document itself)
  reservation: {
    reservation_id: string               // ties this submission to the booking and the audit chain
    arrival_date: ISO-8601 date
    departure_date: ISO-8601 date
    property: {
      al_registration_number: string     // host's official AL number with Turismo de Portugal
      siba_login_code: string            // host's SIBA portal access code (single-string credential
                                          // issued at SEF registration; scoped via secrets broker)
    }
  }
  host: {
    host_id: string                      // for the audit chain
    host_locale: "pt-PT"
    notification_channel: "whatsapp" | "email" // where the SIBA receipt lands after submission
  }
}
```

The skill validates the input against the schema before doing anything else. Any missing or malformed field halts with `submission_failed` and an explanation pointing at the field. Validation errors at this stage are runtime bugs; they should never reach the host's UI.

## Output

```
SefSubmissionOutput =
  | { status: "submitted", siba_receipt: SibaReceipt, audit_chain_pointer: string }
  | { status: "host_rejected_final_review", reason: string, audit_chain_pointer: string }
  | { status: "halt_drift_detected", drift_summary: string, audit_chain_pointer: string }
  | { status: "halt_portal_unavailable", retry_after: ISO-8601 timestamp, audit_chain_pointer: string }
  | { status: "halt_validation_rejected_by_siba", siba_error: string, audit_chain_pointer: string }
  | { status: "idempotency_replay_rejected", existing_receipt: SibaReceipt, audit_chain_pointer: string }

SibaReceipt {
  confirmation_number: string            // SIBA-issued unique identifier for this boletim
  submitted_at: ISO-8601 timestamp       // SIBA-side timestamp from the receipt
  reservation_id: string                 // tied back to the booking
  receipt_pdf_pointer: string            // local path to the SIBA receipt PDF, stored alongside the audit chain
}
```

## The submission flow

The skill is a sequence of typed actions invoked against the `sef-portal` connector. It is not a free-form agentic loop. Every step is declared, deterministic, and audit-logged.

### Step 1 — Idempotency check

Before doing anything that touches the portal, the skill computes the idempotency key (`reservation_id + arrival_date + sha256(document_number)`) and checks the audit chain for a prior `submission_acknowledged` event with the same key. If found, the skill returns `idempotency_replay_rejected` with the existing receipt and does not re-submit. Per the connector framework's idempotency rule, double-runs are detected and rejected; the host sees a friendly "this guest is already registered, here's the receipt" message rather than a duplicate submission.

### Step 2 — Authenticated portal session

The connector opens an authenticated session against the SIBA portal using the host's `siba_login_code` (a single string code issued at AL registration; scoped through the secrets broker, never logged). If the session fails to establish (wrong code, portal down, code revoked), the skill halts with `halt_portal_unavailable` and a retry-after timestamp.

### Step 3 — Form fill

The connector navigates to the boletim submission form and fills the fields from the input payload, per the field mapping below. Field-fill is per-field deterministic: each input field maps to exactly one output field on the form. There is no AI interpretation of the page layout at this stage; the connector uses pinned selectors.

If any selector fails to match (form changed), the skill halts with `halt_drift_detected` and a structured description of what failed. No partial submission is attempted; the form is abandoned cleanly.

### Step 4 — Host final review

The filled form is rendered to the host as a structured summary, with each field labelled and the source highlighted (which fields came from Skill 02 vs. from the guest-facing channel vs. from the reservation context). The host sees:

- The six identity fields from Skill 02
- The home address from the guest-facing channel
- The arrival/departure dates from the reservation
- A primary action: **Submit to SIBA** (with a dwell timer per the connector framework's gating rule)
- A secondary action: **Reject and explain** (which abandons the submission and emits `host_rejected_final_review` with the host's stated reason)

The dwell timer's purpose is to fight rubber-stamping. The host cannot click submit immediately; she must dwell on the form long enough that visual review is plausible. The configured dwell time defaults to ten seconds for first-pilot operators and scales down as historical false-positive rates inform the runtime.

### Step 5 — Submission

On host approval, the connector clicks the SIBA portal's submit action. The portal's response is captured: success → confirmation number + timestamp + a downloadable receipt PDF; failure → a SIBA error message that the connector parses into a structured `submission_failed` or `halt_validation_rejected_by_siba` result.

### Step 6 — Receipt capture and notification

On success, the receipt PDF is stored locally alongside the audit chain entry (never uploaded to any operator-side surface). The host receives a notification through her configured channel (`whatsapp` or `email`) confirming the submission, with the SIBA confirmation number and a link to the local receipt.

## Field mapping (input → SIBA boletim)

The eight fields SIBA actually requires, as documented by an active Portuguese AL operator:

| SIBA field | Source | Notes |
|---|---|---|
| Nome (guest name) | `identity.surname` + `identity.given_names` | family-name-first ordering per Skill 02's output |
| Nacionalidade (nationality) | `identity.nationality` | ISO-3166-alpha-3 |
| Data de nascimento (date of birth) | `identity.date_of_birth` | YYYY-MM-DD |
| Morada (guest's home address) | `home_address` | typed input collected via the guest-facing channel; NOT on the identity document |
| Tipo de documento (document type) | `identity.document_type` | `passport` / `national_id` / `residence_permit` |
| Número do documento (document number) | `identity.document_number` | strip leading zeros only if the MRZ form also stripped them |
| Data de check-in (arrival date) | `reservation.arrival_date` | YYYY-MM-DD |
| Data de check-out (planned departure) | `reservation.departure_date` | YYYY-MM-DD |

**Field set confirmation history.** v0.1 of this skill mapped to 11 fields including `country_of_issue`, `document_expiry`, and a multi-component property address. v0.2 reduces to the 8 fields above based on primary-source documentation from an active AL operator. Confirmation of `sex` field requirement (and any additional fields SIBA may require for non-passport document types) is still pending live-portal validation during first pilot. The `siba_login_code` is consumed by the connector for authentication; it is NOT a form field.

## Failure modes and escalation triggers

The skill is opinionated about what counts as a halt vs. what is acceptable retry behaviour.

**Hard halt (lands in host's hands immediately):**

- `halt_drift_detected` — a selector on the SIBA portal did not match. The portal has changed. The connector framework's contract is to alert, not click blindly. The host sees the structured drift summary and a clear "the portal changed; we are pausing until this is fixed" message.
- `halt_validation_rejected_by_siba` — SIBA returned a validation error (invalid passport format for a particular country, expired document, malformed address). The error is surfaced verbatim alongside the host's filled form so she can decide whether to correct (which loops back to Skill 02 or to the reservation context) or escalate to the guest.
- `host_rejected_final_review` — the host looked at the filled form and said no. Her reason is captured. The submission does not happen. The runtime queues a follow-up: either re-extract the passport (Skill 02), correct the reservation context, or escalate to the guest for clarification.

**Soft retry (transparent to the host):**

- `halt_portal_unavailable` — SIBA is down or under maintenance. The connector waits the suggested retry-after window and tries again. The host is notified only if the submission has not succeeded within a configurable threshold (default: half the legally mandated window remaining).
- Transient network failures — handled at the connector layer with exponential backoff. The skill does not surface these.

**Forbidden behaviour:**

- The skill **does not** submit a boletim that the host has not seen and not approved. Even if the runtime has historical sign-off patterns suggesting auto-approval would be safe, this skill always presents the filled form for review. Regulator-touching submissions are gated, full stop.
- The skill **does not** retry a `halt_validation_rejected_by_siba` automatically. SIBA's validation rules are authoritative; if SIBA says no, the host needs to see why before any retry.

## Audit events emitted

In order of typical flow:

- `submission_attempted` — skill invoked. Captures: reservation_id, host_id, idempotency_key, hashed identity payload.
- `submission_form_filled` — connector finished filling the form. Captures: hashed payload of filled fields, target SIBA endpoint, session_id.
- `host_final_review_required` — form presented for host sign-off. Captures: timestamp, host_id, dwell_time_required.
- `host_final_approved` OR `host_rejected_final_review` — host's decision. Captures: timestamp, dwell_time_observed, host's reason if rejected.
- `submission_submitted` — connector clicked submit on SIBA. Captures: SIBA-side request_id if available.
- `submission_acknowledged` — SIBA returned a confirmation number. Captures: confirmation_number, SIBA-side timestamp, hash of the receipt PDF. The PDF itself is stored locally; only its hash lands in the audit chain.
- `submission_failed` — SIBA returned an error. Captures: structured error from SIBA, retry-or-escalate guidance.
- `drift_detected` — connector encountered an unmatched selector. Captures: which selector, what was expected, what was found. Surfaced to the operator dashboard for connector-maintainer attention.
- `idempotency_replay_rejected` — duplicate run detected. Captures: existing confirmation_number.

Per the redacted-by-construction principle, raw identity values are never emitted to the audit chain — only hashes. The receipt PDF is stored locally with the audit chain entry; it is not uploaded to any operator-side surface.

## Worked examples

### Example A — Clean submission

A guest arrived this morning. Skill 02 verified the passport. The host's runtime invokes Skill 03 with the verified payload and the reservation context.

1. Idempotency check: no prior submission for this key. Proceed.
2. Authenticated session established on SIBA.
3. Form filled with the seven identity fields + four reservation fields. No drift.
4. Host sees the filled form, dwells the ten-second timer, clicks **Submit to SIBA**.
5. SIBA returns confirmation number `SIBA-2026-05-19-AAA-001234` and a receipt PDF.
6. Host receives a WhatsApp message: *"Boletim submetido. Número de confirmação SIBA-2026-05-19-AAA-001234. Recibo guardado localmente."*

Total wall-clock: ~30 seconds of host attention, ~90 seconds end-to-end.

### Example B — Host catches a wrong arrival date

The host reviews the filled form and notices the arrival date is one day off (the runtime pulled the wrong reservation row). She clicks **Reject and explain**, types *"arrival date should be 2026-05-19, not 2026-05-20"*, and the submission is abandoned.

The runtime emits `host_rejected_final_review` with her reason. A follow-up task is queued for the reservation-context layer to correct the arrival date. Once corrected, the runtime re-invokes Skill 03 with the same identity payload and the fixed context. Idempotency key changes (because arrival_date is part of it), so the new submission proceeds normally.

### Example C — Portal drift

A regulatory reorganization (SEF → AIMA transition) changed a selector on the SIBA login page. The connector's pinned selector for the username field no longer matches. The skill halts with `halt_drift_detected` before any data is entered.

The host sees: *"The SIBA portal has changed in a way we did not anticipate. We have not submitted anything. The Obra team has been notified and is fixing the connector. Your submission is queued and will go out as soon as the fix lands. You have [time remaining] in the legally mandated window."*

The operator-side dashboard receives the same `drift_detected` event; the connector maintainer ships a fix; the runtime resumes the queued submission.

## What this skill does NOT do

- **Does not read the passport.** Skill 02 already did that.
- **Does not decide whether to submit.** Submission is mandatory for arrived guests; the host's role is form-fill verification, not policy.
- **Does not handle the inverse case** (a guest cancels before arrival). Cancellation handling is a separate workstream.
- **Does not chase SIBA after acknowledgement** (status updates, post-submission corrections). Those are downstream skills.
- **Does not authenticate the host's SIBA account or manage MFA.** The connector handles authentication; the runtime stores credentials via the secrets broker.
- **Does not store the passport image.** Per Skill 02's scope, the image is discarded after submission acknowledgement.
- **Does not send the receipt to the guest.** The receipt is the host's record, not the guest's.

## Dependencies

- [`02-passport-extraction.md`](02-passport-extraction.md) — produces the verified identity payload this skill consumes.
- [`connectors/sef-portal/`](../connectors/sef-portal/) — Playwright-driven SIBA portal automation with pinned selectors and drift detection.
- A notification connector (WhatsApp or email) for the receipt acknowledgement message back to the host.
- A secrets broker holding the host's SIBA portal credentials.

## Versioning

`v0.2.0` — alpha. See `CHANGELOG.md` at the repo root for the full version history.

**v0.2.0 changes vs. v0.1.0:**
- Input payload `PassportIdentityPayload` → `GuestIdentityPayload` (broader scope; see Skill 02 v0.2 changelog)
- Field mapping reduced from 11 fields to 8 fields based on primary-source documentation of the SIBA portal's actual requirements
- Removed `country_of_issue` and `document_expiry` field mappings (not required by SIBA)
- Added `home_address` as required input from the guest-facing channel (collected as typed input alongside the document image, since it is not on most identity documents)
- Authentication model documented: SIBA uses a single string code (issued at AL registration); not email/password
- Idempotency key updated from `passport_number_hash` to `document_number_hash` to match broader document-type scope
