---
name: 01-pre-arrival-welcome
version: 0.1.1
description: Drafts the pre-arrival welcome message in the guest's language and the host's voice. For direct-booking reservations, also produces a welcome PDF with an embedded QR encoding the secure-upload URL. For booking-platform reservations, the chat body alone is produced; document collection happens in-person at arrival OR via a physical QR placed inside the property, never via attachments routed through the platform's chat. Host approves every artifact before it reaches the guest.
tier: gated
requires_extraction_agreement: false
connector_dependencies:
  - booking-platform-chat (Airbnb / Booking / Vrbo messaging connector, text-only for these channels)
  - direct-booking-email (SMTP connector for direct-booking reservations)
  - pdf-renderer (welcome-pdf-template; only invoked for direct-booking + property-material delivery modes)
  - host-voice-model
  - guest-facing-channel (provides the per-reservation secure-upload URL)
human_signoff: true
regulatory_anchor:
  - Decreto-Lei n.º 128/2014 (Portuguese tourism law, identity reporting context for the request)
audit_event_types:
  - pre_arrival_draft_generated
  - pre_arrival_pdf_rendered
  - host_review_required
  - host_approved
  - host_edited
  - host_rejected
  - pre_arrival_message_sent
  - delivery_failed
---

# Skill: Pre-arrival welcome

> **Version 0.1.1** (2026-05-19, same evening): **revised to remove platform-policy risk for hosts**. The v0.1.0 version recommended embedding the secure-upload URL via QR code in a welcome PDF attached to booking-platform chat ("Shape K"). On review, that pattern bets the host's listing on the workaround surviving the platform's moderation evolution, which is not a bet we are willing to ask our pilot users to make. The architecture is the same; the delivery mechanism is corrected. Document collection on booking-platform reservations now happens via in-person handoff at arrival OR via a physical QR code placed inside the property, both outside the platform's content-moderation jurisdiction. The skill produces a welcome PDF (with QR) only for direct-booking reservations + the physical-property-material flow. The full platform-integrated delivery (Shape H, the structurally right answer for the keybox-majority case) is gated on platform partnership and is not in scope for this skill until that partnership lands. See [`docs/the-trust-gap.md`](../docs/the-trust-gap.md) for the strategic context.

## Purpose

A booking has just landed (or check-in is approaching, depending on the host's chosen schedule). The guest needs to:

1. Feel welcomed by the host, in their own language, in a way that does not read as machine-translated boilerplate
2. Understand the check-in logistics (when, where, how to enter)
3. Understand the regulatory context: Portuguese law obliges the host to register the guest's identity with the SIBA portal within a short window of arrival
4. Receive a structurally-trustable way to share the required identity document and home address, in a form that respects the booking platform's terms of service

This skill drafts the message that does all of those things. It does not send anything autonomously. The host reviews the chat message and the welcome PDF, signs off, and only then does the runtime deliver them through the booking platform.

The skill replaces what hosts currently do today: write a thank-you in their uncertain English, send it through the platform chat, hope the guest can read it warmly, and then later send a more awkward follow-up asking for passport details that the guest probably will not send. Obra produces a single coherent welcome message in the guest's native language, in the host's voice, with a structurally-trustable upload path embedded. The host signs off on every word before the guest sees any of it.

## Input

```
PreArrivalWelcomeInput {
  reservation: {
    reservation_id: string                // ties this welcome to a specific booking and the audit chain
    guest: {
      handle: string                      // guest's name as known on the booking platform
      language_preference?: ISO-639-1      // e.g., "fr" for French; from platform metadata if available
      origin_country?: ISO-3166-alpha-2    // e.g., "FR"; used as language-preference fallback
    }
    arrival_date: ISO-8601 date
    departure_date: ISO-8601 date
    party_size: number                    // total guests on this booking
    nights: number                        // computed; for the message context
    booking_platform: "airbnb" | "booking" | "vrbo" | "direct"
  }
  host: {
    host_id: string                       // for the audit chain
    voice_anchors: HostVoiceAnchors       // see "Host voice model" below
    property: {
      name?: string                       // optional friendly property name ("Casa do Sobreiro")
      address_short: string               // for the message ("Sintra, Portugal"); not the full address
      checkin_instructions: string        // how the guest enters (key code, host meets, etc.)
      checkin_time_from: "HH:MM"
      checkout_time_by: "HH:MM"
    }
    notification_channel: "whatsapp" | "email"
  }
  guest_facing_channel: {
    secure_upload_url: string             // per-reservation URL on the host's local runtime; treated as a secret
    expires_at: ISO-8601 timestamp        // when the URL stops accepting uploads (typically check-in + 24h)
  }
  schedule: {
    send_at: ISO-8601 timestamp           // when the host has chosen to deliver this message
    timing_descriptor: "on_booking" | "n_days_before_arrival" | "manual_now"
  }
}
```

The skill does not pick when to send. The host configures a schedule (or triggers manually); the runtime invokes the skill at the scheduled time. The skill produces the artifacts; delivery happens through the booking-platform chat connector after host approval.

## Output

```
PreArrivalWelcomeOutput =
  | { status: "drafted", chat_message: LocalizedMessage, welcome_pdf: PdfArtifact, audit_chain_pointer: string }
  | { status: "host_approved", chat_message: LocalizedMessage, welcome_pdf: PdfArtifact, host_reason: string, audit_chain_pointer: string }
  | { status: "host_edited", chat_message: LocalizedMessage, welcome_pdf: PdfArtifact, edits: EditDiff, host_reason: string, audit_chain_pointer: string }
  | { status: "host_rejected", reason: string, audit_chain_pointer: string }
  | { status: "delivery_failed", reason: "platform_error" | "rate_limited" | "moderation_flagged", details: string, audit_chain_pointer: string }

LocalizedMessage {
  body: string                            // the text the guest sees in platform chat
  language: ISO-639-1                     // the language the body is written in
  has_pdf_attachment: true                // by design, the chat body refers to the attached PDF
}

PdfArtifact {
  bytes: Buffer                           // the rendered welcome PDF
  language: ISO-639-1                     // language of the PDF body
  embedded_qr_target: string              // the URL the QR code encodes (matches input.guest_facing_channel.secure_upload_url)
  page_count: number                      // typically 1-2 pages
}
```

## Delivery modes

The skill's output depends on which delivery mode applies to the reservation. The mode is determined by the booking platform and the host's check-in style. Every mode is designed to be platform-policy-compliant. None of them route the secure-upload URL through a booking platform's content-moderated surfaces.

### Mode 1A: In-person at arrival

For reservations on Airbnb / Booking / Vrbo where the host meets the guest at check-in.

- **Chat artifact:** a short warm welcome in the guest's language, in the host's voice. **No PDF attachment, no URL, no QR.** The chat message stays inside the platform's allowed comms surface as ordinary host-guest conversation.
- **Document collection:** the host opens the secure-upload page on her own device at the door; the guest takes a photo of their identity document using the host's device (or scans the document on the spot, depending on the host's setup). The bytes go directly to the host's local runtime; nothing transits the platform.
- **first-pilot uses this mode.** Ana lives at her AL property and meets every guest face-to-face. Mode 1A is the highest-confidence, zero-platform-risk option for her first pilot.

### Mode 1B: Physical QR inside the property

For reservations on Airbnb / Booking / Vrbo where the host does **not** meet the guest at check-in (keybox / self-checkin operations, the majority of EU AL hosts).

- **Chat artifact:** same short warm welcome in the guest's language as Mode 1A. **No PDF attachment, no URL, no QR in the chat.**
- **Document collection:** the host prints a per-reservation welcome card (generated by this skill at host's request, sent to a printer on the host's premises). The card contains a QR encoding the secure-upload URL for this specific reservation. The host (or her cleaning crew) places the card inside the property: on the kitchen table, in the keybox welcome envelope, in the printed house manual. The guest arrives, finds the card, scans the QR with their phone, opens the upload page on the host's local runtime, uploads.
- **Why this respects platform policy:** the QR exists in physical material inside the property. The booking platform has no content moderation over what a host prints and places on her own kitchen table. The platform's chat carries only the friendly welcome message, no link delivery of any kind.
- **Trade-off:** the host has to physically place the welcome card before each reservation (or use a stable property-level QR with reservation-number identification at upload time; see "Stable vs. per-reservation QR" below). This is some operational overhead for the host, but it is the operational overhead the host already manages for cleaning + key handoff + welcome notes.

### Mode 1C: Guest-initiated WhatsApp (or other guest-chosen channel)

For platform-channel reservations where the guest has independently reached out to the host on WhatsApp (or another channel outside the booking platform's chat). This is a real and recurring case: guests find the host's WhatsApp through the property's website, through a Google search of the host's name, or because the booking platform's contact-sharing feature has unlocked phone numbers after booking. Some guests simply prefer WhatsApp to platform chat and message the host there first.

- **Chat artifact (in-platform):** the standard short warm welcome as in Mode 1A/1B. No reference to WhatsApp, no diversion attempt.
- **WhatsApp artifact (the welcome PDF with QR):** delivered through WhatsApp once the host has confirmed the guest reached out there first. The host's runtime offers Mode 1C as an option in the operator dashboard only when the host marks a guest's contact as having initiated WhatsApp contact. The skill does not auto-detect or auto-route to WhatsApp. The host's judgment is the gate.
- **Why this is not the same as Shape K:** Mode 1C does not deliver any link or attachment through the booking platform's chat. The booking-platform chat carries only the standard warm welcome. The welcome PDF is delivered through a separate channel the guest has chosen.
- **Documentation defense:** the host marks the reservation as "guest-initiated WhatsApp" in the runtime with an optional note describing how the contact happened. The audit chain captures this with a `mode_1c_guest_initiated_whatsapp` event so the host has a record if she ever needs to defend the channel choice to the platform.
- **Honest about residual risk:** booking platforms' automated detection cannot always distinguish guest-initiated from host-initiated off-platform contact. Mode 1C reduces but does not eliminate the platform-policy risk. The host accepts that residual risk per reservation when she chooses Mode 1C. The runtime makes the choice explicit and never automates it.

### Mode 2: Direct-booking via host-chosen channel

For reservations that do not come through a booking platform: direct-bookings via the host's own website, return guests booking directly, referrals, walk-ins. Because no booking platform sits in the conversation, no platform-policy constraint applies; the host picks whichever channel works best for that guest.

Supported delivery sub-modes:

- **Mode 2-Email:** the welcome PDF delivered via email to the address provided at booking. Standard SMTP connector. The link can also appear in the email body as plain text alongside the QR.
- **Mode 2-WhatsApp:** the welcome PDF delivered via WhatsApp to the phone number provided at booking. Common for direct-booking hosts whose guest communication runs primarily on WhatsApp. The host's WhatsApp Business account (or a WhatsApp-Web automation connector) ships the PDF as an attachment with the chat body as the message.
- **Mode 2-SMS:** less common, but supported for guests who provided a phone number but no WhatsApp / no smartphone. PDF generation reduces to a short SMS with a shortened URL alongside the QR (delivered via the SMS connector if the host has one configured).
- **Mode 2-Other:** host can configure custom delivery channels (Telegram, Signal, etc.) per their guest communication preferences. The skill generates the artifacts; the runtime routes them through whatever connector the host has set up.

The host picks the channel at the time of host review (or pre-configures a default channel per direct-booking guest type). Across all Mode 2 sub-modes the artifact set is identical, the chat/message body + the welcome PDF with QR, and only the connector that delivers them differs.

**Common pattern for direct-booking pilots beyond the first-pilot scope:** hosts who run their own booking website almost universally also use WhatsApp for guest communication. For these hosts, Mode 2-WhatsApp is the default; Mode 2-Email is the fallback when WhatsApp is not available.

### Mode 3: Platform-integrated [REQUIRES PARTNERSHIP, NOT YET AVAILABLE]

The structurally right answer for the keybox-majority case: the secure-upload URL is delivered inside the booking platform's UX through a partner integration the platform has explicitly authorized. The guest interacts with what looks like a native platform feature; the data routes directly to the host's local runtime; the platform is not in the data-processing chain.

- **This mode is gated on platform partnership.** No host should attempt to simulate it by other means. The skill explicitly does not produce a Mode 3 artifact until a platform partnership has been established and the integration is live.
- **The architecture is ready.** The trust-gap thesis exists to make this partnership conversation tractable. The first-pilot via Mode 1A is the evidence we present to platforms that the architecture works and is something they should adopt at scale.

### Stable vs. per-reservation QR (Mode 1B detail)

Mode 1B can deploy with either:

- **Per-reservation QR**: each guest gets a unique URL with a hard expiration. Higher friction for the host (print + place per arrival), tighter security model.
- **Stable property QR**: the QR encodes a URL that takes the guest to a per-property landing page where they identify their reservation (booking platform reference number or surname + arrival date) and then upload. Lower friction (place once, lasts forever), authentication via reservation identifier.

For first-pilot deployments, **per-reservation QR is the default** because the authentication path is simpler. Stable property QR is offered as an option once the host has accumulated enough trust in the runtime to manage the reservation-identification step securely.

## Host voice model

Obra's pre-arrival welcomes need to sound like the host, not like a corporate template translated by an algorithm. This is partly aesthetic (warmer guest experience) and partly functional (guests respond more to messages that read as personal). The host voice model is the runtime component that does this.

```
HostVoiceAnchors {
  about_self: string                      // 2-4 sentences the host wrote about herself + her hospitality style
  example_greetings: string[]             // 2-5 example messages the host wrote in her own language
  tone_descriptors: ToneDescriptor[]      // e.g., ["warm", "concise", "uses_local_idiom", "avoids_formal_salutations"]
  language_native: ISO-639-1              // the host's native language
  languages_comfortable: ISO-639-1[]      // additional languages the host can correct generated drafts in
}
```

The voice anchors are collected during the host's onboarding. They are short: the goal is not to train a personalized model but to give Claude enough specific stylistic signal to mimic the host's idiom on top of its multilingual generation. Over time, the host's accept/edit signals on generated drafts tune the voice anchor set: edits that recur are extracted into the tone descriptors; edits that are one-offs remain one-offs.

The voice model is not a separately-trained model. It is a structured prompt-engineering layer over Claude. The host's voice anchors are included in the system prompt for every generation; the generation produces output adapted to the anchors. This is operationally simple and architecturally compatible with the local-first deployment (no separate ML pipeline to operate per host).

## The generation flow

### Step 1: Mode and language selection

The runtime determines the delivery mode for this reservation:

- **Mode 1A** if `booking_platform` is `airbnb` / `booking` / `vrbo` and `host.checkin_style` is `in-person`
- **Mode 1B** if `booking_platform` is `airbnb` / `booking` / `vrbo` and `host.checkin_style` is `keybox` (and the host has enabled physical-QR welcome cards in her configuration)
- **Mode 2** if `booking_platform` is `direct`
- **Mode 3** if `booking_platform` is `airbnb` / `booking` / `vrbo` AND a platform-integrated upload channel has been negotiated for that platform [NOT YET AVAILABLE in any platform as of v0.1.1]

The runtime selects the language for the welcome based on (in order of preference):

1. `reservation.guest.language_preference` from the booking platform's metadata
2. `reservation.guest.origin_country` mapped to the most likely primary language
3. The platform-provided guest language fallback
4. English as the universal fallback

### Step 2: Chat body generation (all modes)

Claude generates the chat body in the selected language, using the host's voice anchors as the prompt's stylistic context.

- **For Mode 1A and 1B (platform-channel):** the body is constrained to be short (typically 50-100 words), warm, and **explicitly forbidden from referencing any attachment, link, QR code, or upload mechanism.** The prompt asserts this constraint at generation time and the runtime re-asserts it at post-generation validation.
- **For Mode 2 (direct-booking email):** the body is slightly longer (80-150 words) and references the attached welcome PDF in a single sentence in the guest's language.

### Step 3: PDF generation [only for Mode 2 and Mode 1B]

For Mode 2 (direct-booking) and Mode 1B (physical welcome card placed inside the property), a welcome PDF is generated. Skipped for Mode 1A.

Body content: the warm welcome, check-in logistics, the regulatory context, the upload-instructions copy, the architectural-trust reassurance paragraph, the friendly close. Length typically 250-500 words depending on language. Same voice anchors as the chat body; longer-form expression.

### Step 4: PDF rendering [only for Mode 2 and Mode 1B]

The PDF renderer composes:

- The generated PDF body (formatted with the host's chosen template: typography, colors, optional property logo)
- The QR code encoding `guest_facing_channel.secure_upload_url`, with the URL's expiration time printed below the QR in human-readable form
- A footer line in the guest's language: *"For technical support, please reply to your booking conversation."*

The QR code is generated locally by the renderer (a deterministic library call; no external service). For Mode 1B specifically, the renderer also produces a print-ready format (A5 or postcard size; cuttable, foldable layouts depending on host preference) so the host can print the card and place it in the property's welcome materials.

### Step 5: Host review

The host sees, in her own language (the host's native language, regardless of the guest's language):

- The chat body translated back to the host's native language (so she can review what the guest will read without depending on her own foreign-language comprehension)
- The chat body in the guest's language (so she can verify it visually)
- The full PDF rendered to a preview (Mode 2 and Mode 1B only)
- A clear visual indicator of which mode applies and what the delivery mechanism is
- The metadata: which language was chosen, which voice anchors were applied, when the message is scheduled to send
- A primary action: **Approve and schedule** (with dwell timer per the connector framework's gating rule)
- A secondary action: **Edit before sending** (which opens a structured edit surface for both the chat body and the PDF body)
- A tertiary action: **Reject and explain** (which abandons this draft and emits a reason)

### Step 6: Delivery

On host approval:

- **Mode 1A:** the booking-platform chat connector delivers the chat body only. The host gets a reminder, scheduled for the day of check-in, to open the secure-upload page on her device before the guest arrives.
- **Mode 1B:** the booking-platform chat connector delivers the chat body only. The runtime prints the welcome card (or sends a print-ready PDF to the host's chosen printer); the host or her cleaning crew places it in the property's welcome materials before check-in.
- **Mode 2:** the direct-booking-email connector delivers the email with the PDF attachment to the guest's provided email address.
- **Mode 3:** [not yet available]

The connector captures the delivery-acknowledgment and emits `pre_arrival_message_sent`. If delivery fails (rate limit, platform error, moderation flag, email bounce), the skill halts with `delivery_failed` and presents the error to the host with retry options.

## What the chat body looks like (per-language baselines, per delivery mode)

The host's voice anchors will alter these significantly. The baselines below show the rhythm the runtime aims for; the actual generated text adapts to the specific host's voice.

### For Mode 1A and 1B (platform-channel reservations: Airbnb / Booking / Vrbo)

The chat body is a short warm welcome. **It does not reference any attachment, link, or document upload.** Document collection happens in person or via the physical QR inside the property; the chat carries only the relational warmth.

**Portuguese:**

> *"Olá [primeiro nome], obrigada pela reserva! Estamos muito felizes por vos receber no dia [data de check-in]. Se tiverem alguma dúvida antes da chegada, é só responder por aqui. Até breve!"*

**English:**

> *"Hi [first name], thanks so much for booking! We're really looking forward to having you on [arrival date]. If anything comes up before your stay, just reply here. See you soon!"*

**French:**

> *"Bonjour [prénom], merci beaucoup pour votre réservation ! Nous avons vraiment hâte de vous accueillir le [date d'arrivée]. Si vous avez la moindre question avant votre arrivée, n'hésitez pas à répondre ici. À très bientôt !"*

**Spanish:**

> *"Hola [nombre], ¡muchas gracias por tu reserva! Tenemos muchas ganas de recibirte el [fecha de llegada]. Si necesitas cualquier cosa antes de la llegada, responde por aquí. ¡Hasta pronto!"*

**German:**

> *"Hallo [Vorname], vielen Dank für Ihre Buchung! Wir freuen uns sehr, Sie am [Anreisedatum] empfangen zu dürfen. Falls vor Ihrer Anreise etwas auftaucht, antworten Sie einfach hier. Bis bald!"*

### For Mode 2 (direct-booking reservations: email channel)

The email body references the attached welcome PDF, which contains the QR code encoding the secure-upload URL. Platform-policy constraints do not apply.

**Portuguese (email body):**

> *"Olá [primeiro nome], obrigada pela reserva! Estamos ansiosos por vos receber no dia [data de check-in]. Anexei um pequeno documento de boas-vindas com tudo o que precisam saber para o check-in e a forma de partilhar comigo o documento de identificação que a lei portuguesa exige que eu registe. Se houver alguma dúvida, é só responder a este email. Até breve!"*

(Other languages follow the same pattern: a friendly relational opener + an explicit reference to the attached welcome document + a polite close. The full set is generated by the same template; the structurally important addition over Mode 1A/1B is the single sentence pointing at the attachment.)

**Italian, Dutch, Polish, Mandarin:** generated by the same template; native-speaker review of each language is welcome via PR before that language ships to production.

## Failure modes

**Voice-anchor disagreement.** The host's voice anchors are inconsistent (e.g., one example greeting is very formal, another is very casual). The runtime warns the host at onboarding-time when the anchors do not converge to a coherent voice. The skill can still generate, but the host should expect to edit more during the first few drafts.

**Language not supported.** The guest's language preference is one we have not generated a baseline for. The runtime falls back to English with a note in the host-review UI: *"Generated in English because the guest's preferred language (Tagalog) does not yet have a tested baseline. Consider asking the guest if English is acceptable, or rejecting this draft and contacting them in a language she speaks."*

**PDF renderer fails.** Rendering pipeline error (font missing, image render failure, QR generation library error). The skill halts with `pre_arrival_pdf_rendered: failure` and surfaces the technical error to the operator dashboard for support attention. The host sees: *"PDF rendering failed; we've notified support. The chat-body draft is ready and can be sent without the PDF if you would prefer, though the guest will not have the secure-upload channel."*

**Booking platform moderation flag.** The platform's automated systems flag the chat body or the PDF for unknown reasons. The skill halts with `delivery_failed: moderation_flagged` and the host sees the platform's stated reason. This is rare but expected at low frequency; the operator dashboard tracks frequency per platform for connector-maintenance attention.

**Schedule slippage.** The host configured the send time for a specific date but the runtime missed it (host's machine offline; runtime crashed; system clock disagrees with the scheduler). The skill detects the slippage at next-startup and surfaces it: *"This welcome was scheduled to send 14 hours ago but did not. Do you want to send it now, send it at the original time + offset, or skip?"*

## Audit events emitted

- `pre_arrival_draft_generated`: chat body + PDF body generated. Captures: reservation_id, language used, host voice-anchor version, scheduled send time, hash of the draft bodies.
- `pre_arrival_pdf_rendered`: PDF rendered with QR code embedded. Captures: hash of the PDF bytes, expiration time of the embedded URL, page count.
- `host_review_required`: preview presented to host. Captures: timestamp, dwell time required.
- `host_approved` OR `host_edited` OR `host_rejected`: host's decision. Captures: timestamp, dwell time observed, host's required reason, edit diff if edited.
- `pre_arrival_message_sent`: booking-platform connector delivered the message + PDF. Captures: platform send-receipt id, delivered_at timestamp.
- `delivery_failed`: delivery did not complete. Captures: error class, error details, retry-after if applicable.

Per the redacted-by-construction principle, the raw chat-body text and PDF-body text never leave the host's local audit chain. The operator-side audit chain receives only hashes + metadata (language, length, send timing, host action).

## What this skill does NOT do

- **Does not send the message autonomously.** Every pre-arrival welcome is gated; the host approves before delivery.
- **Does not generate the secure-upload URL.** That comes from the guest-facing-channel skill, which produces a fresh per-reservation URL with an explicit expiration.
- **Does not handle the guest's response.** When the guest uploads via the URL, the guest-facing-channel skill receives the document and invokes Skill 02 (extraction). When the guest replies in chat, a separate skill (in-stay support) handles the conversation.
- **Does not put hosts at platform-policy risk.** For booking-platform reservations (Mode 1A and 1B), the platform's chat carries only a short warm welcome: no attachment, no link, no QR, no off-platform diversion attempt. Document collection happens through channels outside the platform's content-moderation jurisdiction (in-person at arrival, or via physical material inside the property, or via a channel the guest independently initiated). The earlier v0.1.0 design that recommended embedding the URL via QR in a PDF attached to platform chat ("Shape K") has been removed because it bets the host's listing on the workaround surviving platform moderation evolution. We will not ask pilot users to make that bet.
- **Does not auto-detect or auto-route to off-platform channels.** Mode 1C (guest-initiated WhatsApp) is only available when the host explicitly marks a reservation as guest-initiated. The skill never routes around platform terms autonomously.
- **Does not work around platform terms.** When the platform-integrated delivery (Mode 3) is required, the skill says so explicitly and provides no other mechanism. The architecture is designed to make Mode 3 desirable to the platforms; it is not designed to simulate Mode 3 from the host side.
- **Does not pick the timing autonomously.** The host configures the schedule (on-booking, N days before arrival, manual). The skill respects whatever the host configured.
- **Does not translate the host's voice across languages dishonestly.** If the host's voice anchors are very colloquial Portuguese, the German generation will be appropriately warm-and-personal in a German register, not literally idiomatic Portuguese expressions translated word-by-word. Cultural register is matched; specific idioms are not.

## Worked examples

### Example A: French guest, Airbnb reservation, Mode 1A (in-person at the host's property)

**Input:** Reservation arrives 2026-06-12 via Airbnb. Guest from Lyon. Arrival 2026-06-15. Ana's voice anchors loaded. Ana's `checkin_style: "in-person"` (she meets every guest in person at check-in). Host's chosen schedule: send 5 days before arrival.

**Step 1 (Mode):** 1A (Airbnb + in-person check-in). **Language:** French.

**Step 2 (Chat body generation):**
> *"Bonjour Pierre, merci beaucoup pour votre réservation ! Nous avons vraiment hâte de vous accueillir le 15 juin. Si vous avez la moindre question avant votre arrivée, n'hésitez pas à répondre ici. À très bientôt !"*

No reference to attachment, link, or document upload. Just a warm welcome.

**Step 3 (PDF generation):** SKIPPED for Mode 1A.

**Step 5 (Host review):** Ana sees the chat body (with back-translation to Portuguese for verification), the mode metadata ("Mode 1A: in-person at arrival; you will collect the document on her phone/tablet at check-in"), and a reminder schedule. Dwells the 10-second timer, enters "ok" as her reason, clicks **Approve and schedule**.

**Step 6 (Delivery):** 2026-06-10 09:00 WEST, the Airbnb chat connector delivers the chat body. `pre_arrival_message_sent` lands in the audit chain.

**At check-in (2026-06-15):** Ana meets Pierre at the door. She opens the secure-upload page on her phone (the runtime alerted her this morning); Pierre takes a photo of his passport using her phone. The bytes go directly to Ana's local runtime. Skill 02 runs on the image. Ana verifies. Skill 03 submits to SIBA within the legal window.

### Example B: Spanish guest, direct booking, Mode 2 (email with PDF + QR)

**Input:** Reservation arrives 2026-08-05 via the host's own website (a future direct-booking pilot beyond the first-pilot scope). Guest from Madrid. Arrival 2026-08-12. Email provided at booking: `[email protected]`.

**Step 1 (Mode):** 2 (direct booking). **Language:** Spanish.

**Step 2 (Chat body (email body) generation):**
> *"Hola Carmen, ¡muchas gracias por tu reserva! Estamos deseando recibirte el 12 de agosto. He adjuntado un breve documento de bienvenida con todo lo necesario para el check-in y una forma segura de enviarme el documento de identidad que la ley portuguesa me obliga a registrar. Si tienes cualquier duda, responde por aquí. ¡Hasta pronto!"*

The email body explicitly references the attached welcome PDF; platform policy does not apply here, the link can also appear in the PDF.

**Step 3 (PDF generation):** longer welcome (~400 words) in Spanish with the canonical sections.

**Step 4 (PDF rendering):** 1-page PDF with the body + QR code encoding Carmen's per-reservation upload URL + expiration date.

**Step 5 (Host review):** Ana sees both artifacts in preview, approves, schedules.

**Step 6 (Delivery):** SMTP connector sends the email. Carmen receives it, scans the QR on her phone, uploads her passport image before arrival. Skill 02 + Skill 03 run when she does. By the time Carmen arrives, the SIBA submission is queued and ready.

### Example C: German guest, Airbnb reservation, keybox host, Mode 1B (physical welcome card)

**Input:** Reservation on Airbnb at a hypothetical second-pilot host's keybox property. Guest from Berlin. Arrival 2026-09-20. Host's `checkin_style: "keybox"`; she has enabled physical-QR welcome cards.

**Step 1 (Mode):** 1B. **Language:** German.

**Step 2 (Chat body):** the standard Mode 1B welcome: short, warm, no attachment reference.

**Step 3 + 4 (PDF/card generation):** a print-ready postcard layout containing the German PDF body + QR code + the URL expiration printed below. Sent to the host's chosen printer (or held in queue for the host to print on her own).

**Step 5 (Host review):** host sees the chat body, the postcard preview, the mode metadata ("Mode 1B: physical welcome card; place inside the property before check-in"). Approves.

**Step 6 (Delivery):** Airbnb chat carries only the warm welcome. The host (or her cleaning crew) prints the welcome card and places it on the kitchen table before the guest arrives. Guest arrives, finds the card with the QR, scans, uploads via the host's local runtime. Document collection happens within the SIBA window without any platform-policy violation.

### Example D: French guest, Airbnb reservation, guest-initiated WhatsApp, Mode 1C

**Input:** Same as Example A but Pierre has independently messaged Ana on WhatsApp two days before arrival asking about parking. Ana has not given him her WhatsApp number directly through Airbnb chat; he found it through her property's listing on a small Portuguese AL directory site that she also lists on.

**Ana's action in the runtime:** marks Pierre's reservation as "guest-initiated WhatsApp" with a note: *"He messaged me first about parking; here's his number."*

**Runtime response:** unlocks the Mode 1C option for this reservation. Ana can now choose Mode 1C (send the welcome PDF via WhatsApp) in addition to Mode 1A (the default for her).

**Ana chooses Mode 1C** because Pierre is clearly comfortable with WhatsApp and pre-arrival upload would be easier than in-person at the door (it's been a long day).

**Step 3 + 4:** PDF generated and rendered as in Example B but smaller-formatted for phone viewing.

**Step 5 (Host review):** Ana sees the chat body (which she will NOT send through Airbnb), the PDF, and the mode metadata showing the audit-chain capture of guest-initiation documentation. Approves.

**Step 6 (Delivery):** WhatsApp connector delivers the PDF to Pierre's number. Pierre scans the QR, uploads. The Airbnb chat carries only Ana's standard short warm welcome from earlier; no link, no diversion attempt, no policy violation visible to the platform's automated detection.

### Example E: Host rejects with tone feedback (voice learning)

**Input:** Same as Example A but the generated chat body opens with "Cher Pierre" (a formal "Dear Pierre" salutation) and Ana feels her usual register is warmer.

**Host action:** Ana clicks **Reject and explain**. Reason: *"Too formal for me; I usually start with 'Bonjour' or 'Olá'. Could you redo it warmer?"*

**Runtime response:** The rejection is captured as `host_rejected` with Ana's reason. The reason text is added to the host voice anchors' tone descriptors (extracted by Claude into structured form: `prefer_warm_opener: true`, `avoid_formal_salutations: true`). The skill is re-invoked; the next draft uses the updated anchors. Ana approves the second draft. The voice model has learned.

## Dependencies

- **Booking-platform chat connector** (`airbnb-chat`, `booking-chat`, etc.): delivers the chat body + PDF attachment through the platform's allowed comms surface.
- **PDF renderer** (`welcome-pdf-template`): composes the welcome PDF with embedded QR code.
- **Host voice model**: structured prompt-engineering layer over Claude that adapts generation to the host's voice anchors.
- **Guest-facing-channel skill**: provides the per-reservation secure-upload URL that the QR encodes.
- **Skill 02** ([`02-passport-extraction.md`](02-passport-extraction.md)): the downstream consumer of whatever the guest uploads through the URL embedded in the QR.

## Versioning

`v0.1.1`, alpha. See `CHANGELOG.md` at the repo root for the full version history.

**v0.1.1 changes vs. v0.1.0 (same-evening correction):**
- **Removed** the Shape K design (QR-in-PDF attached to booking-platform chat) as the recommended delivery mechanism for platform-channel reservations. Same-evening review 2026-05-19 identified that this pattern bets the host's listing on the workaround surviving Airbnb's moderation evolution; not a bet we are willing to ask pilot users to make.
- **Added** four delivery modes that respect platform policy:
  - Mode 1A: in-person at arrival on the host's device (first-pilot path)
  - Mode 1B: physical QR-encoded welcome card placed inside the property (keybox-host path)
  - Mode 1C: guest-initiated WhatsApp (host's judgment call, never auto-routed)
  - Mode 2: direct-booking via host-chosen channel (email, WhatsApp, SMS, etc.)
- **Documented** Mode 3 (platform-integrated delivery) as the structurally right answer for the keybox-majority case, gated on platform partnership and not yet available.
- **Updated** chat-body baselines for Mode 1A/1B to remove any reference to attachments, links, or upload mechanisms (the platform-channel chat is now pure relational warmth; document collection happens via the modes that route outside the platform's content jurisdiction).
- **Updated** worked examples to cover Mode 1A (Example A, Ana + Pierre), Mode 2 (Example B, direct-booking Spanish guest), Mode 1B (Example C, keybox host + German guest), Mode 1C (Example D, guest-initiated WhatsApp), and the voice-learning case (Example E).

Multilingual baselines (PT, EN, FR, ES, DE) tested against Claude's native multilingual generation. Italian, Dutch, Polish, Mandarin queued for native-speaker review before promotion to tested-baseline status. Voice-anchor onboarding flow expected to evolve significantly as the first pilot generates real edit-pattern data.
