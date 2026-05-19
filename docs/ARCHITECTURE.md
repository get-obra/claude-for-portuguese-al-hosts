# Architecture

*The mechanism behind the narrative.*

> **Read this with:** [The trust gap](the-trust-gap.md) (the narrative reasoning) and the skill files in [`/skills`](../skills/) (the operational shape).
>
> **Status:** alpha, in active development. Sections marked with [PLANNED] describe properties the architecture is being built toward; sections without that tag describe properties already implemented in the closed monorepo and being incrementally exposed in this open-source pack.

---

## What this document is

[The trust gap](the-trust-gap.md) makes the case that EU short-term rental compliance is structurally broken at the guest-host trust-channel layer, and that the architecture required to close it has four non-optional properties. The essay asserts those properties. This document defends them.

It is the engineer-readable mechanism document. The audience is anyone who wants to understand *how* Obra is built, *why* each architectural choice exists, and *what kinds of failure* each choice prevents. It is the document a thoughtful technical evaluator (Anthropic engineer, regulator's technical advisor, fellow builder considering a similar problem, contributor preparing a pull request) needs to assess whether the work is real.

It is **not** a complete implementation specification. The full specifications live in component-level documents inside the private monorepo and are being progressively published here as they stabilize. This document gives the shape; the full code lives elsewhere; the public reference skills, connectors, and example transcripts in this repo illustrate the shape concretely.

---

## The four architectural properties, defended

The essay states the minimum specification for an architecture the guest will trust enough to engage with:

1. The guest's image is uploaded to a runtime the host operates, not to platform storage, not to vendor storage.
2. Inference runs under the host's own API key, not the operator's.
3. The audit trail is hash-chained and owned by the host.
4. The telemetry that flows back to whoever is operating the system is redacted by construction.

The essay claims these are not optional. This section explains why, by examining the weaker alternatives a thoughtful engineer would propose, and showing where each falls short of the trust shape the problem requires.

### Property 1 — Guest data is uploaded to a runtime the host operates

**Statement.** When a guest uploads a passport image, the bytes of that image traverse the network from the guest's device directly to a runtime running on the host's premises. The image does not transit through a vendor's server. It does not transit through the booking platform's server. The host's hardware is the first server to receive the image and the only server that stores it persistently. (Anthropic's commercial API endpoint receives the image transiently during inference — see [the data path](#the-data-path) below — under the host's own API key, with Zero Data Retention available for eligible accounts.)

**Why this is necessary.** The guest's refusal to send a passport image through the booking platform's chat is not arbitrary. It is a response to the *centralization* of the channel. The objection is not to "this specific company" — it is to "any centralized intermediate." A solution that routes the image through any centralized intermediate inherits the same trust objection. The objection is structural; the solution must be structural to match it.

**The alternatives that fail this test:**

- **Hosted SaaS with strong encryption.** The vendor still has access to the data (or could acquire it under subpoena, court order, employee misconduct, or breach). Encryption-in-transit and encryption-at-rest protect against external observers; they do not protect against the vendor itself. The guest's trust objection does not distinguish between "the vendor will misuse my data" and "the vendor could misuse my data" — both are sufficient to refuse.
- **EU data residency.** Solves the geographic-jurisdiction concern, not the trust concern. The data still lives on a vendor's server; the vendor's promises about access are still policy-based.
- **Customer-managed keys.** Sounds like the data is under the customer's control, but in practice the keys are managed by the vendor's key-management service, and the vendor still operates the runtime that decrypts the data to process it. The trust position is unchanged.
- **On-premises vendor product (hosted-on-customer-hardware but vendor-controlled).** Closer, but still depends on the vendor's update mechanism, telemetry collection, license-server beacons, support-debugger access, and a dozen other ways the vendor retains effective access. The host does not actually control what the software does on their hardware.

**How Obra implements it.** The guest-facing channel (the secure upload page the guest interacts with) runs as a process on the host's machine, served from a local port over an authenticated session. The bytes of the passport image arrive at that process. They are written to the host-resident audit chain. They are transmitted to Anthropic's commercial API for inference under the host's own API key (Anthropic's commercial commitments apply: no training on customer API data, Zero Data Retention available for eligible accounts) and the response returns to the host's runtime. They are not transmitted anywhere else — not to the operator (Obra), not to the booking platform, not to a vendor's server, not to a SaaS provider. The only third-party network destination the image bytes ever reach is Anthropic, transiently, for inference. The SIBA portal and the operator-side telemetry channel carry payloads from which the image bytes are structurally absent. See [the data path](#the-data-path) below for the full end-to-end walkthrough.

### Property 2 — Inference runs under the host's own API key

**Statement.** The Claude invocation that reads the passport image, extracts the identity data, generates draft communications, or makes any reasoning step about guest-specific content runs against an API endpoint authenticated as the host's, with the host's API key, charged to the host's account, with the prompt-and-response pair logged only in the host's local audit chain. Anthropic's commercial-API commitments apply: no training on customer API data, Zero Data Retention available for eligible accounts. The image bytes transit to Anthropic only for the duration of the inference call, and only inside the host's contractual relationship with Anthropic — the operator (Obra) is not party to that relationship.

**Why this is necessary.** The guest's passport image, the guest's name, the guest's date of birth, and the host's reasoning about them are all special-category personal data under GDPR Article 9. The architecture must minimize the parties who see this data. The runtime that processes the data must be operated on behalf of the data subject's chosen counterparty (the host) and not on behalf of an intermediate vendor whose interests may diverge.

**The alternatives that fail this test:**

- **Vendor-operated multi-tenant Claude proxy.** Vendor sees the prompts and responses; vendor's logs include guest data; vendor's billing reflects per-customer usage in ways that aggregate identifiable patterns. Same structural objection as Property 1.
- **Per-customer dedicated vendor Claude proxy.** Better, but vendor still operates the proxy, still has emergency-access, still has support-debug-access, still aggregates uptime telemetry. Effective access remains.
- **On-premises LLM inference (not Claude).** Solves the vendor-access concern, but the host has to operate inference at production quality for a frontier model, which is not feasible for most small operators, and a non-frontier model fails the capability bar the work actually requires.

**How Obra implements it.** Anthropic publishes Claude as an API service that the host's runtime calls directly with the host's own API key. The host owns the key (stored in the operating system's secrets store, never logged, scoped to the connector that needs it). The runtime is operated by the host's own machine. Anthropic, as the API provider, is in the data-processing chain as a sub-processor under a Data Processing Agreement; this is unavoidable for any Claude-powered workflow and is the same posture as any other production Claude deployment. The architectural property here is that *no fourth party* sits between the host and Anthropic. The vendor (Obra) operates the runtime software but does not operate the runtime instance for the host; Obra does not have access to the host's API key, the host's prompts, or the host's responses.

### Property 3 — Audit trail hash-chained and owned by the host

**Statement.** Every action the runtime takes on the host's behalf emits an audit event whose payload is hash-chained to the prior event. The chain is stored on the host's hardware. The host owns the bytes. The host can produce the chain on demand for a regulator, a tax authority, a data subject under a GDPR Article 15 access request, or her own records, without anyone's permission.

**Why this is necessary.** A vendor-owned audit chain is subject to the vendor's data-retention policies, the vendor's subpoena response posture, the vendor's interpretation of the customer's instructions, and the vendor's continued operational existence. A regulator who suspects a host of non-compliance and seeks evidence cannot trust an audit produced by the vendor whose business depends on the host's continued patronage. A customer-owned audit chain is the only chain that survives the failure modes of the rest of the architecture (vendor bankruptcy, vendor acquisition, vendor policy change, regulator-vendor dispute).

**The alternatives that fail this test:**

- **Vendor-stored audit with customer-export-on-demand.** Subject to vendor uptime, vendor retention policy, vendor cooperation. The export is the customer's only artifact, but if the vendor is uncooperative or non-existent, the customer cannot reconstruct.
- **Centralized blockchain.** Trades vendor-dependency for chain-operator-dependency. The chain operator (or a majority of validators on a public chain) becomes the new trusted party. Public chains also have privacy implications that conflict with special-category data handling.
- **Plain log files on the host with no integrity guarantees.** Trivially tampered with after-the-fact. Regulator cannot rely on a log the host could have edited.

**How Obra implements it.** Each audit event is a JSON object containing a payload-specific hash and a `previous_event_hash` field pointing at the prior event. The first event in a client's chain ("the genesis") is pinned by the operator-side ingester at first sight; any attempt to retroactively rewrite history is detectable by comparing against the pinned genesis (the ingester knows the genesis hash; the host cannot edit history without producing a chain that does not start with the pinned hash). The chain itself lives in a directory on the host's hardware (the JSONL files are append-only by convention and by file-system permissions); the host can export the chain at any time as a single file or stream. The witness-of-record layer ([planned, Phase 2a]) extends this with externally-anchored timestamping for stronger anti-replay guarantees.

### Property 4 — Redacted-by-construction telemetry

**Statement.** Anything that flows from the host's runtime to the operator (Obra), to a vendor, or to any party outside the host's hardware, is restricted at the schema level to non-sensitive metadata: event types, counts, hashes, structured categorical decisions (without their reasons), and the timing of operations. Special-category data, business-content payloads, and the host's working reasoning never appear in any operator-side surface.

**Why this is necessary.** Operating a managed-service product requires telemetry — the operator needs to know if a host's runtime crashed, if a workflow failed, if a connector encountered drift, if a host needs support. The naive approach is to capture rich logs that include "everything that happened" so the operator can debug effectively. The naive approach turns the operator into a second centralized data holder that re-creates the trust gap one level up from the platform. The architecture must make rich diagnostic value available to the operator while ensuring the diagnostic value never depends on operator access to the customer's actual data.

**The alternatives that fail this test:**

- **Standard SaaS logging with PII filters.** Filters are policy, not architecture. They run after the data has been captured; they depend on regex coverage; they fail silently when new data shapes appear. Provable from a careful read of the filtering code; not provable from a schema. Inevitably leaks.
- **Operator-side encryption with customer-managed keys.** Doesn't solve the problem — the operator still has the ciphertext and can compel decryption (legal, contractual, or technical pressure).
- **Customer-controlled telemetry opt-out.** Better but degrades the operator's ability to support the customer. Asymmetric incentive: hosts under stress will opt back in for support, then forget to opt out, then leak.

**How Obra implements it.** The audit emitter, the connector framework, and the operator-side ingester all share a strict schema that defines exactly which event shapes can exist on the operator-side surface. The shape is defined in TypeScript as a Zod schema, validated at three points (emit-time on the host, transport-time at the shipper, ingest-time at the operator). Any field that could carry customer business content is structurally absent from the operator-side shape; the field exists only in the host-local audit chain. This is enforced at the type level (not at the runtime-filter level), which means a passport number cannot reach the operator because no operator-side schema field accepts a passport number. The boundary fix work documented elsewhere addresses the few remaining schema fields that could carry payload-shaped data via free-text reason fields or before/after state snapshots; those have been replaced with categorical hashes and structured class enums in [B1-B4 fixes shipped 2026-05-19].

---

## The data path

The four properties above describe what the architecture is. This section describes what *happens to a guest's identity image* end-to-end, step by step. The intent is to be transparent rather than rhetorical: a thoughtful technical reader should be able to verify each step against the code, the network behavior, and the contractual relationships, and arrive at the same picture.

The walkthrough below uses passport verification as the canonical example because it is the regulator-touching action with the highest sensitivity. Other extraction-heavy actions (invoice parsing, tax-document reading) follow the same path.

### Step 1 — Guest device → host's runtime

The guest accesses the secure-upload page on a URL that points at a process running on the host's hardware. The page is served over an authenticated session — the URL itself contains a per-reservation secret. The upload completes when the bytes of the image have transferred from the guest's device to the host's runtime, over an HTTPS connection terminated at the host's machine.

The image bytes are now at rest on the host's hardware. They are not at rest on a platform's server, a vendor's server, or any cloud storage outside the host's machine.

### Step 2 — Host runtime → Anthropic API → host runtime

The host runtime invokes Claude via Anthropic's commercial API. The HTTPS request is authenticated with the host's own Anthropic API key (managed by the operating system's secrets store, never logged, never transmitted off the host's machine other than to Anthropic). The request payload contains the image bytes and the extraction prompt.

The data is in transit between the host's machine and Anthropic's commercial API endpoint for the duration of this call. Anthropic's commercial-API commitments apply: no training on customer API data, Zero Data Retention available for eligible accounts (meaning the request payload is not persisted by Anthropic beyond the response). The response — a structured JSON payload containing the extracted identity fields — returns to the host's runtime.

This is the only step in which the image bytes leave the host's hardware. They reach exactly one third party (Anthropic), transiently, under a commercial contract the host owns directly.

### Step 3 — Host runtime → host-resident storage

The host runtime takes the extracted JSON from Step 2 and writes it into the host's persistent storage alongside the original image. The audit chain entry for the extraction action is computed and appended to the host-resident audit chain. The chain entry includes a hash of the image bytes and a hash of the extracted payload; it does not include the image bytes or the unredacted payload.

### Step 4 — Host runtime → SIBA portal (regulatory submission)

When the host approves the regulatory submission, the connector framework drives the SIBA portal over a browser session (the connector reference implementation uses Playwright). The fields submitted to SIBA are the eight fields SIBA requires; the underlying image bytes are not transmitted to SIBA (the SIBA portal does not accept the image — only the typed fields). The host's runtime is the only intermediary between the extracted JSON and the regulator's portal.

### Step 5 — Host runtime → operator-side telemetry (Obra)

The audit emitter ships redacted events to the operator-side ingester. The events contain event types, counts, hashes of underlying payloads, structured categorical decisions (without their text reasons), and timing of operations. They do not contain the image bytes, the extracted identity fields, the host's approval reason text, or any other element of the special-category data.

The operator-side ingester pins the genesis of each host's audit chain at first sight, validates the redaction schema on every event it receives, and surfaces operator-visible analytics (workflow health, drift rates, failure modes) from the redacted stream. The operator has no path to the unredacted payloads — the host-local chain holds them, and the operator-side schema cannot represent them.

### Summary

| Step | Actor | What touches the image bytes |
|---|---|---|
| 1 | Guest → host runtime | Host hardware (first server to receive) |
| 2 | Host runtime → Anthropic API → host runtime | Anthropic (transiently, under host's own API key) |
| 3 | Host runtime → host-resident storage | Host hardware |
| 4 | Host runtime → SIBA portal | SIBA receives extracted fields only, never the image |
| 5 | Host runtime → operator-side telemetry | Operator never sees the image — only hashes and metadata |

The operator (Obra) does not appear in Steps 1–4 at all. Step 5 is the only step where the operator is involved, and it is structurally constrained to redacted metadata.

### Anticipated upgrade paths

The architecture admits two strengthening paths the design already anticipates. Neither is required at v1; both are reachable from the v1 design without re-architecting.

**Hybrid extraction.** Step 2 above can be split: on-host OCR (open-source — Tesseract for the visual zone, an ICAO 9303 MRZ parser for the machine-readable zone) extracts the field strings from the image; only the extracted text — not the image bytes — is routed to Claude for reasoning, cross-check, and structured output. Under hybrid extraction, **the image bytes never leave the host's hardware at all**; the only data transmitted to Anthropic is the extracted text. This is a stronger privacy posture than the v1 design and is on the v1.5 roadmap.

**Anthropic on customer-controlled compute.** Anthropic's models are available through Amazon Bedrock and Google Cloud Vertex. A customer that requires inference data to stay inside their own cloud tenancy (e.g., a customer with a strict EU data-residency contract, or a customer deploying through a small operator who hosts inside their own AWS/GCP account in Frankfurt or Lisbon) can run the host runtime against a Bedrock or Vertex endpoint instead of the public Anthropic endpoint. The inference call still goes to Claude; the data never leaves the customer's own cloud account. This is the structurally tightest deployment posture and is available today; we expect to use it for customer cohorts where the data-residency contract demands it.

The reason to document both now: a thoughtful technical evaluator will ask whether the architecture has a higher privacy ceiling than the v1 shape consumes. It does. The path from v1 to either of these upgrades is incremental, and the v1 design does not foreclose either.

---

## The connector framework

Skills do not directly execute external actions. They invoke typed actions defined by connectors. The connector framework is the interface between Claude's reasoning and the external systems it acts on.

### Typed action contracts

Every connector declares its actions in a typed manifest. Each action has:

```typescript
type ActionManifest = {
  name: string;                           // unique within the connector
  tier: 'autonomous' | 'gated' | 'forbidden';
  input_schema: ZodSchema;                // validates inputs at invocation time
  output_schema: ZodSchema;               // validates outputs after execution
  requires_extraction_agreement?: boolean; // see Pre-proposal extraction agreement gate
  idempotency_key_fields: string[];       // which inputs combine to form the idempotency key
  drift_signals: DriftSignal[];           // what to watch for that indicates the external system changed
  audit_event_types: string[];            // which events this action can emit
};
```

A workflow that wants to take an action calls the connector's action by name with input payload that conforms to the input schema. The connector framework validates the input against the schema. If validation fails, the action does not execute. If validation passes, the framework checks the action's tier and proceeds accordingly.

### Tier classification

Three tiers govern how an action progresses from proposal to execution:

- **Autonomous.** The action executes directly. Reserved for low-stakes mechanical work: scheduling a cleaning crew based on check-out time, sending a templated thank-you with no operator-side variability, generating a draft for human review. Autonomous actions still emit audit events; the host can retrospectively inspect every autonomous action that ran.
- **Gated.** The action prepares the call payload, then halts and waits for human approval through the operator dashboard. The host sees the full proposed payload, dwells on it for the configured dwell time (default 10 seconds for first-pilot operators), provides a required reason, and either approves or rejects. The approved payload is what executes; no AI intervention between approval and execution. Used for any action that touches the regulator, the guest's money, external systems with consequence, or the host's reputation.
- **Forbidden.** The action cannot execute without explicit per-action authorization that escalates above the normal gated flow. Reserved for actions a thoughtful operator would never want their AI to perform without a human-driven exception process: data deletion, credential rotation, bulk modifications, anything irreversible.

**The tier is declared on the action, not the workflow.** A workflow cannot smuggle a novel action past human review by phrasing it cleverly; the action's tier is fixed by the connector that defines it. New actions are forbidden by default until explicitly classified.

### Drift detection

Connectors that interact with external systems (PMS portals, regulator portals, booking-platform APIs) declare drift signals — the symptoms of an external system having changed. If a connector encounters a drift signal (a selector no longer matches, an API response is unexpected, a portal returns an unknown error shape), the connector halts the action cleanly and emits a `drift_detected` event. No partial execution. No best-effort attempt to recover.

The drift-detection contract is opinionated for a reason: connectors that try to recover from drift by re-reasoning over the changed system end up making unverifiable decisions on the host's behalf. Halting on drift is conservative; the host gets notified, the connector maintainer ships a fix, and the runtime resumes from a known-good state.

### Idempotency

Every action carries an idempotency key composed of fields declared in its manifest. The framework checks the audit chain for a prior execution with the same key before invoking the action. If a prior execution exists, the framework returns the prior result and does not re-execute. This means "I'm not sure if this ran" is always safely answered by re-running the action; double-runs are detected and rejected.

The idempotency rule is what makes the system safe to retry. Without it, transient failures (network blips, temporary portal outages) produce duplicated submissions. With it, the same workflow can be re-invoked safely any number of times.

---

## The three-gate trust pillar

Anything that touches the regulator, the guest's money, or the host's reputation passes through three independent gates before it executes. The three-gate pillar is the architectural answer to "how does the system act in the world without acting unilaterally."

### Gate 1 — Structured reasoning

Claude proposes the action. The proposal is structured: a typed payload conforming to the connector's input schema, plus a natural-language explanation of why this particular action is appropriate for this particular context. The structured form means downstream gates can validate against schema, not against prose; the natural-language explanation lets the human reviewer audit the reasoning, not just the output.

### Gate 2 — Adversarial review

A second Claude invocation reviews the proposal as a red team. The prompt is different: instead of "what should we do for this guest?" the second invocation gets "this is the proposal; what are the strongest reasons it could be wrong?" The second pass is asked to be specifically skeptical: hallucinated values, mismatched recipient, wrong workflow for the context, regulatory misclassification, unverified premises.

The adversarial gate catches a specific class of failure: cases where the first invocation generated a confident-sounding wrong answer. It does not catch cases where both invocations could share the same hallucination (same input, same model, same context) — that class is handled by [the pre-proposal extraction agreement gate](#the-pre-proposal-extraction-agreement-gate) below.

### Gate 3 — Human confirmation

The host sees the proposal, the adversarial review's verdict, the structured payload, and the natural-language reasoning. She has at least the dwell-time minimum (default 10 seconds for first-pilot operators) before the approve button becomes clickable. She enters a required reason — even if it's just "verified" — which is logged with her approval. She either approves the payload or rejects with a reason.

The human gate exists for the failures the other gates cannot catch:
- Wrong values that pass both reasoning gates because the AI is confident
- Edge cases the AI's training does not cover well
- Context the AI does not have (the host knows this guest, the host remembers the prior reservation, the host saw something off in the conversation)
- Final accountability: the host is the legally responsible party; the architecture preserves that responsibility rather than diluting it

### What the gates do NOT guarantee

The gates do not guarantee zero hallucination outcomes. They guarantee that no action with regulatory, financial, or reputational consequence executes without three independent reviews. A hallucination that survives all three reviews can still execute. The architecture's claim is *"hallucinations cannot act unilaterally,"* not *"hallucinations never act."* The distinction is honest and load-bearing.

---

## The pre-proposal extraction agreement gate

A separate gate runs *before* Gate 1 for extraction-heavy actions — actions where Claude has to read unstructured input (a passport image, an invoice PDF, a tax document) and produce structured output (a typed identity payload, a line-item list, a parsed claim). This gate exists because the standard three-gate pillar catches the wrong class of failure on extraction work.

### The failure mode the standard gates miss

If Claude misreads a passport number as `123456789` when the real value is `123456780`, Gate 1 (structured reasoning) generates a proposal with the wrong number. Gate 2 (adversarial review) is reasoning over the *proposal*, not the source image; it does not catch the misread. Gate 3 (human confirmation) sees the proposed payload; the host can catch the misread only if she compares against the source image visible to her in the verification UI. The architecture cannot rely on the human catching every misread — humans rubber-stamp under time pressure, and an 18-character document number is hard to verify against an image without careful side-by-side inspection.

### The agreement-gate pattern

For extraction-heavy actions, the framework runs two structurally independent reads of the source artifact:

```
[source artifact]
    ├── Pass A: Claude reads visual fields
    │             (printed identity data, human-readable typography)
    │
    └── Pass B: Claude reads MRZ (Machine Readable Zone) and parses per ICAO 9303
                  (structured machine-format with built-in check digits)

    │
    ▼
compare(A, B) structured payloads field by field
    │
    ├── agree → proceed to Gate 1 (proposal)
    │
    └── disagree → halt; escalate to human with both reads visible + the diff
```

The critical property: Pass A and Pass B are **structurally different reads** of the same document. Pass A reads the printed visual fields. Pass B reads the MRZ — a standardized OCR-optimized format encoded at the bottom of every modern passport, with check digits that allow each value to self-validate. The MRZ is not just "the same data again"; it is the data in a different format, decoded by a different parser, with internal cross-validation.

Genuine agreement between visual and MRZ is much stronger evidence the extraction is right than two visual reads. Two visual reads of the same image with the same model could agree on a misread because they share the same OCR-interpretation bias. Visual-vs-MRZ disagrees the moment the bias differs, and the MRZ's check digits surface OCR errors before they reach the comparison stage.

### Why this is a separate gate

The agreement-gate is **structurally independent from the three-gate pillar.** It runs before the pillar (so a disagreement halts the action before it ever enters the proposal stage). It catches a specific failure class (extraction-OCR errors) that the pillar's reasoning gates cannot catch. And it generalizes beyond passports to any extraction-heavy action: invoice line items (visual scan vs. structured PDF parse), tax document fields (printed text vs. embedded form data), patient record details (handwritten vs. structured form), legal document targets (PDF text vs. OCR extraction).

### Trade-offs

- **Cost:** 2× tokens on extraction-heavy actions only. Total operational cost increase is bounded: extraction actions are a small minority of total workflow actions in a typical deployment.
- **Latency:** No wall-clock impact if passes run in parallel. The framework parallelizes by default.
- **Shared-bias risk:** Both passes use the same model (Claude). If Claude has a systematic bias on a particular extraction shape (rare passport script, low-quality scan), both passes could agree on the wrong answer. Mitigations: deliberately varied prompts between Pass A and Pass B (different reading strategies); future cross-model verification when multi-model API access becomes available (Pass A on Sonnet, Pass B on Opus); sampling-based spot-checks where a percentage of agreed pairs are reviewed by humans, with results audited for systematic-agreement-but-wrong patterns.
- **Escalation rate on legitimately ambiguous inputs:** smudged passports, low-resolution scans, edge-case document types. This is the *correct* behaviour — the architecture should escalate when uncertain — but the host-facing UX has to make manual verification fast, not punishing.

---

## The audit chain

Every action the runtime takes — autonomous or gated — emits one or more audit events. The events are hash-chained: each event includes the hash of the previous event, forming a tamper-evident sequence. The chain lives on the host's hardware.

### Event shape

```typescript
type AuditEvent = {
  event_id: string;                       // unique, derived from content + timestamp + nonce
  event_type: string;                     // e.g., 'extraction_attempted', 'host_approved', 'submission_acknowledged'
  client_id: string;                      // identifies the client whose runtime emitted this
  emitted_at: ISO-8601;                   // host-clock timestamp
  previous_event_hash: string;            // hash of the prior event in this client's chain
  payload: RedactedPayload;               // event-specific, redacted-by-construction
  payload_sha256: string;                 // hash of the underlying unredacted payload, locally
  signature?: Signature;                  // operator-mixed entropy + client checkpoint (Phase 1a HMAC; Phase 3 asymmetric)
};
```

Note what the shape allows and what it does not:
- `payload` is the redacted form of the event's payload. It contains structured metadata (event type, counts, categorical decisions) but never raw business content.
- `payload_sha256` is the hash of the *unredacted* payload, computed locally on the host. The host keeps the unredacted payload in their local chain alongside the event; the operator only ever sees the hash. The hash gives the host a way to prove, at audit time, that the unredacted payload they hold is what was emitted at the time of the event.
- `previous_event_hash` chains the events together. Re-ordering or removing events breaks the chain in a way detectable by anyone who has the genesis-pinned starting hash.

### Genesis pinning

The first event in each client's chain ("genesis") is pinned by the operator-side ingester the first time it sees that client. The operator stores the genesis hash and the time of first sight. From that point on, any chain segment the operator receives from that client is validated against the pinned genesis: a chain that does not start with the right hash is rejected as "this is not the chain we have been tracking."

This is the architectural answer to "what if the host edits history?" Pre-genesis-pinning, the host could rewrite their chain to remove inconvenient events. Post-genesis-pinning, any rewrite produces a chain that fails the genesis check. The host cannot retroactively edit events without producing a chain the operator (and any auditor) will detect as forked.

### Customer ownership

The chain bytes live in a directory on the host's hardware (the JSONL files are append-only by convention; the runtime never rewrites prior events). The host can:

- Export the chain at any time as a single concatenated file
- Hand the export to a regulator, a tax authority, a data subject under a GDPR Article 15 access request, or any third party they choose
- Verify the chain themselves by running the local validator (which checks hash continuity from genesis to the latest event)

The operator-side ingester holds a *copy* of the redacted events as they ship for operator-side analytics, but the operator's copy is not the source of truth — the host's local copy is. If the operator's copy ever conflicts with the host's, the host's wins.

### Witness layer [PLANNED]

[Phase 2a, queued.] An additional layer of externally-anchored timestamping extends the chain's tamper-evidence beyond the operator's pin. The candidate witness stack is:

- **OpenTimestamps** (Bitcoin-blockchain-anchored timestamps, free, decentralized)
- **Sigstore Rekor** (Sigstore project's transparency log, free, widely deployed)
- **eIDAS Qualified Electronic Timestamps** (regulatory-grade, EU-specific, fee-based)

A witness operation timestamps a hash of the current chain head against one or more of these external witnesses, producing a certificate the host can store alongside the chain. Tampering after a witness operation is detectable by anyone with the certificate; tampering before is detectable only against the operator's pin. The full witness layer lands in Phase 2a.

---

## Action authorization model

The tier classification (autonomous / gated / forbidden) is enforced at multiple layers:

| Layer | What it checks | What it does on violation |
|---|---|---|
| Action manifest | Tier declared per action at connector-definition time | New actions default to forbidden; explicit classification required |
| Workflow runtime | At invocation, the runtime checks the action's tier and routes accordingly | Mismatched tier vs. runtime context → halt + alert |
| Operator dashboard | For gated actions, presents proposal to host with dwell timer + required reason | Cannot click approve until dwell elapses; cannot approve without entering a reason |
| Connector executor | Receives only the post-approval payload for gated actions; runs the action | Schema validation at execution time; payload tampering detectable |
| Audit emitter | Every step emits structured events; chain validates after-the-fact | Missing or out-of-order events surface in the host's own chain inspection + the operator-side ingester |

The model assumes operator-error is the most common failure mode and designs for it. The architecture makes it hard to misclassify an action as autonomous when it should be gated; hard to bypass the dwell timer; hard to forget the required reason; hard to execute a payload that has been modified after approval (the Phase 2 boundary-fix work removes the last remaining `modified` escape hatch on this path).

### What the operator can see

By design, the operator's view of any client is structurally limited:

**The operator CAN see:**
- Event types and counts (Skill 02 ran 47 times this month for client X)
- Categorical decisions (47 extractions agreed; 3 disagreed and escalated)
- Drift signals (the SEF portal selector changed on date Y)
- System health (connector errors, runtime crashes, version drift)
- Audit chain integrity (genesis pinned; chain continuous; current head hash)

**The operator CANNOT see:**
- Guest names, passport numbers, addresses, or any other identifying data
- The host's reasoning notes
- The contents of the host's communications with guests
- The specific business decisions the host made beyond their categorical class
- The bytes of any document the host processed

This division is enforced by schema, not policy. The schema enforces it by the absence of any field where business content could appear, which means a leak would require changing the schema (a code change, subject to review).

---

## What this architecture does NOT do

To be explicit about scope:

- **It does not replace the host.** Every regulator-touching, money-touching, or reputation-touching action goes through human verification. The architecture reduces the host's workload to verification, not absence. *"You check the values. Obra does the work."*
- **It does not become a booking platform.** Obra does not generate guest demand. The architecture lives inside the platform-dependent reality hosts already inhabit (Airbnb, Booking, Vrbo) and makes that life less broken. We are not building a marketplace.
- **It does not become a regulatory clearance service.** The architecture produces the audit evidence regulators want; it does not interpret regulation on behalf of the host or take legal responsibility for the host's compliance. Compliance liability rests with the host (as the law requires).
- **It does not promise zero hallucination.** Hallucinations cannot act *unilaterally* (the three-gate pillar prevents it). Hallucinations can still occur in proposals and can still pass through if the gates fail to catch them. The architecture's commitment is structural protection against unilateral hallucinated action, not against all hallucinated content.
- **It is not designed for high-frequency low-stakes work.** The three-gate pillar adds latency and human cognitive load. For workflows where speed matters more than verification, other tools exist. Obra is for the work where wrong is expensive and right is verifiable.
- **It does not yet provide an external witness layer for the audit chain.** The chain is hash-chained and genesis-pinned; full external witness anchoring (OpenTimestamps + Rekor + eIDAS-TSA) is Phase 2a, not Phase 1a.

---

## References

- **[The trust gap](the-trust-gap.md)** — the narrative essay that motivates this architecture
- **[Skills](../skills/)** — the operational shape; Skill 02 (passport extraction with agreement gate) and Skill 03 (SEF submission) demonstrate the patterns documented here
- **[Connectors](../connectors/)** — reference implementations of typed-action contracts; `sef-portal/` is first to land
- **[Examples](../examples/)** — end-to-end worked transcripts showing the skills compose [arriving]
- **`THREAT-MODEL.md`** [PLANNED] — the adversaries this architecture assumes and what it defends against natively
- **`COMPLIANCE-NOTES.md`** [PLANNED] — fuller legal-citation treatment of the regulatory anchors referenced in the essay
- **`DRIFT-PLAYBOOK.md`** [PLANNED] — operational response when external systems change
- **[Anthropic Responsible Scaling Policy](https://www.anthropic.com/news/responsible-scaling-policy)** — Anthropic's framework for deploying frontier models safely; the architecture in this document is an instance of the deployment posture Anthropic articulates

---

*This document is maintained alongside the codebase. Updates that change the architecture's commitments will be called out in a CHANGELOG when one lands. Pull requests, corrections, and questions welcome at `team@get-obra.com` or via issues on this repo.*

*Last updated 19 May 2026.*
