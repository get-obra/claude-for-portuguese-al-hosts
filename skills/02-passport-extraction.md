---
name: 02-guest-document-extraction
version: 0.3.0
description: Reads a guest identity document (passport, EU national ID card, residence permit) and produces a verified structured identity payload ready for SIBA submission. Runs the pre-proposal extraction agreement gate (visual fields vs. MRZ) before presenting results to the host for sign-off.
tier: gated
requires_extraction_agreement: true
connector_dependencies:
  - claude-vision
human_signoff: true
regulatory_anchor:
  - Decreto-Lei n.º 128/2014 (Portuguese tourism law, SIBA reporting obligation)
  - Decreto-Regulamentar n.º 14/2019 (boletim de alojamento implementation)
  - GDPR Article 9 (special-category identity data)
audit_event_types:
  - extraction_attempted
  - extraction_agreed
  - extraction_disagreed
  - host_verified
  - host_corrected
  - escalation_to_guest_requested
---

# Skill: Guest document extraction with agreement gate

> **Version 0.3.0** (2026-05-20): corrected against the live SIBA portal (see Skill 03 v0.3.0). The v0.2 schema wrongly dropped the document's **issuing country**, which the live SIBA form (*País Emissor Documento*) **requires**. It is restored here. Also added **place of birth** (*Local Nascimento*), which the form requires and which is read from the document's visual page only (the MRZ does not encode it, so the agreement gate cannot cross-check it; the host verifies it directly). `expiry_date` remains dropped (SIBA does not require it). Document-type coverage (passport / national ID / residence permit) and the residence-collected-separately note are unchanged from v0.2. See CHANGELOG.md for full version history.

## Purpose

The host has received a guest identity document image: usually a passport, sometimes an EU national ID card, occasionally a residence permit. The image arrives through the guest-facing secure upload channel (see Skill 01 and the guest-facing channel skill for delivery mechanism). She needs the identity fields the Portuguese **SIBA portal** (operated under the SEF / AIMA reporting obligation) requires for the mandatory post-arrival report. Reading those fields off a phone-camera photo of a foreign-script document is tedious and error-prone, and a wrong value is a fine.

This skill does the reading work. It runs two independent extractions and only proceeds when they agree. The host's job is reduced to verification: she confirms or corrects what the skill extracted, then signs off.

The skill does **not** submit to the SIBA portal. That is Skill 03 (`03-sef-submission.md`), which consumes this skill's verified output. The skill also does **not** collect the guest's home address. That field is required by SIBA but is not present on most identity documents, so it is collected separately via the guest-facing channel (see Skill 01).

## Input

A single document image plus the reservation context.

```
GuestDocumentExtractionInput {
  image: {
    bytes: Buffer
    mime_type: "image/png" | "image/jpeg" | "application/pdf"
    source_channel: "guest-upload-channel" | "manual-upload"
    received_at: ISO-8601 timestamp
  }
  expected_document_type?: "passport" | "national_id" | "residence_permit"  // optional hint from the guest-facing channel
  reservation_id: string                  // ties the audit chain to a specific booking
  guest_handle: string                    // guest's name as known on the booking platform (for sanity check)
  host_locale: "pt-PT"                    // host's preferred display language for the verification UI
}
```

The image is expected to show the document's **biographical data page** for a passport, the **front side** for an EU national ID card, or the equivalent identity-data page for other accepted documents. If the host received multiple images, this skill is invoked once per image and the runtime picks the one that successfully passes the agreement gate.

## Output

A verified identity payload, ready to be consumed by Skill 03 (SIBA submission), or a structured halt that lands in the host's hands for manual resolution.

```
GuestDocumentExtractionOutput =
  | { status: "verified", payload: GuestIdentityPayload, audit_chain_pointer: string }
  | { status: "host_corrected", payload: GuestIdentityPayload, audit_chain_pointer: string }
  | { status: "halt_disagreement", visual_read: GuestIdentityPayload, mrz_read: GuestIdentityPayload, diff: Diff, audit_chain_pointer: string }
  | { status: "halt_unreadable", reason: "visual_illegible" | "mrz_missing" | "mrz_check_digit_failed" | "wrong_document_type", audit_chain_pointer: string }
  | { status: "escalation_requested", guest_message_draft: string, audit_chain_pointer: string }

GuestIdentityPayload {
  surname: string                         // SIBA expects family-name-first ordering
  given_names: string                     // space-separated if multiple
  document_type: "passport" | "national_id" | "residence_permit"
  document_number: string                 // alphanumeric; format varies by issuing country and document type
  nationality: ISO-3166-alpha-3           // e.g., "FRA" for France
  date_of_birth: ISO-8601 date            // YYYY-MM-DD
  issuing_country: ISO-3166-alpha-3       // País Emissor, SIBA requires it; encoded in the MRZ + on the visual page
  place_of_birth: string                  // Local Nascimento, SIBA requires it; VISUAL PAGE ONLY (not in the MRZ)
}
```

> **Schema status (v0.3 alpha):** the eight-field set above reflects the live SIBA portal's actual requirements (observed 2026-05-20; see Skill 03 v0.3.0). `issuing_country` (*País Emissor*) is **required** and was wrongly removed in v0.2; restored. `place_of_birth` (*Local Nascimento*) is **required** and is read from the visual page only. The MRZ does not encode it, so it cannot be cross-checked by the agreement gate and is host-verified directly. `expiry_date` remains dropped (not required). Residence (*Local Residência* + *País Residência*) is collected separately via the guest-facing channel and passed to Skill 03. Any further field (e.g. `sex`) is flagged via drift if the live form turns out to require it.

## The two-pass agreement gate

The critical property of this skill is that **two structurally independent reads of the same document must agree** before any value is treated as verified. The "two passes" are not two Claude invocations on the same prompt. They are reads of two different parts of the passport, in two different formats, with different failure modes.

### Pass A: Visual fields read

Read the printed identity fields on the document. These are the human-readable values: a typeset name, a printed date of birth, a document number rendered in the country's preferred typography. They are what the host sees when she squints at the image today.

The prompt for Pass A focuses Claude on the visual content:

> "You are looking at a guest identity document: a passport, EU national ID card, or residence permit. Read only the printed, human-readable identity fields. Do not read the machine-readable zone (the lines of `<` characters). Return these fields as structured data: surname, given names, document type, document number, nationality, date of birth, **place of birth**, and **issuing country**. If any field is illegible, return `null` for that field rather than guessing."

### Pass B: MRZ read and parse

The Machine Readable Zone (MRZ) is the band of OCR-optimized characters at the bottom of every modern travel document's data page. It encodes the same identity fields in a strict, ICAO 9303-standardized format, with **check digits** that allow the read to self-validate.

Passport MRZ uses ICAO 9303 TD3 (two lines, 44 characters each). EU national ID cards use TD1 (three lines, 30 characters each). Residence permits typically use TD1 or TD2 depending on the issuing country. The parsing logic dispatches on detected MRZ shape.

The prompt for Pass B focuses Claude on the MRZ:

> "You are looking at a guest identity document. Read only the machine-readable zone (the lines of OCR-optimized characters with `<` fillers). Do not read the printed visual fields. Identify whether this is TD1 (three lines, 30 chars each, typically national ID), TD2, or TD3 (two lines, 44 chars each, typically passport) MRZ format. Transcribe the MRZ exactly, character by character. Then parse it per ICAO 9303 into the fields the MRZ encodes: surname, given names, document type, document number, nationality, date of birth, and **issuing country** (the issuing-state code). The MRZ does **not** encode place of birth. Verify every check digit. If any check digit fails, return a check-digit error rather than the parsed values."

### Comparison

Pass A returns all eight fields; Pass B returns the seven the MRZ encodes (place of birth is visual-only). The comparison logic walks the seven shared fields:

1. **Surname, given names**: case-insensitive, accent-folded, whitespace-normalized comparison. The MRZ uses `<` as a name separator and substitutes some Unicode characters; normalize both sides to a common form before comparing.
2. **Document type**: exact match (passport / national_id / residence_permit). Disagreement here indicates one of the reads misidentified the document; halt.
3. **Document number**: exact string match after stripping spaces. (Some countries pad with leading zeros; strip leading zeros only if both reads agree on the stripped form.)
4. **Nationality**: exact ISO-3166-alpha-3 match.
5. **Date of birth**: exact ISO-8601 match. (The MRZ uses YYMMDD with a century-disambiguation rule per ICAO 9303 §4.2.2.5; the parser must apply the rule correctly.)
6. **Issuing country**: exact ISO-3166-alpha-3 match. The MRZ encodes the issuing state in line 1; the visual page shows it on the data page.

**Place of birth** is read by Pass A only (it is not in the MRZ), so it cannot be cross-checked. The host verifies it directly against the document at sign-off.

If every shared field matches → **status: agreed**, proceed to host verification.
If any field disagrees → **status: halt_disagreement**, present both reads side by side to the host with the original image.
If the MRZ is missing or any check digit fails → **status: halt_unreadable**, escalate.

## Field-by-field reasoning

Each field has known failure modes that the skill must be aware of:

- **Surname / given names**: Different cultures order names differently. The MRZ enforces family-name-first with `<<` between surname and given names. The visual page may show given names first. The skill must reconcile to family-name-first (the SIBA expectation).
- **Document type**: Detected from MRZ shape (TD1/TD2/TD3) and from the visual document layout. Passport vs. national-ID disagreement is a hard halt. These have different SIBA reporting paths.
- **Document number**: Some countries pad with leading zeros; the MRZ may or may not include them depending on document age. Strip leading zeros only if both reads agree on the stripped form. If unclear, default to the MRZ form (which is the canonical machine-readable representation).
- **Nationality**: ISO-3166-alpha-3. The MRZ encodes it in a fixed position. The visual page may use a localized country name; the skill must normalize to the alpha-3 code.
- **Date of birth**: The MRZ uses two-digit years with a century disambiguation rule (years 00-29 → 2000s, 30-99 → 1900s, with adjustment for current date). Apply ICAO 9303 §4.2.2.5 exactly.

## Escalation triggers (when the skill halts and lands in the host's hands)

The skill halts on:

1. **MRZ unreadable** (no two lines of OCR characters detected at the bottom of the image). Likely cause: the image cropped out the MRZ, or the document is not a modern machine-readable passport. → Request a clearer photo of the full data page.
2. **MRZ check digit failed**. Likely cause: OCR misread of a critical character. The MRZ itself signals "do not trust this read." → Request a clearer photo.
3. **Visual fields illegible**. Likely cause: image too blurry, low light, glare on the laminate. → Request a clearer photo.
4. **Visual and MRZ disagree on any field**. Most consequential case: one of the two reads is wrong, and the skill cannot determine which. → Present both reads side by side to the host. She picks the correct value, types a custom value, or declines and requests a new image.
5. **Document is not a passport** (e.g., the guest sent a national ID card, a driving licence, or a screenshot of a phone wallet). → Request a passport specifically. The SEF reporting obligation applies to passports for non-EU/EEA guests.

Each escalation emits a structured audit event and presents the host with both the reason and the suggested next action.

## Host verification UI (what the runtime must surface)

When the agreement gate passes, the host sees:

- The original image, zoomable, with each extracted field annotated with a callout pointing to the source on the image
- The eight extracted fields, each editable, each marked with a confidence indicator (place of birth, read from the visual page only, is flagged as not MRZ-cross-checked)
- A single primary action: **Sign off and continue** (which moves to Skill 03)
- A secondary action: **Reject and request a new photo** (which generates a draft message back to the guest in their language; see escalation flow below)

When the gate fails (halt_disagreement), the host sees:

- The original image
- Two columns: **Visual read** | **MRZ read**, with the disagreeing fields highlighted
- For each disagreeing field, three actions: **Accept visual**, **Accept MRZ**, **Type a custom value**
- A final action: **Reject everything and request a new photo**

## Escalation flow (request clearer photo)

When the host chooses to escalate back to the guest, this skill drafts a polite message in the guest's language asking for a clearer photo of the full passport data page (including the bottom two lines). The draft uses the host's voice (see Skill 01 for voice handling). The host approves the draft before send. Send routes through the original inbound channel (WhatsApp, email, booking DM).

The escalation draft is intentionally short, warm, and specific. The template is generated per-language by the host's voice model. Reference baselines below.

**Portuguese (host's native; default for Portuguese-speaking guests):**

> *"Olá [primeiro nome], a foto chegou um bocadinho desfocada. Podia mandar uma mais nítida da página completa do passaporte (com as duas linhas de letras e números no final)? A lei portuguesa obriga a registar no próprio dia, e uma foto clara poupa-nos um passo. Muito obrigada!"*

**English:**

> *"Hi [first name], the photo came through a bit blurry on my end. Could you send a sharper one of the full passport page (with the two lines of letters and numbers at the very bottom)? Portuguese law needs me to log it within the day, and a clear photo saves us both a step. Thanks so much!"*

**French:**

> *"Bonjour [prénom], la photo est arrivée un peu floue chez moi. Pourriez-vous m'en envoyer une plus nette, de la page complète du passeport (avec les deux lignes de lettres et de chiffres tout en bas) ? La loi portugaise demande que je l'enregistre dans la journée, et une photo claire nous évite à tous les deux une étape. Merci beaucoup !"*

**Spanish:**

> *"Hola [nombre], la foto me llegó un poco borrosa. ¿Podrías enviarme una más nítida de la página completa del pasaporte (con las dos líneas de letras y números al final)? La ley portuguesa me obliga a registrarlo en el día, y una foto clara nos ahorra un paso a los dos. ¡Muchas gracias!"*

**German:**

> *"Hallo [Vorname], das Foto kam etwas unscharf bei mir an. Könnten Sie mir bitte ein schärferes der vollständigen Passseite schicken (mit den zwei Zeilen Buchstaben und Zahlen ganz unten)? Das portugiesische Gesetz verlangt, dass ich es noch am selben Tag registriere, und ein klares Foto erspart uns beiden einen Schritt. Vielen Dank!"*

Each baseline preserves the same five-beat rhythm: greeting, what went wrong, the specific ask (with the MRZ called out by description), the regulatory reason, the thank-you. The host's voice model adjusts register, formality, and idiom per guest demographic while keeping the structure stable. Additional languages (Italian, Dutch, Polish, Mandarin) follow the same template; native-speaker review of each language baseline is welcome via PR.

## Audit events emitted

This skill emits, at minimum:

- `extraction_attempted`: when the skill is invoked. Captures: reservation_id, image hash, source channel, received_at.
- `extraction_agreed` OR `extraction_disagreed`: outcome of the agreement gate. Captures: hashed extracted values from both passes, diff if disagreed. (Raw identity values are never emitted to the audit chain; only hashes, per the redacted-by-construction telemetry principle.)
- `host_verified`: when the host signs off on the agreed-and-presented payload. Captures: timestamp, host identifier, the host's required reason field (defaults to "verified" but the host can elaborate).
- `host_corrected`: if the host changed any field before signing off. Captures: which field was changed (not the values), so we can analyze systematic disagreement patterns post-pilot.
- `escalation_to_guest_requested`: if the host requested a clearer photo. Captures: reservation_id, draft message hash.

## What this skill does NOT do

To be explicit about scope:

- **Does not handle delivery of the secure-upload channel.** The guest-facing channel skill provides the secure upload page; this skill activates when an image arrives through that channel.
- **Does not submit to SIBA.** Skill 03 (`03-sef-submission.md`) consumes this skill's verified payload and drives the SIBA portal.
- **Does not collect the guest's home address.** That field is required by SIBA but is not present on most identity documents; the guest-facing channel collects it as a typed input alongside the document image.
- **Does not store document images long-term.** Per GDPR data minimization, the runtime is expected to discard the original image after the SIBA submission is acknowledged. Only the hashed extraction values land in the audit chain.
- **Does not handle driving licences.** Driving licences are not accepted by SIBA as identity documents for guest registration. If the guest sends one, the skill halts with `halt_unreadable / wrong_document_type` and the runtime requests a passport, national ID, or residence permit.
- **Does not draft the welcome message asking for the document.** That is Skill 01 (`01-pre-arrival-welcome.md`).

## Worked examples

> The JSON snippets below show the core cross-checked fields for brevity. v0.3 also extracts `issuing_country` (cross-checked via the MRZ) and `place_of_birth` (read from the visual page only, host-verified).

### Example A: Agreement, smooth verification (passport)

**Input:** Clear photo of a French passport uploaded through the guest-facing channel.

**Pass A visual read:**
```
{ surname: "DUBOIS", given_names: "MARIE CLAIRE",
  document_type: "passport", document_number: "12AB34567",
  nationality: "FRA", date_of_birth: "1985-03-12" }
```

**Pass B MRZ read (TD3, passport):**
```
P<FRADUBOIS<<MARIE<CLAIRE<<<<<<<<<<<<<<<<<<<
12AB345674FRA8503125F3008221<<<<<<<<<<<<<<04
```
Parsed (note: SIBA does not require `expiry_date`, so it is decoded for check-digit validation only; the **issuing country** IS surfaced to the payload, and **place of birth** comes from the visual read since the MRZ does not carry it):
```
{ surname: "DUBOIS", given_names: "MARIE CLAIRE",
  document_type: "passport", document_number: "12AB34567",
  nationality: "FRA", date_of_birth: "1985-03-12" }
```

**Comparison:** All cross-checked fields match. Status: `extraction_agreed`.
**Host UI:** all fields displayed with image callouts. Host clicks **Sign off and continue**. Status: `host_verified`. Payload passes to Skill 03 along with the residence details collected separately.

### Example B: Agreement, EU national ID card

**Input:** Photo of a German `Personalausweis` (national ID card) front side uploaded through the guest-facing channel.

**Pass A visual read:**
```
{ surname: "MUELLER", given_names: "ANNA SOPHIE",
  document_type: "national_id", document_number: "L01X00T47",
  nationality: "DEU", date_of_birth: "1992-07-04" }
```

**Pass B MRZ read (TD1, national ID, three lines of 30 characters):**
```
IDD<<L01X00T47<<<<<<<<<<<<<<<<
9207042F<<<<<<<<DEU<<<<<<<<<<<5
MUELLER<<ANNA<SOPHIE<<<<<<<<<<
```
Parsed:
```
{ surname: "MUELLER", given_names: "ANNA SOPHIE",
  document_type: "national_id", document_number: "L01X00T47",
  nationality: "DEU", date_of_birth: "1992-07-04" }
```

**Comparison:** All cross-checked fields match. Status: `extraction_agreed`. Same downstream flow as Example A.

### Example C: Disagreement on document number, host resolves

**Input:** Slightly blurry photo of a Brazilian passport.

**Pass A visual read:** `document_number: "FE123456"`
**Pass B MRZ read:** `document_number: "FE123450"` (last digit ambiguous in the visual but unambiguous in the MRZ which has a check digit)

**Comparison:** Disagree on `document_number`. Status: `halt_disagreement`.
**Host UI:** image + side-by-side comparison + the disagreeing field highlighted. The MRZ value's check digit validates correctly. The host clicks **Accept MRZ**. Status: `host_corrected`. Payload passes to Skill 03 with the MRZ value.

### Example D: MRZ check digit failure, escalation back to guest

**Input:** Glare on the bottom of a passport image, MRZ partially obscured.

**Pass B MRZ read:** transcribes the two lines, but the check digit for `document_number` fails the ICAO 9303 verification.

Status: `halt_unreadable` with reason `mrz_check_digit_failed`.
**Host UI:** explanation of the failure + a pre-drafted "could you send a clearer photo" message in French (the guest's language as detected from the booking).
**Host action:** approves the message; it sends. Status: `escalation_to_guest_requested`.

## Dependencies

- `claude-vision` connector for image input to Claude.
- An inbox connector that delivered the image (WhatsApp, email, or booking DM).
- Skill 03 (`03-sef-submission.md`) as the downstream consumer of the verified payload.
- The host's voice model (see Skill 01) for escalation message drafting.

## Versioning

This skill is `v0.3.0`, alpha, in active development against the first pilot. See `CHANGELOG.md` at the repo root for the full version history. Breaking changes to the output schema are possible during alpha and are called out in CHANGELOG.

**v0.3.0 changes vs. v0.2.0** (corrected against the live SIBA portal, 2026-05-20):
- **Restored `issuing_country`** (`País Emissor`): v0.2 wrongly removed it; the live form requires it (cross-checked via the MRZ issuing-state code)
- **Added `place_of_birth`** (`Local Nascimento`): required by the live form; read from the visual page only (not in the MRZ), so host-verified without an agreement cross-check
- Output payload grows from 6 to 8 fields; Pass A reads all 8, Pass B reads the 7 the MRZ encodes
- `expiry_date` remains dropped (not required)

**v0.2.0 changes vs. v0.1.0:**
- Renamed from `02-passport-extraction` to `02-guest-document-extraction` (broader scope)
- Output payload `PassportIdentityPayload` → `GuestIdentityPayload` (broader scope)
- Removed `country_of_issue` and `expiry_date` fields (not required by SIBA portal per primary-source confirmation)
- Renamed `passport_number` → `document_number`
- Added `document_type` field (`passport` | `national_id` | `residence_permit`)
- Pass B MRZ parsing now dispatches on TD1/TD2/TD3 shape detection (national ID + passport + residence-permit support)
- `home_address` field clarified as collected separately via the guest-facing channel (not on documents)
