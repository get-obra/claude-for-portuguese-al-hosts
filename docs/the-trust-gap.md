# The trust gap

*How Europe's short-term rental compliance silently breaks at the channel layer, and what kind of infrastructure is required to fix it.*

> **Canonical version:** [get-obra.com/the-trust-gap](https://get-obra.com/the-trust-gap). This markdown copy is mirrored in the public vertical pack repo so engineers and contributors discovering the work via GitHub can read the same essay alongside the architecture documentation. The two versions stay in sync.

---

Somewhere on the Portuguese coast, a vacation rental host opens her email at six in the morning. A booking has come in overnight. The guest is from Lyon and arrives in three days. Before he gets here, she has six things to do. The one that will go wrong is the passport.

Portuguese law requires her to register the guest's identity with the border authority within twenty-four hours of arrival, verified against a passport or equivalent document. The law has been on the books for more than a decade. The penalty for missing a submission is substantial. Most hosts she knows have never been audited.

A few days before the guest arrives, she sends a polite message through the booking platform's chat. *Hello, we are excited to have you as our guest. We need to ask you for some information to provide to the police. We are legally required to. Could you please send a photo of your passport so I can register it.* Her English is uncertain. The message arrives in the guest's app translated by an algorithm, where it reads colder than she intended.

Eight times out of ten, the guest refuses.

The reasons vary. He does not want to send a passport image through a booking platform's chat. He does not know what *SEF* is or whether it is real. He has heard stories about identity theft and is wary on principle. He offers to send the numbers instead, typed into the chat. She accepts what she can get. She types the numbers into the police portal. She does not verify them against a document she has not seen. She submits.

She knows this is not what the law requires. She also knows that nobody she knows has ever been audited. Everyone does it this way. The compliance regime, as it actually exists in 2026, is paperwork that runs on trust she cannot independently verify.

This essay is about why that is the case, and what kind of infrastructure is required to make it stop being the case.

---

## What the law requires

Portugal's *Decreto-Lei n.º 128/2014*, with its subsequent amendments, obliges anyone operating short-term rental accommodation to register the identity of every non-EU/EEA guest with the relevant border authority within a short window of their arrival. The mechanism is a web portal where the host enters the guest's name, document number, date of birth, nationality, country of issue, and the dates of the stay. The portal stores nothing about the host's verification process. It accepts whatever the host types.

Equivalent obligations exist across the EU. Spain has *Registro Único de Huéspedes*. France has the *fiche individuelle de police*. Italy has the *Alloggiati Web* portal. The Netherlands has the *nachtregister*. Greece has the *Special Register*. The details differ. The shape is the same: short-term rental hosts must register every guest, the registration is independently of the booking platform, and the verification step exists in the law but is not architecturally enforced.

The Schengen Acquis underpins this. The regime predates short-term rental at scale. It was designed for hotels, where a uniformed receptionist takes the passport, swipes it through a scanner, and hands it back. The legal framework assumed a face-to-face verification step. Short-term rental did not invent the absence of that step. Short-term rental discovered it by stretching the regime into a deployment shape it was never designed for.

> *A note on rigor: the above is a non-lawyer's summary. The operative text for Portugal is Decreto-Lei n.º 128/2014 of 29 August 2014, as amended (consolidated text available at diariodarepublica.pt); the SEF reporting obligation is implemented through Decreto-Regulamentar n.º 14/2019. EU equivalents are named with their common labels rather than full citations; specific instruments vary by jurisdiction and we invite correction from qualified counsel. A fuller compliance-notes document is forthcoming in this repo at `docs/COMPLIANCE-NOTES.md`.*

---

## What the hosts actually do

The 80% refusal rate is not a survey. There is no public statistic for it, and there is unlikely ever to be one. No host can publicly admit how often the verification step fails without exposing themselves to regulatory liability they have not earned and platform retaliation they cannot afford. The pattern lives in conversations hosts have with each other in private, in the shrugs at local meetups, in the messages exchanged with accountants who quietly handle their books. We know about it because the hosts who shared this with us trust that what they said would not put them at risk. The figure is an order-of-magnitude claim grounded in those private exchanges. The order of magnitude is what matters. The precise number is unknowable until either the platforms or the regulators choose to make it knowable.

Of the small fraction who do not refuse outright, most send the identity data as typed text in the platform's chat: a name, a document number, a birthday. Some send a photo. Hosts who insist on the photo risk the guest cancelling the reservation. Hosts who do not insist on the photo accept the text and proceed.

The hosts know that what they are doing is not the verification step the law assumes. They do it anyway. The platforms' complaint mechanisms create a chilling effect: a guest who feels too scrutinized can leave a bad review, file a complaint, request a refund, or all three. The host's livelihood lives at the mercy of those mechanisms. The regulator's audit power is theoretical. The platform's punishment power is immediate.

This is not negligence. It is a rational response to a system whose incentives are misaligned. The host has been left to navigate, alone, a contradiction that nobody set out to create but that everyone in the system has reason to maintain.

---

## How the platforms amplify the gap

The booking platforms do not solve this problem. They also do not stand neutrally outside it. They actively make it harder to solve, on three fronts.

**They penalize off-platform communications.** Hosts who try to move guests to WhatsApp, where guests are more likely to share an image, face downranking and listing penalties. The platforms enforce in-platform communication as a business priority. The compliance workaround that the host community knows would help is the workaround the platforms forbid.

**They penalize external links in chat.** Hosts who try to share a link to a host-controlled upload page, where a guest might trust the architecture more than they trust the platform's chat, have those links classified as off-platform reservation diversion. The link gets flagged. The host gets warned. Repeat infractions risk delisting.

**Their complaint mechanism punishes hosts who try to verify in person.** A host who meets a guest at the door and asks to see a passport before handing over the keys creates a moment of friction. The guest can complain to the platform, request a refund, or write a bad review. Hosts who depend on platform ranking, which is most of them, cannot afford the friction. The verification step that the law assumes happens at the door is the verification step the platform's economic incentives discourage.

The platforms are caught in a contradiction. Their in-platform enforcement protects their core business. Their growing EU regulatory exposure pushes them toward stronger guest-identity verification at the platform level. Their hosts' regulatory obligations sit between the two. They are aware of this contradiction. They have not yet solved it themselves, because solving it would require either weakening platform-comms lock-in or becoming a regulated data processor for special-category identity data. Both are positions they have actively avoided for years.

---

## Why no centralized player can fix this

The intuitive answer is *the platforms should just build this*. The platforms cannot build this, for a reason that is more structural than political.

The guests' refusal to send passport images is not arbitrary. It is a response to the *centralization* of the channel they are being asked to send it through. The guest does not trust the booking platform with their special-category identity data. A hosted SaaS vendor solving the problem on the platform's behalf would inherit exactly the same trust position, because the guest would correctly perceive the SaaS vendor as another centralized party that the data has to flow through. The data has not become more private; it has just changed which company holds it.

The problem is not a missing feature. The problem is that the architecture of every existing party, platform, vendor, hosted-SaaS, is incompatible with the trust shape the guest is asking for. Centralization is the substrate the problem grows on. Any solution that runs on the same substrate cannot grow a different fruit.

The architecture that can close this gap has four properties that are not optional:

1. **The guest's image is uploaded to a runtime the host operates**, not to platform storage, not to vendor storage. No third-party data processor sits between the guest and the host's runtime. The guest can be told this and the architecture demonstrates it.
2. **Inference runs under the host's own API key**, not the operator's. Each inference call sends a single image to Anthropic's commercial endpoint for a single response and nothing more; Anthropic's commercial commitments apply (no training on customer API data; Zero Data Retention available for eligible accounts). The reasoning core never sees a centralized database of identity images. The host's runtime never has more than one guest's image at a time, and discards it after the regulatory submission is acknowledged.
3. **The audit trail is hash-chained and owned by the host**. Not stored on a vendor's server. Not stored on the platform. The host can produce it on demand for a regulator, a tax authority, or a curious data subject, without anyone's permission.
4. **The telemetry that flows back to whoever is operating the system is redacted by construction**. The operator sees event types, counts, and hashes. They never see guest names, passport numbers, addresses, or any element of the special-category data. The architecture proves this rather than promises it.

This is not a list of marketing claims. It is the actual minimum specification for an architecture the guest will trust enough to engage with. Anything missing one of these properties leaves the guest with the same objection that drove them to refuse the platform's chat in the first place. A longer treatment of why each property is necessary, with case-by-case analysis of why weaker alternatives (centralized SaaS with strong encryption, EU data residency, customer-managed keys) still fail the trust test, is forthcoming in this repo's `docs/ARCHITECTURE.md`.

The corollary is that the architecture the gap requires is one that **the parties who have the distribution to deploy at scale cannot build**, and the parties who can build it (small, independent, structurally local-first) lack the distribution. This is the textbook shape of a market gap that compounds the longer it remains unaddressed.

The architecture also admits two strengthening paths the design already anticipates. Hybrid extraction, on-host OCR and machine-readable-zone parsing, with only extracted text fields routed to Claude, keeps the image bytes from leaving the host's hardware at all. Inference on customer-controlled cloud compute via Amazon Bedrock or Google Vertex keeps the data inside the customer's own EU cloud tenancy. Neither is required for the first pilot; both are reachable from the v1 design without re-architecting. The architecture's privacy ceiling is higher than the v1 shape consumes.

---

## What this implies more broadly

The Portuguese short-term rental case is a local instance of a continent-wide pattern. Every EU jurisdiction has analogous guest-registration laws. Every one has the same shape of trust gap at the channel between platforms, hosts, and guests. The estimated population of affected operators is not in the thousands. It is millions, across hospitality, vacation rental, B&Bs, hostels, and adjacent regulated lodging categories.

The implications extend further. Wherever an EU regulated SMB is obligated to collect, verify, and report special-category data on behalf of a regulator, and wherever the channel that data has to travel through is mediated by a platform whose architectural assumptions are incompatible with the trust the data requires, the same gap exists. Vacation rentals is the most visible case. It is not the only case.

This is also a story about *how AI can be deployed responsibly in a regulated production environment*. The reasoning core that does the actual document-reading, the multilingual draft, the audit-trail generation, and the decision support is a frontier large language model. Done badly, an AI like this becomes a centralized vendor consuming special-category data at scale and creating its own version of the trust gap. Done well, it runs in a deployment shape where the operator is structurally outside the inference data path, the customer controls the AI relationship directly, and human review gates every regulator-touching action. The same model, in a different deployment shape, produces opposite outcomes for guest trust and regulatory compliance. Both halves matter: the capability the model brings, and the architectural posture that lets the capability reach the user safely. Neither alone is sufficient.

---

## What we are building

We are a small group at the start of this work. The vacation rental wedge is where we are beginning, because we work directly with a Portuguese host whose business is exactly the shape this essay describes, and because the architecture we believe is required for the broader EU regulated SMB market can be proved out here at small scale first.

The platform is called **Obra**. It is built on **Claude**. It runs on the host's infrastructure, invoking Claude under the host's own Anthropic API key. It runs guest-identity collection through a structurally trustable channel the host controls. It produces a hash-chained audit trail the host owns. It pairs every regulator-touching action with a human verification step the host signs off on, before the action runs. It is being released as a managed service, with the vertical-specific skills released open-source under Apache 2.0.

We are not trying to become a booking platform. We are not trying to compete for guests. We are trying to make the compliance step that the law already requires actually work, inside the channel reality hosts already live in.

You are reading this inside the open-source vertical reference pack. The architecture is documented; two of the skill files are in. The rest land over the coming weeks. The first pilot launches with the Portuguese host who is the subject of much of this essay, this summer.

The work is also a hypothesis about what responsible AI deployment in regulated EU sectors looks like at all. We believe the architectural properties that close the trust gap for short-term rental are the same properties that close the analogous trust gaps for the EU regulated SMB universe more broadly: accountants under data-minimization obligations, property managers under tenant privacy law, small healthcare providers under HIPAA-equivalent regimes, legal practices under privilege constraints. The wedge is one vertical. The bet is the architecture.

---

## A note on what this essay is not

This is not a product pitch. The product page is at [get-obra.com](https://get-obra.com). This essay exists because the problem it describes is real and load-bearing for the deployment of AI in regulated European sectors, and because we believe the analysis is more useful as public reading than as a slide deck for ten meetings.

It is also not a complaint about the booking platforms. The platforms are large companies operating in a complex regulatory environment with constraints we do not face. We believe a partnership-shaped path exists where their compliance posture improves and their hosts' lives get easier, with us providing the infrastructure layer they have correctly chosen not to build internally. We expect that conversation to happen on a future date that depends on work we have not yet done.

It is, mainly, the kind of essay we would have wanted to read six months ago when we started building. If it is useful to you, or if you see something we have got wrong, an email to **team@get-obra.com** finds us. Pull requests welcome on this repo.

---

*Obra is built on Claude. The vacation rental host whose work appears throughout this essay is real, has reviewed this account, and consents to her workflow being described in anonymized form. The 80% figure is an order-of-magnitude estimate grounded in her direct experience and the broader network of Portuguese AL operators we have spoken with. No real host name, real guest name, real passport number, real booking platforms' internal data, real property location, or any other identifying detail appears in this essay or in our public repositories. Every scenario described is illustrative.*

*Maintained by the Obra team. Released under CC BY 4.0. Last updated 19 May 2026.*
