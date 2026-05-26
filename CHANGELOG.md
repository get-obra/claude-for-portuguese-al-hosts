# Changelog

All notable changes to this pack are documented here. Follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/) conventions.

The pack contains multiple independently-versioned components (skills, connectors, documents). Each entry below names which components changed.

---

## 2026-05-26 (latest, Essay: the unenforced law)

### Documentation

- **Published** `docs/the-unenforced-law.md`: the second essay in the series. Diagnoses why a market with a real, universally-known short-term rental compliance gap has no buyers, names the existing tools honestly with what they file and what they verify (Chekin, ChargeAutomation, Sincro, SEFScan, SIBA GO, Homeit, CheckinScan, Your.Rentals, Ynnov, Smoobu, Hostaway, Avantio, Lodgify, Hostfully covered), distinguishes filing from verification, and explains the architecture worth building before the trigger event arrives. Quotes Hostaway's own support documentation on identity verification verbatim, with the source URL. Canonical version at [get-obra.com/the-unenforced-law](https://get-obra.com/the-unenforced-law). ~2,950 words.
- **Updated** `README.md` and `docs/README.md` to feature both essays in the "Start here" / Published lists.

---

## 2026-05-20 (Skills 02 + 03 v0.3.0: live-portal correction)

Both regulator-facing skills corrected against the **live SIBA portal** (`siba.ssi.gov.pt`), observed end-to-end during first-pilot preparation. The earlier v0.2 field set was a secondary-source approximation and was wrong in several specifics.

### Skill 03: SIBA submission `v0.3.0`

**Breaking** field-mapping and flow corrections from direct observation of the live `RegistaBoletins` form:

- **Field set corrected from 8 to the real 11 fields**: Nome Completo, Data de Nascimento, **Local Nascimento** (place of birth, added), Nacionalidade, **Local Residência** + **País Residência** (residence split into place + country), Número Documento, Tipo Documento, **País Emissor Documento** (issuing country, *restored*; v0.2 wrongly removed it), Data de Check-in, Data de Check-out.
- **Documented the list/batch model**: open a list (`LotesEnvio`) → per guest, "Nova BAL" unlocks the entry form → fill → **Save** commits the boletim to the list → **Enviar Lista** sends the batch to SIBA. Replaces the prior per-guest single-submit model.
- Entry fields are **locked until "Nova BAL"**; the three date fields are **readonly calendar widgets** (date-picker, not typed).
- **Save is irreversible** (no delete on the live portal). Host review before each Save is the load-bearing gate; Enviar Lista (the regulator send) is always the host's explicit act.
- Input `home_address` → `residence { place, country }`.
- Authentication claim softened (exact SIBA login mechanism pending precise live verification).
- Portal location updated to the SSI domain (`siba.ssi.gov.pt`) following the SEF→AIMA reorganization.

### Skill 02: Guest document extraction `v0.3.0`

**Breaking** output-schema corrections to match Skill 03 v0.3:

- **Restored `issuing_country`** (`País Emissor`): required by the live SIBA form; v0.2 wrongly removed it. Cross-checked by the agreement gate via the MRZ issuing-state code.
- **Added `place_of_birth`** (`Local Nascimento`): required by the live form; read from the document's **visual page only** (the MRZ does not encode it), so it is host-verified without an agreement cross-check.
- Output payload grows from 6 to 8 fields; Pass A (visual) reads all 8, Pass B (MRZ) reads the 7 the MRZ encodes.
- `expiry_date` remains dropped (not required by SIBA).

### Note on provenance

These corrections come from observing the real portal during pilot preparation, not from production guest data. No real guest, host, or property information is included; the field names are the portal's own public on-screen labels.

---

## 2026-05-19 (Skill 01 v0.1.1 same-evening correction)

### Skill 01: Pre-arrival welcome `v0.1.1`

Same-evening correction of the v0.1.0 design. On review we identified that the v0.1.0 recommended-delivery mechanism for booking-platform reservations ("Shape K", embedding the secure-upload URL via QR code in a welcome PDF attached to platform chat) bets the host's listing on the workaround surviving Airbnb's moderation evolution. Not a bet we are willing to ask pilot users to make.

**Removed:**

- Shape K as a recommended delivery mechanism. The skill no longer produces a PDF-with-QR for booking-platform reservations.
- The "two-artifact structure (chat body + PDF)" framing that implied PDF delivery on platform channels.
- The empirical-test-of-Shape-K queued task (moot since we are not shipping Shape K).

**Added, four platform-policy-compliant delivery modes:**

- **Mode 1A**: In-person at arrival on the host's device. First-pilot path. For hosts who meet guests face-to-face.
- **Mode 1B**: Physical QR-encoded welcome card placed inside the property. Keybox-host path. The QR lives in physical material the booking platform has no content-moderation jurisdiction over.
- **Mode 1C**: Guest-initiated WhatsApp (or other guest-chosen channel). Real recurring case per first-hand operator reporting; the host's explicit judgment is the gate; never auto-routed by the runtime.
- **Mode 2**: Direct-booking via host-chosen channel (email / WhatsApp / SMS / Telegram / Signal / etc.). No platform constraint applies because no booking platform sits in the conversation. Many direct-booking hosts use WhatsApp as their primary guest channel.

**Documented:**

- Mode 3 (platform-integrated delivery via Shape H, native partner integration with Airbnb / Booking / Vrbo) as the structurally right long-term answer for the keybox-majority case. Gated on platform partnership and not yet available. The skill explicitly does not simulate Mode 3 from the host side.
- Language baselines updated: chat-body baselines for Modes 1A/1B contain NO reference to attachments, links, or upload mechanisms; chat body is pure relational warmth. Mode 2 baselines reference the attached PDF (no platform constraint).
- Five worked examples covering Modes 1A, 1B, 1C, 2, and the voice-learning case.
- "Does not put hosts at platform-policy risk" added to the explicit-non-goals list.

This correction was applied to the public repo within ~90 minutes of the v0.1.0 publication.

---

## 2026-05-19 (earlier, Skill 01 v0.1.0 + Skills v0.2 bumps)

### Skill 01: Pre-arrival welcome `v0.1.0` *(superseded by v0.1.1)*

Initial publication of the multilingual welcome skill that opens the canonical workflow.

- **Drafts** the welcome message in the guest's language (Portuguese, English, French, Spanish, German baselines tested; Italian, Dutch, Polish, Mandarin queued)
- **Adapts** to the host's voice via structured prompt-engineering layer (`HostVoiceAnchors`: about-self, example-greetings, tone descriptors). No separately-trained model, just an in-prompt stylistic context
- **Produces** two artifacts per reservation: a short chat body (for delivery through the booking platform's chat surface) and a longer welcome PDF (with embedded QR code encoding the per-reservation secure-upload URL)
- **Implements** Shape K delivery per the trust-gap thesis: chat body never contains a text URL (avoids the Airbnb-style external-link filter); PDF carries the URL via QR (a delivery form the platform's automated systems do not parse)
- **Gated**: every welcome is reviewed by the host before send, with dwell timer and required reason
- **Voice learning**: host edit signals tune the voice anchors over time; rejection reasons are extracted into structured tone descriptors that feed back into the next generation
- Three worked examples (French guest on-booking; Portuguese guest in host's language; host rejects with tone feedback that updates the voice anchors)

### Skill 02: Guest document extraction with agreement gate `v0.2.0`

**Breaking changes** to the output payload schema based on primary-source documentation of the SIBA portal's actual field requirements:

- **Renamed** the skill internally from `02-passport-extraction` to `02-guest-document-extraction` (file name preserved for URL stability)
- **Renamed** output payload `PassportIdentityPayload` → `GuestIdentityPayload`
- **Removed** `country_of_issue` field (not required by SIBA)
- **Removed** `expiry_date` field (not required by SIBA)
- **Renamed** `passport_number` → `document_number` (broader scope)
- **Added** `document_type` field (`passport` | `national_id` | `residence_permit`)
- **Documented** Pass B MRZ parsing dispatches on TD1 (national ID, three lines of 30 chars), TD2, TD3 (passport, two lines of 44 chars) shape detection, supports passports, EU national ID cards, and residence permits
- **Clarified** that `home_address` (a SIBA-required field) is collected separately via the guest-facing channel since it is not present on most identity documents
- Worked examples updated to cover both passport (Example A) and EU national ID card (Example B) extraction paths
- "What this skill does NOT do" section updated to note non-acceptance of driving licences by SIBA

### Skill 03: SIBA submission `v0.2.0`

**Breaking changes** to the input schema and field mapping to match Skill 02 v0.2 and the actual SIBA portal field set:

- **Updated** input schema to consume `GuestIdentityPayload` (Skill 02 v0.2's output) instead of `PassportIdentityPayload`
- **Updated** input schema to additionally consume `home_address` from the guest-facing channel
- **Reduced** field mapping from 11 fields to 8 fields based on primary-source documentation of SIBA's actual requirements:
  - Guest name (surname + given names)
  - Nationality
  - Date of birth
  - Home address
  - Document type
  - Document number
  - Check-in date
  - Check-out date
- **Removed** `country_of_issue` and `document_expiry` field mappings (not required by SIBA)
- **Documented** SIBA authentication model: AL operators authenticate using a single string code issued at AL registration (not email/password). The `siba_login_code` is consumed by the connector for authentication; it is NOT a form field
- **Updated** idempotency key from `reservation_id + arrival_date + passport_number_hash` to `reservation_id + arrival_date + document_number_hash` to match the broader document-type scope
- **Removed** Lei n.º 73/2025 reference from regulatory anchor (the AIMA transition continues to be documented in the skill's regulatory note, but specific instrument reference deferred to COMPLIANCE-NOTES.md when it lands)

### Documentation

- **Published** `docs/the-trust-gap.md`: the narrative essay on why this pack exists; canonical version at https://get-obra.com/the-trust-gap. Mirrored here for engineers and contributors discovering the work via GitHub.
- **Published** `docs/ARCHITECTURE.md`: the engineer-readable mechanism document. Defends the four architectural properties from the essay with case-by-case analysis of weaker-alternative failure modes. Covers the connector framework, the three-gate trust pillar, the pre-proposal extraction agreement gate, the audit chain, and the action authorization model.
- **Updated** `README.md` to feature both depth pieces as the "Start here" reading order.
- **Updated** `docs/README.md`: `the-trust-gap.md` and `ARCHITECTURE.md` moved from "Planned" to "Published"; `THREAT-MODEL.md`, `COMPLIANCE-NOTES.md`, `DRIFT-PLAYBOOK.md` remain in "Planned".

---

## 2026-05-19 (earlier, initial publication)

### Initial scaffold

- **Created** Apache 2.0 license
- **Published** main `README.md`, `CONTRIBUTING.md`, `CODE_OF_CONDUCT.md`
- **Created** `skills/`, `connectors/`, `examples/`, `docs/` directories with stub READMEs explaining intended contents

### Skill 02: Passport extraction with agreement gate `v0.1.0` *(superseded by v0.2.0)*

- Initial publication of the pre-proposal extraction agreement gate pattern
- Visual-vs-MRZ two-pass independent reads with ICAO 9303 check-digit validation
- Five-language escalation message baselines (Portuguese, English, French, Spanish, German) for "request clearer photo" flow
- Three worked examples demonstrating agreement, disagreement-resolved-by-MRZ, and check-digit-failure paths

### Skill 03: SEF submission `v0.1.0` *(superseded by v0.2.0)*

- Initial publication of the regulator-touching companion to Skill 02
- Idempotency check; authenticated portal session; per-selector deterministic form fill; host final review with dwell timer; receipt capture
- Documented authority status (SEF → AIMA transition) and submission window as runtime config (not hard-coded)
- Five hard-halt conditions and transparent soft-retry behaviour documented
- Three worked examples (clean submission, host catches wrong arrival date, portal drift detected)

---

## Versioning policy

Each component (skill, connector, document) versions independently with semver. The pack as a whole does not yet have a release version; we will tag releases at the repo level once skill content stabilizes after the first pilot.

During alpha:

- Major version bumps (`v0.X.0` → `v0.(X+1).0`) indicate breaking changes to a component's contract (skill output schema, connector action manifest, etc.)
- Minor version bumps indicate added content that does not break existing consumers
- Patch version bumps indicate corrections that preserve the contract

After v1.0 (expected post-first-ten-paying-customers, end of 2026), normal semver applies.

---

*Pull requests, corrections, and questions welcome via issues on this repo or `team@get-obra.com`.*
