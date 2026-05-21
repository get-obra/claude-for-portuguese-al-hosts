---
name: 03-siba-submission
version: 0.3.0
description: Consumes the verified GuestIdentityPayload from Skill 02 + the residence details collected by the guest-facing channel, and files the boletim de alojamento on the SIBA portal (siba.ssi.gov.pt) within the legally mandated window. SIBA uses a list/batch model: each guest is added to a list, then the whole list is sent. Host signs off on each filled form before it is saved, and explicitly sends the list. Captures the SIBA confirmation in the audit chain.
tier: gated
requires_extraction_agreement: false
connector_dependencies:
  - siba-portal
human_signoff: true
idempotency_key: "reservation_id + arrival_date + document_number_hash"
regulatory_anchor:
  - Decreto-Lei n.º 128/2014 (Portuguese tourism law, SIBA reporting obligation)
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

> **Version 0.3.0** (2026-05-20): schema, field mapping, and flow corrected against the **live SIBA portal** (`siba.ssi.gov.pt`, the `RegistaBoletins` form), observed end-to-end during first-pilot preparation. The earlier v0.2 8-field set was a secondary-source approximation and was wrong in several specifics. Corrections: (1) the real form has **11 fields**, adding *Local Nascimento* (place of birth), *Local Residência* + *País Residência* (residence is split into place and country), and **restoring *País Emissor Documento*** (issuing country), which v0.2 had wrongly removed; it IS required. (2) SIBA uses a **list/batch model**: open a list, add each guest's boletim, then send the list, not a per-guest single submit. (3) The three date fields are **readonly calendar widgets** (entered via a date picker, not typed). (4) The entry fields are **locked until a "Nova BAL" action unlocks them**. (5) The portal now lives on the SSI domain (`siba.ssi.gov.pt`) following the SEF→AIMA reorganization. See CHANGELOG.md for full version history.

## Purpose

The host has a verified identity payload for a newly-arrived guest, produced by [Skill 02 (`02-passport-extraction.md`)](02-passport-extraction.md). She also has the reservation context: the property where the guest is staying, the arrival date, and the planned check-out date. Portuguese tourism law obliges her to submit a *boletim de alojamento*, a guest registration record, to the SIBA portal operated by the authorities within the legally mandated window after the guest's arrival. Missing this submission carries substantial fines.

This skill drives the submission. It does not propose new identity data; it consumes what Skill 02 already verified and the host already approved. Its job is to fill the SIBA form correctly, surface the filled form for one final human review before the submit action executes, and then submit, capture, and audit-log the receipt.

The skill does **not** decide whether to submit. Submission is the regulatory baseline; it always happens for an arrived guest who is not exempt. The host's role at this stage is verification of the form, not approval of the decision to submit.

## Regulatory note

> **Authority status (2026 alpha):** SEF (Serviço de Estrangeiros e Fronteiras) was reorganized in 2023; the boletim-de-alojamento reporting obligation now sits with AIMA (Agência para a Integração, Migrações e Asilo). As of the 2026 live-portal observation, the SIBA boletim system is served on the **SSI domain** (Sistema de Segurança Interna) at **`siba.ssi.gov.pt`**. The boletim entry form is `/S/bal/RegistaBoletins.aspx`, reached from the submission-lists page (`/S/bal/LotesEnvio.aspx`). This skill's connector binding (`siba-portal`) follows the live endpoint. Any change is flagged via `drift_detected` events and surfaced to the host as a runtime alert.

> **Authentication model:** AL operators authenticate to the SIBA portal with credentials issued at AL registration, scoped through the secrets broker, stored in the operating system's secrets store, never logged. The exact SIBA login mechanism (single access code vs. user/password) is **pending precise live verification**. The connector treats it as a scoped credential set regardless. Per the local-first model, the human logs in interactively where possible so credentials are entered by hand and never stored in plaintext by the runtime.

> **Submission window (2026 alpha):** the current practical guidance is *within the legally mandated window after arrival*. This skill reads the configured window from the host's locale settings rather than hard-coding "24 hours" or "3 working days," so a regulatory update is a configuration change rather than a code change. Founders deploying this skill should verify the current window with their compliance counsel before going live.

## Input

The verified identity payload from Skill 02, plus the residence details collected via the guest-facing channel, plus the reservation context the runtime already has.

```
SibaSubmissionInput {
  identity: GuestIdentityPayload         // verified output of Skill 02; must include place_of_birth + issuing_country
  residence: {
    place: string                         // Local Residência, city / address; typed via the guest-facing channel
    country: string                       // País Residência, country of residence (Portuguese label)
  }
  reservation: {
    reservation_id: string               // ties this submission to the booking and the audit chain
    arrival_date: ISO-8601 date
    departure_date: ISO-8601 date
    property: {
      al_registration_number: string     // host's official AL number with Turismo de Portugal
      siba_credentials_ref: string        // reference to the host's SIBA login, scoped via the secrets
                                          // broker, never inlined here (exact mechanism per the auth note above)
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

The skill is a sequence of typed actions invoked against the `siba-portal` connector. It is not a free-form agentic loop. Every step is declared, deterministic, and audit-logged.

### Step 1: Idempotency check

Before doing anything that touches the portal, the skill computes the idempotency key (`reservation_id + arrival_date + sha256(document_number)`) and checks the audit chain for a prior `submission_acknowledged` event with the same key. If found, the skill returns `idempotency_replay_rejected` with the existing receipt and does not re-submit. Per the connector framework's idempotency rule, double-runs are detected and rejected; the host sees a friendly "this guest is already registered, here's the receipt" message rather than a duplicate submission.

### Step 2: Authenticated portal session

The connector opens an authenticated session against the SIBA portal (`siba.ssi.gov.pt`) using the host's scoped credentials (entered interactively where possible; never logged). If the session fails to establish (bad credentials, portal down), the skill halts with `halt_portal_unavailable` and a retry-after timestamp.

### Step 3: Open the submission list

SIBA works on a **list (lote) model**: boletins are added to a list, then the whole list is sent as a batch. The connector opens the submission-lists page (`LotesEnvio`) and starts (or resumes) a list, landing on the boletim entry form (`RegistaBoletins`). A list can hold one or many guests' boletins.

### Step 4: Per-guest form fill (unlock → fill → review → save)

For each guest in the submission, the connector:

1. **Unlocks the entry form.** The boletim fields are readonly/disabled until a **"Nova BAL"** (new boletim) action is triggered; the connector triggers it before filling. (Skipping this is why a naive fill silently fails. The fields aren't editable yet.)
2. **Fills the fields** from the payload, per the field mapping below. Field-fill is per-field deterministic: each input maps to exactly one form field; no AI interpretation of the page layout. Text fields and dropdowns are filled with real interactions; the three **date fields are readonly calendar widgets** handled by the connector's date-control routine.
3. **Surfaces the filled boletim to the host for review, then Saves it to the list.** The per-entry **Save** commits the boletim to the list; it is not yet sent to the regulator.

If a selector fails to match (form changed), the skill halts with `halt_drift_detected`. No partial entry is saved; the form is abandoned cleanly.

> **Save is effectively irreversible on this portal.** The live SIBA portal offers no delete for a saved boletim. A wrong save must be corrected by contacting the authority directly. So the host's review **before each Save** is the load-bearing gate: nothing unverified is ever saved. Any field the connector cannot yet fill with full confidence (e.g. the readonly calendar dates, until that path is validated end-to-end) is entered by the host by hand while the connector fills the rest, a deliberate hybrid that keeps the irreversible step certain.

### Step 5: Host final review and send (Enviar Lista)

When the list is complete, it is rendered to the host as a structured summary: each boletim, each field labelled with its source (Skill 02 vs. guest-facing channel vs. reservation context). The host sees:

- The identity fields from Skill 02 (name, date of birth, place of birth, nationality, document type, document number, issuing country)
- The residence place + country from the guest-facing channel
- The arrival/departure dates from the reservation
- A primary action: **Enviar Lista, send to SIBA** (with a dwell timer per the connector framework's gating rule)
- A secondary action: **Reject and explain** (abandons the send; emits `host_rejected_final_review` with the host's stated reason)

The dwell timer fights rubber-stamping: the host cannot send immediately; she must dwell long enough that review is plausible (default ten seconds for first-pilot operators). **The send (Enviar Lista) is the regulator-touching action and is always the host's explicit act. The runtime never sends a list autonomously.**

On host send, the connector triggers "Enviar Lista". The portal's response is captured: success → confirmation + timestamp + a downloadable receipt; failure → a SIBA error parsed into a structured `submission_failed` or `halt_validation_rejected_by_siba` result.

### Step 6: Receipt capture and notification

On success, the receipt PDF is stored locally alongside the audit chain entry (never uploaded to any operator-side surface). The host receives a notification through her configured channel (`whatsapp` or `email`) confirming the submission, with the SIBA confirmation number and a link to the local receipt.

## Field mapping (input → SIBA boletim)

The **11 fields** on the live SIBA boletim form (`RegistaBoletins`), with the exact on-screen labels:

| SIBA field (form label) | Source | Notes |
|---|---|---|
| Nome Completo | `identity.surname` + `identity.given_names` | full name |
| Data de Nascimento | `identity.date_of_birth` | **readonly calendar widget** (date picker; `dd-mm-yyyy`) |
| Local Nascimento | `identity.place_of_birth` | from the document's visual page (not the MRZ); Skill 02 must capture it |
| Nacionalidade | `identity.nationality` | dropdown of Portuguese-language country names |
| Local Residência | `residence.place` | typed input via the guest-facing channel (city / address) |
| País Residência | `residence.country` | dropdown of PT country names; collected with the residence |
| Número Documento | `identity.document_number` | |
| Tipo Documento | `identity.document_type` | dropdown: **PASSAPORTE / BILHETE DE IDENTIDADE / OUTROS** |
| País Emissor Documento | `identity.issuing_country` | dropdown of PT country names, **required** (v0.2 wrongly removed this) |
| Data de Check-in | `reservation.arrival_date` | **readonly calendar widget** |
| Data de Check-out | `reservation.departure_date` | **readonly calendar widget** |

**Field-set correction history.** v0.1 guessed 11 fields (some wrong). v0.2 cut to 8 from secondary-source documentation. But that cut was over-aggressive: it dropped *País Emissor* (issuing country), which the live form **requires**; it collapsed residence into one "address" field when the form splits it into *Local Residência* + *País Residência*; and it missed *Local Nascimento*. The table above is the field set observed **directly on the live `RegistaBoletins` form** (2026-05-20). The four dropdowns (nationality, residence country, document type, issuing country) take Portuguese-language labels. Any further field SIBA may require for specific document types is flagged via `drift_detected` if the connector encounters it. Login credentials are consumed by the connector; they are NOT form fields.

## Failure modes and escalation triggers

The skill is opinionated about what counts as a halt vs. what is acceptable retry behaviour.

**Hard halt (lands in host's hands immediately):**

- `halt_drift_detected`: a selector on the SIBA portal did not match. The portal has changed. The connector framework's contract is to alert, not click blindly. The host sees the structured drift summary and a clear "the portal changed; we are pausing until this is fixed" message.
- `halt_validation_rejected_by_siba`: SIBA returned a validation error (invalid passport format for a particular country, expired document, malformed address). The error is surfaced verbatim alongside the host's filled form so she can decide whether to correct (which loops back to Skill 02 or to the reservation context) or escalate to the guest.
- `host_rejected_final_review`: the host looked at the filled form and said no. Her reason is captured. The submission does not happen. The runtime queues a follow-up: either re-extract the passport (Skill 02), correct the reservation context, or escalate to the guest for clarification.

**Soft retry (transparent to the host):**

- `halt_portal_unavailable`: SIBA is down or under maintenance. The connector waits the suggested retry-after window and tries again. The host is notified only if the submission has not succeeded within a configurable threshold (default: half the legally mandated window remaining).
- Transient network failures: handled at the connector layer with exponential backoff. The skill does not surface these.

**Forbidden behaviour:**

- The skill **does not** submit a boletim that the host has not seen and not approved. Even if the runtime has historical sign-off patterns suggesting auto-approval would be safe, this skill always presents the filled form for review. Regulator-touching submissions are gated, full stop.
- The skill **does not** retry a `halt_validation_rejected_by_siba` automatically. SIBA's validation rules are authoritative; if SIBA says no, the host needs to see why before any retry.

## Audit events emitted

In order of typical flow:

- `submission_attempted`: skill invoked. Captures: reservation_id, host_id, idempotency_key, hashed identity payload.
- `submission_form_filled`: connector finished filling the form. Captures: hashed payload of filled fields, target SIBA endpoint, session_id.
- `host_final_review_required`: form presented for host sign-off. Captures: timestamp, host_id, dwell_time_required.
- `host_final_approved` OR `host_rejected_final_review`: host's decision. Captures: timestamp, dwell_time_observed, host's reason if rejected.
- `submission_submitted`: connector clicked submit on SIBA. Captures: SIBA-side request_id if available.
- `submission_acknowledged`: SIBA returned a confirmation number. Captures: confirmation_number, SIBA-side timestamp, hash of the receipt PDF. The PDF itself is stored locally; only its hash lands in the audit chain.
- `submission_failed`: SIBA returned an error. Captures: structured error from SIBA, retry-or-escalate guidance.
- `drift_detected`: connector encountered an unmatched selector. Captures: which selector, what was expected, what was found. Surfaced to the operator dashboard for connector-maintainer attention.
- `idempotency_replay_rejected`: duplicate run detected. Captures: existing confirmation_number.

Per the redacted-by-construction principle, raw identity values are never emitted to the audit chain, only hashes. The receipt PDF is stored locally with the audit chain entry; it is not uploaded to any operator-side surface.

## Worked examples

### Example A: Clean submission

A guest arrived this morning. Skill 02 verified the passport. The host's runtime invokes Skill 03 with the verified payload and the reservation context.

1. Idempotency check: no prior submission for this key. Proceed.
2. Authenticated session established on SIBA (`siba.ssi.gov.pt`).
3. New list opened. "Nova BAL" unlocks the entry form; the connector fills the 11 fields (the 8 text/dropdown fields directly; the 3 calendar dates per the validated date routine, or by the host in hybrid mode). No drift.
4. Host reviews the filled boletim, **Saves** it to the list, dwells the ten-second timer, and clicks **Enviar Lista**.
5. SIBA returns confirmation `SIBA-2026-05-20-AAA-001234` and a receipt.
6. Host receives a WhatsApp message: *"Boletim submetido. Número de confirmação SIBA-2026-05-20-AAA-001234. Recibo guardado localmente."*

Total wall-clock: ~30 seconds of host attention, ~90 seconds end-to-end.

### Example B: Host catches a wrong arrival date

The host reviews the filled form and notices the arrival date is one day off (the runtime pulled the wrong reservation row). She clicks **Reject and explain**, types *"arrival date should be 2026-05-19, not 2026-05-20"*, and the submission is abandoned.

The runtime emits `host_rejected_final_review` with her reason. A follow-up task is queued for the reservation-context layer to correct the arrival date. Once corrected, the runtime re-invokes Skill 03 with the same identity payload and the fixed context. Idempotency key changes (because arrival_date is part of it), so the new submission proceeds normally.

### Example C: Portal drift

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

- [`02-passport-extraction.md`](02-passport-extraction.md): produces the verified identity payload this skill consumes.
- [`connectors/siba-portal/`](../connectors/siba-portal/): Playwright-driven SIBA portal automation with pinned selectors and drift detection.
- A notification connector (WhatsApp or email) for the receipt acknowledgement message back to the host.
- A secrets broker holding the host's SIBA portal credentials.

## Versioning

`v0.3.0`, alpha. See `CHANGELOG.md` at the repo root for the full version history.

**v0.3.0 changes vs. v0.2.0** (corrected against the live `siba.ssi.gov.pt` portal, observed 2026-05-20):
- Field mapping corrected to the real **11 fields** on the `RegistaBoletins` form
- **Restored `País Emissor` (issuing country)**: v0.2 wrongly removed it; the live form requires it
- Split residence into `Local Residência` (place) + `País Residência` (country); input `home_address` → `residence { place, country }`
- Added `Local Nascimento` (place of birth): Skill 02 must now capture `place_of_birth` + `issuing_country`
- Documented the **list/batch model** (open list → per-guest Nova BAL unlock → fill → Save → Enviar Lista), replacing the per-guest single-submit model
- Documented that the three date fields are **readonly calendar widgets**, and that entry fields are **locked until "Nova BAL"**
- Noted **Save is irreversible** (no delete): host review before each Save is the load-bearing gate
- Softened the authentication claim (exact SIBA login mechanism pending precise live verification)
- Portal location updated to the SSI domain (`siba.ssi.gov.pt`)

**v0.2.0 changes vs. v0.1.0:**
- Input payload `PassportIdentityPayload` → `GuestIdentityPayload` (broader scope; see Skill 02 v0.2 changelog)
- Field mapping reduced from 11 fields to 8 fields based on primary-source documentation of the SIBA portal's actual requirements
- Removed `country_of_issue` and `document_expiry` field mappings (not required by SIBA)
- Added `home_address` as required input from the guest-facing channel (collected as typed input alongside the document image, since it is not on most identity documents)
- Authentication model documented: SIBA uses a single string code (issued at AL registration); not email/password
- Idempotency key updated from `passport_number_hash` to `document_number_hash` to match broader document-type scope
