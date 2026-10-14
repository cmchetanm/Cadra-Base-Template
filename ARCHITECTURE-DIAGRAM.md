# Larkspur Health — System Architecture Diagram

Linked from `APPROACH.md` § C. All PHI-touching nodes are BAA-covered or self-hosted; no raw audio is retained (Section B: A1–A5, PD1–PD2).

```mermaid
flowchart TB
    subgraph PAT["Patient Channels"]
        direction LR
        PH["Phone<br/>(PSTN / SIP)"]
        SM["SMS<br/>(A2P 10DLC)"]
        PC["Patient Portal Chat<br/>(MyChart)"]
    end

    subgraph GW["Channel &amp; Media Gateway (BAA-covered)"]
        direction LR
        VGW["Voice Gateway<br/>bilingual streaming ASR + TTS<br/>barge-in · no audio retention"]
        SGW["SMS Gateway<br/>sender-ID hint only"]
        CGW["Chat Gateway<br/>session-token auth"]
    end

    subgraph ORCH["Shared AI Orchestration Layer"]
        direction TB
        SES["Session &amp; Context Manager<br/>channel-agnostic state · language"]
        AUTH["Identity &amp; Auth Service<br/>KBA + OTP · proxy/authorized rep"]
        CLS["Intent + Red-Flag Classifier"]
        POL["Dialogue &amp; Policy Engine<br/>deterministic regulated flows"]
        LLM["Constrained LLM<br/>understand · generate (tool-calling)"]
        SAFE["Clinical Safety &amp; Disposition<br/>protocol-based, no diagnosis"]
        TOOLS["Tool / Action Layer<br/>scoped, audited, idempotent"]
        HAND["Context Handoff &amp; Warm Transfer"]
        CFG["Config &amp; Policy<br/>per-clinic / per-state"]
    end

    subgraph INT["Systems of Record &amp; Integrations"]
        direction TB
        EMR["Epic Adapter<br/>FHIR · RPA fallback"]
        SCHED["Scheduling Service"]
        REFILL["Refill Service"]
        PAYER["Payer / Eligibility<br/>X12 270/271"]
        ESC["Escalation / On-call<br/>+ Answering Service"]
        CAMP["Outreach Scheduler<br/>post-discharge follow-ups"]
    end

    subgraph DATA["Data &amp; Compliance"]
        direction LR
        RED["Redaction at Ingest"]
        AUD[("Immutable Audit Log")]
        STATE[("Ephemeral Session State")]
    end

    PH --> VGW
    SM --> SGW
    PC --> CGW
    VGW --> SES
    SGW --> SES
    CGW --> SES

    SES --> AUTH
    AUTH --> CLS
    CLS --> POL
    POL <--> LLM
    POL --> SAFE
    POL --> TOOLS
    CFG -.-> POL
    CFG -.-> TOOLS

    TOOLS --> EMR
    TOOLS --> SCHED
    TOOLS --> REFILL
    TOOLS --> PAYER
    SAFE --> ESC
    POL --> HAND
    HAND --> ESC
    HAND --> EMR

    CAMP --> VGW
    CAMP --> SGW

    EMR --> RED
    RED --> AUD
    RED --> STATE
    AUTH --> AUD
    TOOLS --> AUD
    SAFE --> AUD
```

## Legend

- **Channels** normalise into one conversation event model; the orchestration layer is channel-agnostic (Section B: A17, PD12).
- **Identity & Auth** gates every PHI access; caller-ID and self-declared DOB are never sufficient (PD3).
- **Dialogue & Policy Engine** owns regulated flows; the **LLM** only understands and generates language under policy control (A11, PD4).
- **Clinical Safety & Disposition** performs red-flag detection and protocol dispositions, then escalates — it does not diagnose.
- **Epic Adapter** is an abstraction boundary so a FHIR→RPA capability change is config, not a rewrite (A7, PD7).
- **Payer / Eligibility** is shown as deferred/optional until a clearinghouse is confirmed (PD10).
- **Redaction → Audit** ensures only minimised, minimised-at-ingest data is retained; audio is not (A3, PD2, PD9).
