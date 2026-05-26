# The unenforced law

*Why nobody is paying to fix Europe's short-term rental compliance gap, and the architecture we are building before the market knows it needs it.*

> The second of two essays on the architecture of EU short-term rental compliance. The first essay, *[The trust gap](https://get-obra.com/the-trust-gap)*, described how the verification step silently breaks at the channel layer between platforms, hosts, and guests. This essay is about the question that essay leaves unanswered: if the gap is real, why is nobody paying to fix it?

---

At a resort somewhere on the Portuguese coast last summer, the duty receptionist types a passport number into the compliance portal. She has not seen the document. The number came from a guest profile a colleague typed by hand from a photograph nobody can locate. Sometimes she types numbers no one handed her, because the booking came in late and the system needs something. The portal accepts whatever she submits. The property management system the resort pays a real annual fee for files the registration on time, every time. The green box on its dashboard says compliant. Nobody at the resort believes the green box. Nobody at the regulator has ever called. Everyone in this arrangement has been here for years.

This resort is not a small operator. By the system's audit trail, every reservation has been compliant on every filing for years. By the law's verification clause, almost none of those filings are compliant at all.

This essay is about how a market arrives at this state and stays there. It is about the mechanism that makes the gap invisible to the people who are most exposed by it, the economics that prevent anyone from being paid to close it, and the architecture that would actually close it if anyone built it. And it is about why we are building that architecture before the market knows it needs to.

---

## What the market is paying for

Across Portugal, vacation rental hosts and resorts pay for software that promises compliance with the country's guest registration law. The market divides cleanly into three layers. There are SIBA-specialist tools that connect directly to the authority's API and submit the bulletin on the host's behalf. There are property management systems that integrate one of those specialists into their own onboarding. There are adjacent products, smart locks and channel managers, whose marketing touches compliance but who do not themselves file.

The first layer is what most of this essay is about, because filing is the part of the pipeline that works. The bulletin reaches the regulator. The deadline is met. The dashboard's green box is honestly green for the step it represents. The vendors in this layer are well-built products at consumer-friendly price points (Chekin from around four euros per property per month, ChargeAutomation around five dollars plus a platform fee, Sincro from free up to four euros, SIBA GO on a credit-pack model, others in the same band). The market has priced filing as a routine commodity, and the vendors deliver on the price.

What varies across the layer, and what the marketing makes legible to anyone who reads it, is what sits between the moment a guest opens the link and the moment the bulletin gets submitted. Chekin's identity-verification page is the most precise the category gets: OCR plus machine-readable-zone checksum, with the two compared against each other to catch tampered documents; a biometric selfie match against the photo on the document; active liveness prompts; deepfake and mask detection; with the explicit final clause, *"From document capture to verified guest — no manual review needed."* This is a real, full-stack verification offering, and it is the strongest the market has to show. Other specialists pick parts of the stack: CheckinScan claims the MRZ scan but no biometric, SEFScan claims OCR with retry prompts but no MRZ check, ChargeAutomation claims biometric identity verification but is silent on the rest, and Avantio markets biometric verification through its Chekin integration. And then there is a longer list (Sincro, SIBA GO, Homeit's SIBA workflow, Your.Rentals' compliance module, Ynnov, Smoobu's native check-in, Lodgify's native registration module, and Hostaway's native check-in form) where the public product pages we have reviewed describe data collection and submission but no verification step.

Hostaway's [own support documentation](https://support.hostaway.com/hc/en-us/articles/17027928676123-Listings-Online-Check-in-Form) puts this in the most concise terms anyone in the category will commit to in writing: *"Hostaway does not check the ID of the Guest to make sure that it is not a fake."* A major global property management system, openly stating in its own documentation that the guest-portal it ships does not perform identity verification.

So the question is not whether verification exists in this market. It exists. The question is what the market is paying for. The verification tiers, where they exist, sit at the top of the pricing ladder. Chekin's full identity-verification stack is a Premium or Enterprise add-on, above the Basic tier most operators buy. Most operators do not buy the verification tier, because the regulator does not ask for the verification tier. The regulator asks for the filing. The filing is what the cheaper tier covers. The deadline is met, the box is green, and the host has the receipt for the filing in their inbox.

This is the load-bearing pattern. The market is not deprived of verification options. The market chooses, by what it buys, to pay for filing alone. The verification offering exists for the operators who decide they need it. The operators do not yet decide they need it, because nobody is asking them whether they did. The market is paying for a different product than the law requires, knowingly, and the vendors are building, knowingly, the product the market is paying for.

## How everyone in the chain plays a part

A market with this shape is not held together by anyone's bad faith. It is held together by the rational behaviour of everyone in it.

The host is rational. Most of the time, the guest refuses to send a passport image through the booking platform's chat, for the reasons described in the first essay. The host accepts what she can get, types the numbers into the portal, and submits. She knows this is not what the law requires. She also knows that nobody she knows has ever been audited, and that the booking platform's complaint mechanism punishes her for verifying in person more reliably than the regulator punishes her for not verifying at all. She is responding to the incentives that exist.

The vendor is rational. Building a verification product when the market is paying for a filing product is not a strategy a vendor can sustain at the base tier. The vendors that do offer verification offer it as an add-on above the filing tier, because that is where the willingness to pay sits. The vendor who tries to sell verification as the default is undercut by the vendor who sells filing as the default. The market has chosen, by what it buys, that filing is the product. The vendor builds what the market buys, and tiers the verification above it for the operators who decide they need it.

The platform is rational, in the way described in the first essay. The platform protects in-platform communications and penalizes the side channels through which a host might verify a document. The platform's incentives are unchanged by the host's regulatory obligation.

The regulator is rational. Auditing a population of tens of thousands of small operators across a fragmented sector, with limited tooling and limited budget, is structurally unattractive. Enforcement actions concentrate where they are economically rational, on the largest operators with the most visible violations. The long tail is mathematically invisible.

The guest is rational. They have never been told why the question is being asked. They have heard stories about identity theft. They are not wrong to be cautious about a passport image sent through a booking platform's chat to a stranger they have not met.

The result of all this rational behaviour is a compliance regime that runs on paperwork the law would not recognize as compliance. The phrase for this in security circles is *theater*, which is not quite the right word here, because nobody at the resort is performing. They are filling in the box because the box is what the system asks for, and the system is what their compliance is. The performance is the system itself, in front of an audience that has not yet arrived.

## The address that never moves

There is one more layer that makes the gap invisible, and it is the one that surprised us most when a Portuguese operator first articulated it to us in their own words.

If you asked a host whose name is on the regulatory complaint the day their property management system leaks fifty thousand passports, four out of five would name the system. They would be wrong. Under EU data-protection law, the host is in almost all cases the data controller. The system is, in the standard contractual posture, a data processor. The contract between them shifts money on certain kinds of failure. It does not shift the regulatory address. The host has been carrying liability for the system's behaviour for as long as they have used the system, and they cannot see it, because nobody in their world has ever had to look.

This is the load-bearing mechanism behind compliance theater. The hosts have delegated, in their own minds, the compliance function to the system. The legal regime has not delegated anything. The regime regards the host as the party responsible for the guest's data, the verification of the guest's identity, the lawful basis of the processing, and the eventual deletion of the document. The system the host uses is, in the eyes of the regulator, a tool the host has chosen, a tool whose failures are also the host's failures.

We have not seen this stated, in this form, in the marketing materials of any vendor in the category. We have not seen it stated by the platforms. We have not seen it stated in the regulator's published guidance to hosts. We have seen it understood, after we explained it, by every host we have spoken to. The reaction has been the same every time: a long pause, and then a question about what their contract with the vendor actually says. The contracts say what contracts usually say. They contain indemnities and limitations of liability that have not been tested in a Portuguese court, because no Portuguese court has yet been asked.

The result is not that the law is wrong. The result is that there is a structural blind spot of liability sitting on every operator in this category, and the blind spot is the precise reason the operator does not feel compelled to buy verification. They do not believe they need it, because they do not believe the address is theirs.

> *A note on rigor: this is a non-lawyer's summary of the framework in Articles 4(7), 4(8), and 28 of the GDPR. The specific allocation between a host and a property management system depends on the contract, the actual processing pattern, and the relevant supervisory authority's interpretation. There is real caselaw on joint controllership (Fashion ID, Wirtschaftsakademie) where a tool determines purposes or means in ways that make it a co-controller. The general point, that a host using a vendor remains a controller in the eyes of the regime, is not contested. We invite correction from qualified counsel.*

## Why the market does not pay

There is a name in market design for the kind of market this essay is describing. It is a *latent regulated market with no trigger event*. The legal obligation exists. The downside exposure is real. The felt urgency is zero. Everyone is non-compliant. Nobody is punished. Nobody buys.

GDPR looked like this from 2016 to about 2019. Vendors building serious GDPR-compliance infrastructure in 2017 had a thin year. The first large fines crystallized the market over the following twelve months, and the integrators who had been ready inherited it. PCI-DSS looked like this in its early years, before the card networks began enforcing the standard at scale. Accessibility compliance in the United States looked like this until the ADA lawsuit wave reached the small business segment. The pattern is always the same shape: real law, dormant enforcement, no market, until one event flips it. After the flip, the market materializes in twelve to eighteen months.

What is the trigger going to be here? The honest answer is that we do not know. The structural force is the EU's 2024 regulation on data collection and sharing relating to short-term accommodation rental services, which phases in over the coming years and obliges the booking platforms to surface host activity data to national authorities. That visibility shift is exactly the change that makes enforcement economically rational for regulators for the first time. The platforms light up the hosts. The regulators audit the lit-up ones. The market wakes up.

The specific spark could be a fine. It could be a leak. It could be a media story about one resort or one apartment block whose registration records reach a journalist with the time to ask whether the documents matched. We are not in the business of predicting which spark or when. We are in the business of recognizing the shape and being ready.

So the question for someone building infrastructure into a market with this shape is the question every vendor before us has had to answer. Do you wait for the trigger event and try to scale into it when it arrives, or do you build now and sell something the market does want today, with the unsold infrastructure as a quiet property that becomes load-bearing on the morning after the trigger?

We have made the second bet. The architecture sits inside a product the market is paying for today for an entirely different reason.

## What real verification would look like

If a regulator did call, if the contract with the data processor was tested in court, if the address turned out to be the host's, what would actually defend the host?

The honest answer is that there are two distinct questions the regulator might ask, and the answers come from different parts of the architecture.

The first question is whether the document was real. This is where the strongest specialists in the category, Chekin in particular, already do serious work. Machine-readable-zone checksums catch a class of tampered documents. Biometric selfie matching catches identity substitution. Liveness detection catches a recorded video or a high-resolution print. For the operators who buy the verification tier, this work really happens. We are not going to pretend otherwise. The architectural property we hold against this is narrower and complementary: two structurally independent reads of the document, each from a separate vision pass with different prompts and context, that must agree on every field before any field gets filed. If the two reads disagree, the system halts and surfaces both reads to the host. The MRZ catches the cryptographic class of tampering. The two-read agreement gate catches the silent-extraction-error class, where a single read produces a confidently wrong field that a single-read system would file. Both layers matter. We build the second; the strongest specialists build the first; in a category that took the law seriously, both would be present in the same pipeline, and the host would have a record of both.

The second question, and the harder one, is whether the host verified the document. This is the question Chekin's marketing answers with *"no manual review needed."* It is precisely the question the law asks of the host. The data controller's defense, in front of a regulator, is not that an automated system on the vendor's cloud passed the document. The defense is that the host saw the document, saw the data extracted from it, compared one against the other, and pressed submit. Without that chain, an automated verification on the vendor's cloud, with no host-visible review step, is an act the vendor performed, not an act the host performed. The contract between them may shift money on a false positive. It does not move the regulatory address.

The architecture we are building is shaped by the second question more than the first. The host approves the action, and the thing the host approves is the eleven fields the document produced, displayed against the document image, on a board the host reads. The approval is bound cryptographically to the action and to the rendered bytes the host saw. The audit trail records what the host saw at the moment of decision, not what someone reconstructed afterwards. The submission to the regulator is a button the host presses, not a callback an automation fires. This is the difference between a system that automates work and a system that automates a regulated action. The line we will not cross is the line where the host's authorization stops being the host's.

The data is then deleted, and the deletion is recorded on a tamper-evident chain the host owns. Not on a vendor's server. Not on the platform. The host will be able to produce a deletion certificate for any document the system has processed, without asking permission from the vendor, the platform, or the regulator. Provable deletion is the answer to the most common question a guest asks: what happens to my passport. It is also the answer to the most common question a future regulator will ask: prove that the data minimization principle was honoured for this guest, in this filing, on this date.

The reasoning core that does all of this does one more thing the specialists in the category, for the most part, do not. It drafts the request to the guest in the guest's own language, in the host's voice, explaining plainly why the law requires the document and what becomes of it afterwards. It reads the guest's replies. When the guest pushes back, it is designed to read the specific worry behind the pushback and draft a response to that worry, for the host to send. The category's standard is a generic check-in link that lands in a guest's email in stilted English. The architecture we are building is meant to read the conversation, not script it. How much that lifts acceptance is a question the first pilots are about to answer.

None of these properties are theoretical. The two-read agreement gate, the reasoning-bound approval, the irreversible clicks reserved for the host, and the multilingual guest comms are in production today. The audit trail is also in production, with the property that the operator sees event types, counts, and hashes but never the guest's name, document number, or any element of special-category data. The deletion certificate user experience is in build on top of the same audit chain. Each property is in the system being installed on the first host's machine in Portugal as this essay is being written.

## What we are doing about it

We are building this architecture and we are not selling it.

The market is not paying for verified compliance, because the market does not believe verified compliance is what it needs. We have no quarrel with that belief. It is empirically correct given the enforcement state of the regime today. What the market is paying for is the daily work the host is doing herself or paying a co-host to do. A co-host in Portugal costs somewhere between fifteen and twenty-five percent of revenue. A bookkeeper costs more. The labour the host is currently doing or paying for is the real anchor against which a verified-compliance product gets priced. The point tools at four to six euros per property are below the floor of that price. The labour is the ceiling. The gap between them is the room.

We are building inside that room. The host installing the system today is doing it because she is tired of typing the same eleven fields into the same portal at midnight after a guest checks in late, in a language the portal will only accept one specific way. The system writes the polite message to the guest in their language. The system extracts the document. The system fills the eleven fields. The host reviews them on a board, presses Save, and goes to bed. That is the value she feels. The architecture sits behind it as a quiet property of the way it was built.

A sharp reader could object that verification is already on the menu, that operators can pay for it today, and that the market's choice not to pay is the market's choice. The objection is correct, and it does not change the bet. We are not selling verification as a tier above filing. We are selling the daily work the host is doing at midnight, at a price anchored to the labour she would otherwise pay for. The architecture sits inside that product as a property of the way it was built. The operator does not have to choose to pay extra for verification. They get it because they bought the labour relief. On the morning enforcement comes, they are the ones with the architecture they did not consciously buy.

The morning the law gets enforced, the value she feels stays the same. The architecture stops being a quiet property and becomes the only thing that matters. The hosts who already have it wake up in a different position than the hosts who do not. They are not the only ones who will solve the problem then. They are the ones who will not have to solve it.

We do not know which morning that is. We know the morning comes for every regime that has this shape, because every regime that has had this shape has had it before. The architecture takes longer to build than the market takes to wake up. By the time the market wakes up, the architecture is what is on the shelf.

We are writing this in public because the analysis seems more useful as reading than as a slide deck for ten meetings. The category of operator this essay is about is large, distributed, and largely silent. The next operator who reads this and recognizes the resort in the opening paragraph will know things about their own compliance arrangements they did not know an hour ago. Some of those operators will decide they would like the architecture to be theirs by the time it matters. We would like to hear from them.

## A note on what this essay is not

This is not a product pitch. The product page is at [get-obra.com](https://get-obra.com).

It is not a competitor hit piece. The vendors named in this essay are good products doing what their customers are paying them to do. The market they serve has decided what it is buying. Verification is on the menu at the top of the price ladder, but it is not what the market is buying. We do not think any of the named companies is failing to deliver the product they have promised. We think the product the market has been asking for is a different product from the one the law actually requires.

It is not a prediction. We do not know when the trigger event arrives. We know the shape, and we know the architecture, and we know that the hosts who pay us for the work the work removes from their day are the same hosts who will have the architecture on the morning enforcement comes.

If you see something in this essay we have got wrong, or if the opening rings true to your own situation, an email to **team@get-obra.com** finds us.

---

*Obra is built on Claude. The Portuguese host who is the subject of this essay's closing paragraphs is real, has reviewed this account, and consents to her workflow being described in anonymized form. The resort in the opening section is composited and anonymized from the lived experience of staff who shared the workflow with us under condition of source protection. No real host name, real guest name, real passport number, real booking-platform internal data, real property location, or any other identifying detail appears in this essay or in our public repositories. Every scenario described is illustrative.*

*Maintained by the Obra team. Released under CC BY 4.0. Last updated 26 May 2026.*
