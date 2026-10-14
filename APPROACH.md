# Larkspur Health — Patient Voice & Chat Agent Design

## Problem Summary

Source: [Brief.md](./Brief.md). Larkspur Health, a seed-stage startup, is building voice and chat agents that talk to patients on behalf of clinics; its first customer is **Saguaro Family Medicine**.

**The problem.** Saguaro's phone lines are drowning — about **2,800 calls/day**, with Monday-morning hold times over **20 minutes**. After 5pm everything goes to an answering service that pages the on-call doctor for things a nurse could handle and misses things it shouldn't. Larkspur wants **one agent** answering the **phone, the text line and the patient-portal chat** that can book and move appointments, take refill requests, answer "is my MRI approved yet" and "do you take my plan," and make the **post-discharge follow-up calls owed within two business days**. Maya also wants it to tell patients whether they need to come in.

**Key constraints.**
- **Scale/geography:** 14 primary care clinics — 8 in Arizona (Phoenix, Tucson), 4 in Nevada (Las Vegas), 2 in New Mexico (Albuquerque); ~75,000 active patients; >40% on Medicare or Medicare Advantage.
- **Language:** ~30% of patients list Spanish as preferred; some clinics have bilingual front-desk staff, most do not — bilingual support is core, not optional.
- **Compliance:** the privacy officer's single question is **"where does the audio go?"** — audio/PHI handling and BAAs are the gating concern.
- **Systems:** the front desk **lives in Epic**, so the agent must integrate with it as the system of record.
- **Timeline:** go live at **one clinic in eight weeks**.

**Five representative contacts (A–E):** A) phone, Las Vegas — lisinopril refill + Humana insurance change; B) portal chat, 11:40pm — post-heart-failure-discharge worsening shortness of breath and ankle swelling (potential emergency); C) phone, Tucson (Spanish) — prior-auth status for a spouse's MRI (proxy access); D) text from an unknown number — requests A1C results by self-declared DOB (identity/PHI risk); E) phone from a 612 area code — seasonal visitor in Mesa (85206), new patient, wants a same-week appointment for a persistent cough. Working out what the agent should and should not do autonomously on each is part of the job.

## Section A - Open Questions (Pre-Design)

Twelve unanswered questions, ranked by impact on architecture, implementation and the 8-week scope. Priorities weighted: patient security/HIPAA, authentication, multilingual support, reliability. This is the pre-design decision log; the working assumptions, design, scope and next step follow in Sections B–E. (The number one question — audio handling and BAAs — leads because it gates every PHI path; authentication ranks third because it gates PHI disclosure specifically, while integration ranks second because it gates feasibility across the whole pilot.)

1. **Where exactly does the audio go, and under which BAAs?** Does every hop touching raw audio/transcripts (carrier, ASR, TTS, translation, LLM, the Cadra proxy) sit under a signed BAA? Is audio recorded/retained, for how long, in what region, and who can retrieve it? Is redaction required before any third-party call? — *If the model proxy is not BAA-covered, PHI cannot be sent to it (forcing a covered endpoint or de-identified prompts, which breaks name/DOB verification). "No recording" forces streaming ASR → immediate discard, changing storage, replay, QA and metrics. A "no" on any hop can force self-hosted ASR/TTS — a fundamentally different 8-week plan.*

2. **What is the actual Epic integration surface?** Which interfaces are available to a third party — Epic on FHIR (`Patient`, `Appointment`, `Slot`, `Schedule`, `MedicationRequest`, `DocumentReference`, `RelatedPerson`), EpicCare Link, HL7v2, MyChart APIs, or read-replica/RPA into Hyperspace? Is write-back permitted? — *Decides whether the agent is a system of action or a message-taker. UI-only access forces brittle RPA and collapses scope to "capture intent, queue for humans." Also gates reads of prior-auth status, A1C results (case D), caregiver relationships (case C) and eligibility.*

3. **How do we authenticate callers and prove identity before any PHI?** MyChart fallback? Is KBA (DOB + last-4 SSN/ZIP) and SMS OTP permitted, at what NIST 800-63 IAL? Is caller-ID a hint only? How are authorized representatives verified (case C), and are proxies modeled in Epic (`RelatedPerson`)? — *Identity proofing is the gate on PHI disclosure. Drives a dedicated auth service (OTP, rate limiting, lockout), the challenge flow (case D would release A1C on DOB alone — insufficient), and proxy access control.*

4. **What is the clinical scope of "tell patients whether they need to come in"?** Formal triage (Schmitt-Thompson protocols with documented dispositions) or non-clinical scheduling guidance? Who owns liability, and is a clinician on the loop? May the agent give advice, or only dispositions? — *Case B (worsening heart failure at 11:40pm) is the highest-liability contact. True triage means building/licensing and validating a protocol engine plus nurse routing — a large clinically-reviewed workstream. Routing-only hard-limits to red-flag detection + escalation and shrinks the 8-week plan.*

5. **What is the emergency, after-hours and human-handoff path?** At 2am, on detecting an emergency or being unable to proceed safely: warm transfer to on-call, page a nurse, instruct 911, or the existing answering service? Is it integrated programmatically, at what SLA, and what happens when no human is reachable? — *Defines a first-class escalation subsystem (live-transfer telephony via SIP/REFER, on-call roster integration, guaranteed fallback so no patient is stranded). If the human path is the fallback, its uptime is on our SLA.*

6. **What is the multilingual strategy and quality bar?** English + Spanish only, or more (Navajo in NM, ASL)? Native bilingual model vs machine translation around an English core vs live interpreters? How are code-switching and medical terminology handled, and what accuracy across ASR, reasoning and TTS? — *30% prefer Spanish makes this core. Native bilingual ASR/TTS changes vendor selection and eval-set design; an MT shim adds clinical-safety risk and another BAA hop; an interpreter fallback needs live-human integration. Decides one policy vs two parallel NLU/eval tracks.*

7. **What are the consent, recording and AI-disclosure obligations across AZ, NV, NM?** Are calls recorded or transcribed-and-retained? What consent, in which language, and does any state require disclosing that the patient is speaking to an AI? Do outbound follow-up calls/texts require prior express consent (TCPA), and how is it captured? — *Recording-consent and AI-disclosure rules vary by state and gate the audio pipeline: no consent means no recording, changing QA, replay and the "where does audio go" answer. Disclosure injects a mandatory opening turn + consent capture into every flow.*

8. **What is the PHI logging, retention and audit architecture?** Minimum-necessary for prompts/logs — full transcript, redacted transcript, or metadata only? Retention period and jurisdiction? What audit trail must we produce, and who reviews it? May agent traces go to a non-BAA observability vendor? — *Picks the logging/observability stack (BAA-covered tracing vs self-hosted), forces redaction-at-ingest, and sets data residency. Determines whether failed conversations may be retained to improve the agent — a key reliability lever.*

9. **Are payer eligibility and prior-auth status integrations real, or "we'll call you back"?** Is real-time eligibility (X12 270/271 via a clearinghouse) and prior-auth status available programmatically, or does the agent route to staff? Which payers/plans at go-live (case A names Humana)? — *Cases A and C are integration-dependent. A clearinghouse integration is a distinct 2–3 week workstream with its own vendor and BAA; without it the agent becomes a ticket-creator and containment weakens. Sets committed vs deferred scope.*

10. **What is the telecom and outbound-compliance stack (SMS + follow-up calls)?** Which numbers do we own (clinic lines, portal, SMS short code/10DLC)? Is A2P 10DLC registration complete? How is inbound SMS sender identity verified (case D, unknown number)? Do outbound post-discharge calls need consent and a recognized caller-ID? — *10DLC registration and carrier provisioning are long-lead and can block the entire SMS channel and all outbound outreach — a direct go-live schedule risk. Also shapes safe handling of unverified senders and whether the required two-business-day follow-ups are permitted.*

11. **What must vary by state and by clinic, and who owns that config?** Beyond language: hours, on-call routing, accepted plans, escalation numbers, available services, scope-of-practice rules across AZ/NV/NM and 14 clinics. Who maintains it, and can it change without redeploying the agent? — *Decides a per-state/per-clinic policy layer vs hard-coding one clinic. Case E (visitor near ZIP 85206) needs geo/clinic matching and new-patient acceptance knowledge. Non-data-driven config makes each added clinic an engineering project, undermining 1→14 scalability.*

12. **What are the reliability targets, concurrency and definition of "live"?** Which channel and call types go live first, at what containment/auto-resolve target, latency budget and uptime SLA? What concurrency must the pilot absorb against the 2,800 calls/day baseline, and is graceful degradation to human/callback acceptable? — *These are the acceptance criteria and they size the architecture (streaming ASR vs turn-based, autoscaling, queueing) and the MVP cut. They force transaction-safe, idempotent booking/refills, where reliability failures become patient-safety failures.*

## Section B - Assumptions & Provisional Decisions

Purpose: state the working assumptions and the **provisional (default) decisions** taken against the open questions so architecture design can proceed. Each item records what is assumed, why it is reasonable given the brief, and what changes if it proves incorrect. Consistency with the brief's Context is checked in B.3.

### B.1 Assumptions

**Compliance & security**

| # | Assumption | Why reasonable | If it proves incorrect |
|---|---|---|---|
| A1 | Every PHI-touching hop (carrier, ASR, TTS, translation, LLM/Cadra proxy, hosting) will be under a signed BAA. | HIPAA requires BAAs for all business associates; a healthcare agent cannot legally operate otherwise. | We cannot use any uncovered cloud vendor. Model/ASR/TTS must be BAA-covered or self-hosted; the 8-week plan is materially more expensive and slower. |
| A2 | Saguaro is the Covered Entity; Larkspur acts as a Business Associate. | Saguaro is the provider with patient relationships; Larkspur is a vendor acting on its behalf. | If Larkspur were itself a covered entity, its own HIPAA program, Notice of Privacy Practices and BAA posture would change. |
| A3 | Raw audio is **not** retained by default; only transient processing, with redacted/minimal transcripts. | The privacy officer's single question ("where does the audio go?") signals audio handling is the primary concern. | If full retention is permitted, we gain call replay, QA review and model-improvement data; the storage, metrics and redaction layers simplify. |
| A4 | HIPAA-grade audit logging and access accounting are mandatory and built in from day one. | Required for any system accessing PHI; also needed to answer the privacy officer and support breach response. | Little changes — this remains best practice even if not explicitly enforced. |
| A5 | PHI stays within an agreed US region; no cross-border transfer. | US provider, US patients, US regulatory regime; default expectation of US residency. | If a global model/host is used, we must re-evaluate residency, subcontractor disclosure and possibly export controls. |

**Integration**

| # | Assumption | Why reasonable | If it proves incorrect |
|---|---|---|---|
| A6 | Epic remains the system of record; the agent reads/writes around it and never replaces it. | "Their front desk lives in Epic" — the agent must fit the existing workflow. | If Epic must be displaced or middleware-owned, the integration and change-management scope explodes. |
| A7 | Epic access is available programmatically (Epic on FHIR / App Orchard, at least read, ideally write-back). | Modern Epic deployments expose FHIR; an agent of this ambition requires it. | UI-only/Hyperspace access forces brittle RPA; scheduling/refills/results become "capture intent, queue for humans" and autonomy collapses. An adapter abstraction (below) is our hedge. |
| A8 | A patient portal (MyChart) exists and can support portal chat and serve as an authentication fallback. | The brief explicitly cites "the chat on the patient portal," which implies a live portal. | If it is a bespoke/third-party portal, chat integration and auth fallback use different APIs. |
| A9 | The architecture must be multi-clinic/multi-state configurable from day one even though go-live is one clinic. | The brief frames 1→14 clinics and three states; intent is clearly to scale. | Hard-coding one clinic makes each added clinic an engineering project, breaking the 1→14 path. |
| A10 | We can provision/forward telephony numbers and complete A2P 10DLC registration. | Standard for a US voice/SMS agent; the brief requires phone, text and outbound calls. | If carrier provisioning or 10DLC is blocked/long-lead, the SMS channel and all outbound follow-up calls slip past go-live. |

**Clinical**

| # | Assumption | Why reasonable | If it proves incorrect |
|---|---|---|---|
| A11 | The agent gives **dispositions and routing**, not diagnosis, and is not practicing medicine. | Safer default and consistent with scope-of-practice; "tell patients whether they need to come in" can be met with protocol-based dispositions plus escalation. | If Maya expects autonomous triage/advice, we need a licensed clinical protocol engine (e.g., Schmitt-Thompson) with clinician validation and likely nurse oversight — a major, non-8-week workstream. |
| A12 | A clinical escalation path exists (on-call, nurse line, or the after-hours answering service) and can be reached/notified. | The brief describes an existing answering service and on-call paging after 5pm. | If no reliable human path exists, the agent's only safe fallback for red flags is instructing 911/ED plus a callback ticket — weaker and higher-liability. |
| A13 | Refills are captured and routed for prescriber approval; the agent never autonomously authorizes, and controlled substances are excluded. | DEA/state rules and safety; case A is a routine refill, not an authorization request. | If autonomous renewal of routine meds is expected, clinical governance and pharmacy integration are in scope — much larger. |
| A14 | Post-discharge follow-up is standardized outreach plus red-flag triage with consent, not clinical management. | The brief defines it as a fixed obligation ("within two business days of discharge"). | If it requires individualized clinical assessment, it overlaps A11 and requires clinician review/approval. |

**Language**

| # | Assumption | Why reasonable | If it proves incorrect |
|---|---|---|---|
| A15 | English + Spanish are the only required go-live languages; other languages warm-transfer to a human/interpreter. | The brief quantifies Spanish at ~30% and says most clinics lack bilingual staff — Spanish is the named gap. | If Navajo (relevant to New Mexico) or ASL are required, add parallel ASR/TTS/eval tracks and possibly video — a scope increase. |
| A16 | Spanish is handled by a natively bilingual ASR/NLU/TTS pipeline rather than an English-core MT shim. | Safety-critical medical content plus code-switching make MT wrappers risky; and MT adds another BAA hop. | If only MT is available, add clinical-safety risk, extra vendor/BAA, and degraded evaluation confidence. |

**Operational & scope**

| # | Assumption | Why reasonable | If it proves incorrect |
|---|---|---|---|
| A17 | 8-week MVP = one clinic, phone + portal chat first, covering all five intent classes with safe fallbacks; SMS and full outreach phased. | "Go live at one clinic in eight weeks" implies a constrained pilot, not full 14-clinic, all-channel launch. | If all three channels and every intent must be live at the pilot, the schedule is not credible and scope must be renegotiated. |
| A18 | Human staff (front desk or after-hours) exist to receive warm transfers and queue fallbacks. | The brief describes a staffed front desk and an answering service. | If handoffs are unstaffed, the agent must be more autonomous (raising A11 risk) or degrade to callback-only. |
| A19 | Pilot volume/concurrency is modest (a fraction of ~2,800 calls/day at one clinic). | Volume is 2,800/day across 14 clinics; one pilot clinic absorbs a small share even at peak. | If a single clinic concentrates peak concurrency, autoscaling, queueing and latency-budget design change. |
| A20 | The five example contacts are representative, anonymized, and define the go-live intent set. | The brief presents them as "five recent contacts, with names and details changed." | If the real intent mix differs, the MVP intent taxonomy and routing must be rebuilt from live data. |
| A21 | The agent discloses it is an AI and captures recording/AI-disclosure consent by default. | Safe default across multi-state consent and developing AI-disclosure rules. | If disclosure is not required, we can shorten the opening turn; more friction is removed rather than added. |

### B.2 Provisional Decisions Required Before Architecture

Defaults chosen to unblock design. Each maps to an Open Question (Q#); the phase marked **blocking** cannot be designed without a (provisional) answer.

| Decision | Default taken now | Blocks? | Revisit trigger / owner |
|---|---|---|---|
| PD1 (Q1) | All PHI hops BAA-covered; model endpoint must be BAA-covered or self-hosted. | **Blocking** | Any uncovered vendor discovered → self-host ASR/TTS. |
| PD2 (Q1) | No raw-audio retention by default; transcripts redacted/minimal, configurable per Saguaro. | **Blocking** | Privacy officer permits retention → enable replay/QA. |
| PD3 (Q3) | Prove identity with name + DOB **plus** a second factor (last-4 SSN, OTP, or portal login); caller-ID is a hint only. | **Blocking** | If Saguaro disallows KBA/OTP → PHI withheld on phone/SMS. |
| PD4 (Q4) | Clinical scope = disposition + red-flag escalation only; no diagnosis, no autonomous advice. | **Blocking** | Maya requires true triage → engage protocol/nurse workstream. |
| PD5 (Q5) | Human escalation is a required integration; safe fallback is 911/ED instruction + callback ticket. | **Blocking** | Confirm on-call/answering-service integration point and SLA. |
| PD6 (Q6) | English + Spanish go-live via natively bilingual pipeline; other languages route to humans. | **Blocking** | Additional required language → new ASR/TTS/eval track. |
| PD7 (Q2) | Assume FHIR-based Epic access; isolate all Epic calls behind an adapter so a UI/RPA path can substitute. | **Blocking** | Actual contracted surface confirmed → swap adapter. |
| PD8 (Q11) | Per-state/per-clinic config layer from day one (hours, routing, plans, escalation numbers, scope rules). | **Blocking** | Confirm which fields vary by site at go-live. |
| PD9 (Q8) | Audit/observability stack is BAA-covered or self-hosted with redaction-at-ingest. | **Blocking** | Vendor trace tool lacks BAA → self-host. |
| PD10 (Q9) | Eligibility/prior-auth treated as **deferred** unless a clearinghouse integration is confirmed; agent acknowledges and creates a staff ticket meanwhile. | Non-blocking (scope) | Confirm clearinghouse → commit 2–3 week workstream. |
| PD11 (Q10) | SMS/outbound assumed long-lead (10DLC); pilot starts with phone + portal chat; follow-up outreach phased. | Non-blocking (schedule) | 10DLC timeline confirmed → re-sequence channels. |
| PD12 (Q12) | Acceptance = one clinic live, phone + portal chat, all five intents with safe fallbacks; human-in-the-loop allowed. | **Blocking** (for MVP cut) | Confirm containment/latency/uptime targets with Saguaro. |

### B.3 Consistency Check vs the Brief's Context

- **No contradiction on scale/geography.** A9 assumes multi-clinic/multi-state config, matching the 14 clinics across AZ/NV/NM in the brief.
- **No contradiction on language.** A15/A16 align with "~30% Spanish, most clinics lack bilingual staff" — Spanish is treated as core, not optional.
- **No contradiction on systems.** A6/A7/A8 respect "the front desk lives in Epic" and the portal chat channel; we integrate rather than replace.
- **No contradiction on after-hours.** A12 ties escalation to the existing answering service / on-call paging described in the brief, rather than inventing a new one.
- **No contradiction on obligations.** A14 keeps the two-business-day post-discharge follow-up as a hard requirement.
- **One tension flagged, resolved provisionally.** The brief says the agent should "tell patients whether they need to come in," which could imply autonomous clinical judgement; A11/PD4 provisionally restrict this to protocol-based dispositions plus escalation. **This is the single most important assumption to validate with Maya and Saguaro's clinical leadership before architecture is frozen.**
- **Boundary respected.** A20 assumes the five contacts are anonymized examples used to define MVP intents, consistent with the brief's "names and details changed."

## Section C — Solution design

Design consistent with Section A (open questions) and Section B (assumptions A1–A21, provisional decisions PD1–PD12). Each design choice below cites the assumption or decision it depends on, so a change in Section B propagates cleanly.

### C.1 Design principles

1. **One conversation core, many channels.** Phone, SMS and portal chat are adapters into a single channel-agnostic orchestration layer so intent, identity, policy and handoff logic are defined once (A17, PD12).
2. **Policy constrains the model.** Regulated flows are deterministic state machines; the LLM interprets and phrases, but cannot decide clinical or disclosure actions (A11, PD4).
3. **No PHI without proof.** Identity & Auth is a hard gate before any record access; caller-ID and self-declared DOB are hints only (PD3).
4. **Systems of record stay authoritative.** Epic remains the chart; the agent integrates through an adapter and never replaces it (A6, PD7).
5. **Minimise and audit everything.** Raw audio is not retained; data is redacted at ingest and every access is logged (A3, A4, PD2, PD9).
6. **Never strand a patient.** Every path has a human escalation fallback, including a 911/ED instruction when no human is reachable (A12, PD5).

### C.2 System architecture — channels into a shared orchestration layer

The end-to-end diagram is in **[ARCHITECTURE-DIAGRAM.md](./ARCHITECTURE-DIAGRAM.md)**.

**Channel & Media Gateway (BAA-covered).**
- **Voice:** PSTN/SIP ingress; real-time bilingual streaming ASR, TTS with barge-in, VAD and silence/timeout handling. Audio is processed transiently and discarded; only the transcript and a redacted record persist (A3, PD2).
- **SMS:** A2P 10DLC messaging; inbound sender is treated as unverified (PD3, PD11).
- **Portal chat:** session-token authenticated channel; also an authentication fallback (A8).

**Shared AI Orchestration Layer (channel-agnostic).**
- **Session & Context Manager** — holds conversation state, detected language, verified-identity level and prior actions across turns and across channel switches (e.g., chat → callback).
- **Identity & Auth Service** — KBA + OTP + portal-login verification, rate limiting and lockout; resolves authorized-representative/proxy relationships (PD3).
- **Intent + Red-Flag Classifier** — first pass that separates routine intents from safety-critical signals before any other handling (A11, PD4).
- **Dialogue & Policy Engine** — deterministic flows for scheduled, refill, results, plan, prior-auth, outreach and escalation; owns consent/AI-disclosure and language selection (A21, PD6).
- **Constrained LLM** — natural-language understanding and generation, function/tool calling only; no autonomous clinical or disclosure decisions (PD4).
- **Clinical Safety & Disposition** — protocol-based disposition and red-flag escalation; explicitly not diagnosis (PD4).
- **Tool / Action Layer** — scoped, audited, idempotent calls to integrations; enforces identity and consent preconditions (PD3, PD12).
- **Context Handoff & Warm Transfer** — packages conversation context for staff (C.5).
- **Config & Policy** — per-clinic/per-state parameters (hours, routing, accepted plans, escalation numbers, scope rules) loaded as data, not code (A9, PD8).

**Integrations.** Epic Adapter (FHIR primary, RPA fallback behind the same interface), Scheduling Service, Refill Service, Payer/Eligibility (deferred unless a clearinghouse is confirmed), Escalation/On-call + answering service, and the Outreach Scheduler for post-discharge follow-ups driven against the two-business-day obligation (A14).

**Data & Compliance.** Redaction-at-ingest, an immutable audit log, and ephemeral session state; no raw audio store (A3, A4, A5, PD9).

### C.3 Agent workflow — classify, route, process, complete

Every contact follows the same eight stages; the channel only changes the media layer.

1. **Ingest & normalise** — media → transcript/event; detect language; capture consent/AI disclosure (A21, PD6).
2. **Identify** — establish identity level before PHI; unverified contacts proceed in a no-PHI lane (PD3).
3. **Classify** — intent + red-flag classification. Red flags short-circuit to safety (A11).
4. **Route** — policy engine selects a regulated flow and the correct clinic/state config (PD8).
5. **Process** — tool calls to Epic/scheduling/refill/payer under identity and consent preconditions (PD7).
6. **Resolve or escalate** — complete with confirmation, or hand to a human with context (PD5).
7. **Handoff** — structured packet + warm transfer when required (C.5).
8. **Audit** — redacted record and access trail written (PD9).

**Autonomy matrix (what the AI can and cannot do independently).**

| Capability | Independent action | Condition | Requires human / never autonomous |
|---|---|---|---|
| Identity verification | Run KBA/OTP/proxy checks | Always required before PHI | Disclosing PHI when proofing fails |
| Schedule / move appointment | Book within configured slot types and rules | Established or registered patient; idempotent | Overrides, special slots, policy exceptions |
| Refill capture | Capture request + route to prescriber queue | Non-controlled medication (A13) | Approving/authorizing refills; any controlled substance |
| Result / record read | Release after verification + release policy | Identity + policy satisfied | Release contrary to holding policy; unverified requesters |
| Plan acceptance ("do you take my plan") | Answer from clinic/payer config | Config current | Coverage adjudication / billing disputes |
| Prior-auth status | Read if integration exists | Clearinghouse confirmed (PD10) | If integration absent → staff ticket |
| Clinical triage | Red-flag detection + protocol disposition | Protocol-backed (A11, PD4) | Diagnosis, advice, or reassurance |
| Post-discharge outreach | Place call/text + scripted red-flag screen | Consent present (A21); within 2 business days (A14) | Any clinical management of findings |
| Complaints, sensitive topics, minors, behavioral health | Capture only | — | Human handling; warm transfer |

### C.4 Patient record, appointment and insurance workflows (subject to confirmed API capabilities)

Because the exact Epic surface is unconfirmed (A7, PD7), all record work sits behind the **Epic Adapter** with three capability tiers; the tier is configuration, not a rewrite.

- **Tier 1 — FHIR read + write:** full workflow. Patient lookup, `Appointment`/`Slot` search and booking, `MedicationRequest` reads, `DocumentReference` for results/prior-auth letters, `RelatedPerson` for proxies.
- **Tier 2 — read only / external APIs:** agent reads and confirms, but booking and refill actions are emitted as staff tasks rather than written back.
- **Tier 3 — RPA / UI only:** agent captures intent and creates a structured task; no autonomous write-back.

**Patient record.** Lookup is by verified identity only. Reads are limited to minimum-necessary fields for the active flow and are written to the audit log (PD9). Proxy/caregiver access requires an established relationship (`RelatedPerson`) or documented authorization, else refused (PD3).

**Appointment scheduling.** Slot search is filtered by clinic, appointment type, provider availability and new-vs-established status (A9). Booking and moves are **idempotent** with a confirmation token to prevent double-booking (PD12). Cancellations, waitlists and reminders follow clinic config. Anything outside configured rules escalates.

**Insurance / plan and prior authorization.** Plan acceptance is answered from clinic/payer config. Real-time eligibility (X12 270/271) and prior-auth status are **deferred until a clearinghouse is confirmed** (PD10); until then the agent acknowledges, captures the details (e.g., the Humana change in contact A), and creates a billing/staff ticket rather than guessing.

### C.5 Transferring context to clinic staff (no repetition)

When a human is needed, the agent sends a **structured handoff packet** so the patient never repeats themselves:

- verified identity level and method (not the secret factors);
- channel, detected language and clinic/state context;
- classified intent, red flags and consent status;
- actions already attempted, with their result IDs (e.g., booking reference, refill ticket);
- pending items and a suggested next step;
- a redacted transcript excerpt;
- a deep link into the Epic chart / staff queue.

**Voice** handoff uses a warm transfer (SIP REFER) with the packet attached for a screen-pop. **Chat/SMS** handoff becomes a task in the staff queue / Epic In Basket carrying the same packet, with the option to continue the conversation on the patient's channel. This keeps continuity across the answering-service/on-call boundary too (A12, PD5).

### C.6 How the architecture handles each recent contact

The brief lists five contacts (A–E); each is traced through the stages above. Note the "five intent classes" in D.1 are the go-live capability set (scheduling, refills, plan/prior-auth questions, post-discharge follow-up, and disposition/routing) and are **not** the same as contacts A–E, which are illustrative examples — contact B is an emergency and contact D a results request, neither of which is a standalone go-live intent class.

- **A — Phone, Las Vegas, Mon 7:52am. Refill + insurance change (Humana).** Bilingual-capable voice lane; identity verified (name + DOB + second factor). Refill intent → `MedicationRequest` read confirms lisinopril is non-controlled → request captured and routed to the prescriber queue with confirmation; the agent does **not** authorize it (A13). Plan question answered from config; the Humana change is captured and routed to billing because eligibility is deferred (PD10). No red flags; completes with confirmations and ticket IDs; audit written.
- **B — Portal chat, Thu 11:40pm. Post-HF-discharge worsening shortness of breath and ankle swelling.** Red-flag classifier fires immediately on "more short of breath" + "ankles look bigger" after a heart-failure discharge; the flow short-circuits before any scheduling. The agent gives a **safety disposition only** — advise seeking emergency care now (911/ED) — and does **not** diagnose or reassure (A11, PD4). After-hours, it raises an urgent handoff to on-call via the escalation integration with the C.5 packet (A12, PD5). If the caller is a proxy, safety guidance is given without waiting on PHI proofing; no chart details are disclosed.
- **C — Phone, Tucson, Tue 2:15pm (Spanish). Prior-auth status for spouse's MRI.** Spanish lane via the bilingual pipeline (PD6). Intent = prior-auth status + authorized-representative. Because the caller is speaking for her husband, disclosure requires a verifiable proxy relationship (`RelatedPerson` or documented authorization); if unverified, the agent explains it cannot release the status and routes to staff (PD3). If a clearinghouse/prior-auth integration is confirmed it reads the status; otherwise it creates a callback ticket within the two-business-day expectation (PD10). Completes with a clear next step in Spanish.
- **D — SMS, unknown number. "Dana Whitfield DOB 3/14/1961 … text me my A1C results."** Inbound sender is unverified and self-declared DOB is insufficient (PD3). The request is for PHI (lab results) over an insecure channel, so the agent **does not** text results. It withholds disclosure, explains why, and offers verified paths — an OTP to a number on file or MyChart login; results release then follows holding policy. No PHI is returned in the unverified lane; the exchange is audited.
- **E — Phone, 612 area code, Feb. Seasonal visitor in Mesa 85206, new patient, same-week cough appointment.** Area code is treated as a hint; the agent geo-matches the ZIP to the appropriate Phoenix-cluster clinic and checks clinic config for new-patient acceptance and availability (A9, PD8). The cough is screened for red flags; absent red flags it offers same-week slots within configured rules, and if new-patient registration/records transfer is required it routes that step rather than assuming the patient is established. It cannot guarantee a same-week slot or a new-patient acceptance; completes with a booking or a routed task and context for staff.

### C.7 Architecture diagram

See **[ARCHITECTURE-DIAGRAM.md](./ARCHITECTURE-DIAGRAM.md)** for the full channel → gateway → orchestration → integrations → data/compliance flow.

## Section D — Eight-week scope

### D.1 MVP definition

The smallest **safe and useful** agent that can go live at one clinic in eight weeks, consistent with A17/PD12 and the Section C architecture.

**MVP = one pilot Arizona clinic · phone + portal chat · all five intent classes in a *safe-fallback* form · English + Spanish · human-in-the-loop.** SMS and outbound post-discharge dialing are phasing targets gated on 10DLC and consent; eligibility/prior-auth remain ticket-based until a clearinghouse is confirmed (PD10, PD11).

Pilot clinic selection criteria (not a name, a test): Epic integration readiness (sandbox + scopes), existing bilingual staff for handoff validation, a defined on-call/answering-service escalation path, and manageable peak volume (A19).

### D.2 Capability evaluation — automate vs staff approval

| Capability | Safe to automate | Requires staff approval / human |
|---|---|---|
| **Appointment requests** | Slot search, book, move, cancel within configured types/rules; idempotent confirmation (C.4) | Exceptions, provider-specific overrides, waitlist policy, new-patient onboarding decisions, same-day overbooking |
| **General clinic information** | Hours, locations, directions, services, accepted plans, refill/records policy — answered from clinic config | Anything requiring interpretation of coverage or clinical suitability |
| **Patient routing** | Intent classification, department/provider/clinic routing, language routing, warm transfer | Complaints/grievances, sensitive topics, minors, behavioral health, ambiguous intent after bounded clarification |
| **Identity verification** | KBA + OTP/portal-login challenge and PHI gating; proxy lookup against `RelatedPerson` | Any PHI release when proofing fails; proxy disputes; identity-edge cases |
| **Human handoff** | Context packet generation, warm transfer, task creation, schedule callback | The conversation itself once escalated |

Routine appointment requests, general info, routing and identity verification are **safe to automate** under Section C's autonomy matrix. Anything clinical, any failed identity, and any policy exception is **staff-approved only** (A11, A13, PD3, PD4).

### D.3 Eight-week plan

| Phase | Weeks | Objective | Key work | Exit criteria |
|---|---|---|---|---|
| **1. Requirements & alignment** | 1 | Freeze scope, sign-offs, access | Confirm Section B assumptions with Maya/clinical/privacy; intent taxonomy from the five contacts (A20); red-flag protocol and disposition scripts approved; pilot clinic selected; BAAs initiated; Epic access request (PD1, PD7) | Signed scope, approved safety scripts, named clinic, BAA/access in flight |
| **2. Integration validation** | 2 | Prove paths before building | Epic sandbox FHIR validation across the required resources and capability tier (C.4); telephony + portal-chat connectivity; bilingual ASR/TTS quality check; identity/OTP sandbox; start 10DLC (PD11) | End-to-end "hello world" on each channel; tier confirmed; latency measured |
| **3. Implementation** | 3–5 | Build the orchestration + flows | Session manager, identity gate, classifier, policy flows for all five intents, Epic adapter, escalation + handoff packet, audit/redaction, per-clinic config (C.2–C.5) | Flows demo end-to-end in sandbox in EN + ES; audit writes verified |
| **4. Safety testing** | 6 | Break it before patients do | Red-team red-flag/disclosure cases; recall test on emergency set; PHI-without-proof tests; Spanish clinical QA; idempotency/duplicate-booking tests; escalation drills incl. no-human-available; load test | Zero unauthorized disclosures, 100% red-flag recall on test set, escalation SLA met |
| **5. Pilot deployment** | 7 | Go live safely, staged | Shadow mode → limited-hours live at one clinic; staff training on handoff packet; rollback drill; observability dashboards | Staff sign-off; rollback proven; monitored live window |
| **6. Evaluation & hypercare** | 8 | Confirm and stabilise | Review success criteria, incident/QA review, tune intents/scripts, decide next-channel/next-clinic readiness | Go/no-go report against D.6; backlog for phase 2 |

### D.4 Exclusions (explicitly deferred)

| Deferred | Reason |
|---|---|
| Autonomous clinical triage / diagnosis / advice | Clinical risk + liability; needs licensed protocol engine and clinician validation (A11, PD4) |
| Autonomous refill authorization; controlled substances | DEA/state rules; prescriber approval required (A13) |
| Real-time eligibility (X12 270/271) and prior-auth adjudication | Integration dependency; needs clearinghouse + BAA + 2–3 weeks (PD10) |
| SMS channel and outbound post-discharge follow-up calls | Long-lead 10DLC/consent; schedule risk (A10, PD11) |
| Languages beyond English/Spanish (e.g., Navajo, ASL) | Scope/time; routed to humans (A15) |
| Multi-clinic/multi-state rollout | Pilot is one clinic; config layer built to scale later (A9) |
| Payments, billing, insurance-change adjudication | Out of scope; captured and routed |
| New-patient self-registration automation | Edge-case risk; routed to staff (contact E) |

**Confirm, don't assume:** the brief states the post-discharge follow-up ("within two business days of a hospital discharge") as a hard obligation, so phasing it out of the pilot is a scope decision that must be explicitly agreed with Maya, not silently deferred (A14).

### D.5 Dependencies

| Dependency | Need | Owner | Risk if late |
|---|---|---|---|
| Epic access | Contracted FHIR scopes + sandbox + write-back tier | Saguaro IT / Epic | Blocks all scheduling/refill/record work (PD7) |
| HIPAA / BAAs | Signed BAAs for carrier, ASR, TTS, LLM, hosting; privacy-officer sign-off on audio flow | Larkspur + Saguaro privacy | Legal blocker for any PHI (A1, A2, PD1, PD2) |
| Clinical approval | Red-flag protocol + disposition scripts + liability owner | Saguaro clinical leadership | Blocks safety testing/go-live (A11, PD4) |
| Clinic staff availability | Training, UAT, handoff acceptance | Pilot clinic manager | Blocks pilot deployment (A18) |
| Operational readiness | On-call/answering-service integration + escalation SLA | Saguaro ops | Weakens escalation fallback (A12, PD5) |
| Telecom / 10DLC | Number provisioning + campaign registration | Larkspur | Defers SMS/outbound (A10, PD11) |
| Clinic config data | Hours, plans, routing, escalation numbers | Pilot clinic | Blocks general-info + routing accuracy (PD8) |

### D.6 Success criteria (proposed pilot thresholds)

| Dimension | Measure | Proposed threshold |
|---|---|---|
| **Safety** | Unauthorized PHI disclosures; red-flag recall on emergency set; emergency escalation within SLA | 0 disclosures; 100% recall; ≥ 95% within SLA |
| **Reliability** | Uptime; p50/p95 response latency; duplicate bookings; fallback success on integration failure | ≥ 99.5% uptime; p95 within budget; 0 duplicates; 100% clean fallback |
| **Request handling** | Containment/auto-resolve for routine intents; correct routing; ticket accuracy | ≥ 60% containment; ≥ 95% correct routing; ≥ 98% ticket accuracy |
| **Patient experience** | CSAT; language match; abandonment; "asked to repeat" rate | CSAT ≥ 4/5; ≥ 95% correct-language; ≤ existing abandonment; repeat rate ≈ 0 |
| **Escalation quality** | Warm-transfer success; context-packet completeness; staff-rated usefulness; time-to-human | ≥ 99% transfer success; 100% packets complete; staff rating ≥ 4/5; time-to-human ≤ current |

Targets are proposals to confirm with Saguaro (PD12).

### D.7 Rollback

- **Granular kill switches.** Feature flags per channel, per intent and per clinic, plus an emergency global stop.
- **Safe mode.** One switch degrades the agent to a **message-taker**: capture the request, create a staff task, and warm-transfer — the existing workflow continues.
- **Full disable.** Phone returns to the existing PBX/answering service, portal chat goes to staff, SMS stays off; no patient is left without a path.
- **Circuit breakers.** On Epic/payer/escalation failure, the tool layer auto-falls back to human/queue rather than degrading autonomously (C.3, PD12).
- **Authority & testing.** Named on-call owner can pull the switch 24/7; the rollback path is drilled in Week 7 and monitored throughout.
- **Data integrity.** Idempotent bookings mean no partial writes; the immutable audit log is retained for review regardless of rollback (A4, PD9).

### D.8 Consistency with Sections B and C

- **Scope matches the assumptions.** MVP = one clinic, phone + portal chat, all five intents with safe fallbacks, EN/ES, human-in-the-loop — exactly A17/PD12. SMS/outbound deferred per PD11; eligibility/prior-auth deferred per PD10.
- **Architecture supports it.** The channel-agnostic orchestration layer (C.2) lets SMS phase in without redesign; the Epic adapter (C.4) absorbs whichever capability tier is confirmed (PD7); the autonomy matrix (C.3) enforces the automate-vs-staff split in D.2.
- **Safety posture matches Section B.** No-diagnosis (A11/PD4), no autonomous refills (A13), identity gate before PHI (PD3), and a guaranteed escalation fallback (A12/PD5) are all preserved.
- **No contradiction with the brief.** The two-business-day post-discharge obligation (A14) is acknowledged and explicitly phased with SMS/outbound rather than dropped; the pilot remains one clinic within the eight weeks stated by Maya.
- **Feasibility.** No phase depends on a non-blocking decision (PD10/PD11) to go live; the blocking *design* decisions (PD1–PD9, PD12) are provisionally set by Week 2, but the external dependencies they rely on — signed BAAs and live Epic access — must land early and are the primary schedule risk (D.5), not resolved merely by choosing a default.

## Section E — Next step

**The single most important next action:** obtain Saguaro's written clinical sign-off on the safety boundary — that the agent gives protocol-based dispositions and red-flag escalation but never diagnoses or advises (A11, PD4) — because it resolves the one tension flagged in Section B.3 and gates the Section C autonomy matrix, Week 6 safety testing, and the eight-week go-live.