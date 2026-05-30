# Example 01: French guest arrival, end to end

A complete worked transcript of the canonical flow: a booking lands, Obra
drafts the welcome, the guest's passport is read and cross-checked, and the
boletim de alojamento is filed on the SIBA portal. The host signs off at every
gate. Every step that touches the guest or the regulator waits for her.

This composes three skills in order:
[`01-pre-arrival-welcome`](../skills/01-pre-arrival-welcome.md) ->
[`02-guest-document-extraction`](../skills/02-passport-extraction.md) ->
[`03-siba-submission`](../skills/03-siba-submission.md).

> **All data here is synthetic.** The guest, the passport number, the property,
> and the confirmation number are invented for this walkthrough. No real
> person's identity data appears in this file. See the
> [examples README](README.md) for the synthetic-data policy.

## Cast and setup

- **Host:** Ana, who runs a two-bedroom AL (alojamento local) in Sintra,
  `Casa do Sobreiro`. She meets every guest at the door, so this runs in
  Skill 01's **Mode 1A** (in-person document capture, nothing routed through
  the booking platform's chat).
- **Guest:** Camille Laurent, booking from France for two nights.
- **Booking platform:** Airbnb. Obra reads the reservation from the host's
  connected inbox; it never logs in to Airbnb.
- **Host locale:** `pt-PT`. **Notification channel:** WhatsApp.

Wall-clock and token figures at the end are approximate, against representative
Claude pricing. They exist to show the shape of the cost, not to quote a bill.

---

## Stage 0: The trigger

Six days before arrival, an Airbnb reservation confirmation lands in Ana's
connected email inbox. Obra reads it and extracts the reservation context. No
skill has run yet; this is the inbound signal that starts the flow.

```
Reservation {
  reservation_id: "resv_8842",
  guest: { handle: "Camille Laurent", language_preference: "fr", origin_country: "FR" },
  arrival_date: "2026-06-12",
  departure_date: "2026-06-14",
  party_size: 2,
  nights: 2,
  booking_platform: "airbnb"
}
```

Obra surfaces a single card on Home: *"New booking, Camille Laurent, arriving
Fri 12 Jun. Want me to prepare the welcome?"* Ana taps yes.

---

## Stage 1: Pre-arrival welcome (Skill 01)

**Gate: host approves every word before the guest sees it.**

Ana has set her welcomes to send three days before arrival. At that time the
runtime invokes Skill 01 with the reservation, Ana's voice anchors, and the
property details. Because this is an Airbnb reservation and Ana meets guests at
the door, the runtime selects **Mode 1A**: a short warm welcome in the guest's
language, **no PDF, no link, no QR** in the platform chat. Document collection
happens in person at check-in.

Skill 01 drafts the message in French, in Ana's voice:

> *"Bonjour Camille, quelle joie de vous accueillir bientot a Casa do Sobreiro.
> Je vous retrouve a l'entree le 12 a partir de 15h pour vous remettre les cles
> et vous montrer la maison. La loi portugaise me demande d'enregistrer une
> piece d'identite le jour de l'arrivee; on s'en occupe ensemble a la porte, ca
> prend une minute. A tres vite!"*

- Audit event: `pre_arrival_draft_generated`.
- The draft lands in front of Ana for review (`host_review_required`). She reads
  it, changes nothing, and approves.
- Audit event: `host_approved`, then `pre_arrival_message_sent` once the
  booking-platform chat connector delivers it.

The chat message stays inside Airbnb's allowed conversation surface. No
identity-collection link of any kind has touched the platform. **Output:**
`status: "host_approved"`.

---

## Stage 2: Guest document extraction (Skill 02)

**Gate: two independent reads must agree, then Ana verifies and signs off.**

Friday arrives. Ana meets Camille at the door, opens the secure-upload page on
her own phone, and takes a photo of Camille's French passport data page. The
bytes go straight to Ana's local runtime; nothing transits Airbnb. The image
arriving invokes Skill 02.

The skill runs the **two-pass agreement gate**. These are two structurally
different reads, with different failure modes, not the same prompt twice.

**Pass A, visual read** (printed fields, MRZ ignored):
```
{ surname: "LAURENT", given_names: "CAMILLE", document_type: "passport",
  document_number: "19FH52098", nationality: "FRA",
  date_of_birth: "1990-09-23", place_of_birth: "LYON",
  issuing_country: "FRA" }
```

**Pass B, MRZ read and parse** (TD3, the two `<`-filled lines; visual fields
ignored), every ICAO 9303 check digit verified:
```
P<FRALAURENT<<CAMILLE<<<<<<<<<<<<<<<<<<<<<<<
19FH520982FRA9009235F3206142<<<<<<<<<<<<<<06
```
Parsed (the MRZ does not encode place of birth, so Pass B returns the seven
fields the MRZ carries):
```
{ surname: "LAURENT", given_names: "CAMILLE", document_type: "passport",
  document_number: "19FH52098", nationality: "FRA",
  date_of_birth: "1990-09-23", issuing_country: "FRA" }
```

**Comparison:** all seven shared fields match (name accent-folded and
whitespace-normalized, document number space-stripped, dates exact, nationality
and issuing country exact alpha-3). The check digits validate.

- Audit event: `extraction_attempted`, then `extraction_agreed` (hashes of both
  reads, never the raw values).

**Host verification UI:** Ana sees the photo with each field called out on the
image. `place_of_birth` ("LYON") carries the not-MRZ-cross-checked flag, so she
glances at the passport and confirms it directly. Everything is correct. She
taps **Sign off and continue**.

- Audit event: `host_verified`.

**Output:** the verified `GuestIdentityPayload` passes to Skill 03. Per GDPR
data minimization, the passport image will be discarded once the SIBA
submission is acknowledged; only hashes ever entered the audit chain.

The residence fields SIBA needs (`Local Residência` + `País Residência`) are
not on the passport. Camille gives them at the door and Ana types them into the
runtime: `{ place: "Lyon", country: "França" }`.

---

## Stage 3: SIBA submission (Skill 03)

**Gate: host reviews each filled boletim before Save, then explicitly sends the
list. The runtime never sends autonomously.**

The verified payload, the residence details, and the reservation context flow
into Skill 03.

```
SibaSubmissionInput {
  identity: { surname: "LAURENT", given_names: "CAMILLE", document_type: "passport",
              document_number: "19FH52098", nationality: "FRA",
              date_of_birth: "1990-09-23", place_of_birth: "LYON",
              issuing_country: "FRA" },
  residence: { place: "Lyon", country: "França" },
  reservation: { reservation_id: "resv_8842", arrival_date: "2026-06-12",
                 departure_date: "2026-06-14",
                 property: { al_registration_number: "12345/AL",
                             siba_credentials_ref: "vault://siba/ana" } },
  host: { host_id: "host_ana", host_locale: "pt-PT", notification_channel: "whatsapp" }
}
```

1. **Idempotency check.** Key = `resv_8842 + 2026-06-12 + sha256(document_number)`.
   No prior `submission_acknowledged` for this key. Proceed.
2. **Authenticated session** on `siba.ssi.gov.pt` using Ana's scoped
   credentials (entered interactively, never logged).
3. **Open the list.** SIBA's list/batch model: the connector opens the
   submission-lists page and lands on the boletim entry form.
4. **Fill the boletim.** A `Nova BAL` action unlocks the fields; the connector
   fills the **11 fields** from the payload (the text and dropdown fields
   directly; the three readonly calendar dates via the validated date routine).
   - Audit event: `submission_attempted`, then `submission_form_filled`
     (hashed payload of the filled fields, target endpoint, session id).

   | SIBA field | Value |
   |---|---|
   | Nome Completo | LAURENT CAMILLE |
   | Data de Nascimento | 23-09-1990 |
   | Local Nascimento | LYON |
   | Nacionalidade | França |
   | Local Residência | Lyon |
   | País Residência | França |
   | Número Documento | 19FH52098 |
   | Tipo Documento | PASSAPORTE |
   | País Emissor Documento | França |
   | Data de Check-in | 12-06-2026 |
   | Data de Check-out | 14-06-2026 |

5. **Host final review and send.** The filled boletim is rendered to Ana with
   each field labelled by source (Skill 02 / guest-facing / reservation). She
   reviews, **Saves** it to the list (Save is effectively irreversible on this
   portal, which is exactly why the review sits before it), waits out the
   ten-second dwell timer that fights rubber-stamping, and clicks
   **Enviar Lista**.
   - Audit event: `host_final_review_required`, then `host_final_approved`
     (dwell time observed), then `submission_submitted`.
6. **Receipt.** SIBA returns confirmation `SIBA-2026-06-12-AAA-007731` and a
   receipt PDF. The PDF is stored locally next to the audit chain; only its hash
   lands in the chain.
   - Audit event: `submission_acknowledged` (confirmation number, SIBA
     timestamp, receipt hash).
7. **Notification.** Ana gets a WhatsApp message:
   *"Boletim submetido. Número de confirmação SIBA-2026-06-12-AAA-007731.
   Recibo guardado localmente."*

**Output:** `status: "submitted"` with the receipt pointer.

---

## What landed in the audit chain

In order, hash-linked, each signed, raw identity values never present (only
hashes and field counts):

```
pre_arrival_draft_generated
host_review_required
host_approved
pre_arrival_message_sent
extraction_attempted
extraction_agreed
host_verified
submission_attempted
submission_form_filled
host_final_review_required
host_final_approved
submission_submitted
submission_acknowledged
```

If Ana, her accountant, or an inspector ever asks "what did Obra do for the
Laurent reservation, and who approved it," this chain is the answer, and it can
be exported and verified without revealing Camille's passport data.

## The three gates, in one view

- **Stage 1:** Ana approved the welcome message before it sent.
- **Stage 2:** two independent reads had to agree, then Ana verified the result
  against the document and signed off.
- **Stage 3:** Ana reviewed the filled boletim, saved it, dwelled the timer, and
  was the one who clicked send to the regulator.

At no point did Obra send a guest message, accept an identity value, or file
with the authority on its own. It did the work; Ana kept the decisions.

## Approximate cost and time

Representative, not a quote. Claude usage priced against current rates; the
exact figure varies with image size and model.

- **Host attention:** about two minutes total across the booking, spread over
  the welcome approval, the door-step verification, and the final SIBA review.
- **Claude usage:** roughly a few cents. The welcome draft is a small text
  generation; the two-pass document read is the largest single cost (vision on
  the passport image, twice); the SIBA fill is connector-driven and barely
  touches the model.
- **Wall-clock for the SIBA filing itself:** about 90 seconds end to end once
  Ana starts the final review.

## What this example does not cover

- The keybox / self-checkin case (Skill 01 Mode 1B, physical QR in the
  property) and the direct-booking case (Mode 2). This walkthrough uses Mode 1A.
- A disagreement at the agreement gate, or a host correction. See Skill 02's
  worked examples C and D.
- Portal drift or a SIBA validation rejection. See Skill 03's worked examples B
  and C.
- The in-stay and month-end flows (planned examples 02 and 03).
