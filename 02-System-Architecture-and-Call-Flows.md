# 02 — System Architecture and Call Flows

This chapter answers one question: **when Alice presses PTT, what actually happens in the network?**

You do not need to memorize every box. Read once for the story, then come back to the diagrams when you debug.

Companion textbook: [01-MCPTT-Master-Handbook.md](01-MCPTT-Master-Handbook.md).

---

## 1. The big picture in everyday words

Think of three layers stacked on top of each other:

1. **Radio + packet core (Open5GS)**  
   The phone attaches to the cell, proves its SIM, and gets an IP address. Without this, the MCPTT app has nowhere to send packets.

2. **Call control (IMS P-CSCF + MCPTT application server)**  
   The app registers and places calls with SIP (the same family of messages VoLTE uses). The first SIP hop is the **P-CSCF**. That proxy forwards MCPTT calls to our **application server** (`mcx-as`), which decides who is in the group call and tells the media server to open ports.

3. **Media (floor + audio/video)**  
   Who may talk is decided by **floor control**. The actual voice/video bytes are **RTP** through `mcx-media`.

On top of that, operators use **Hub** (a browser portal). Hub is *not* part of 3GPP MCPTT — it is just how you configure users and watch the lab.

### Why quality of service matters here

If MCPTT voice ran on the same “best effort” internet bearer as WhatsApp, a busy cell could make PTT choppy. Standards therefore define special **QoS classes** (QCI on 4G, 5QI on 5G) for mission-critical traffic:

| Class | Typical use |
|------:|-------------|
| **65** | MCPTT voice (and related floor media) |
| **67** | MCVideo |
| **69** | MCPTT signalling (SIP) |
| **70** | MCPTT data (CMS, GMS, IdMS, file distribution, …) |
| **1** | Ordinary VoLTE audio (must stay available for normal IMS calls) |

### How this lab asks the core for those classes

**The intended, standards-based way (this is what we build toward in production):**

When a call starts, the **P-CSCF** already sees the SIP/SDP. It tells the policy engine “this session needs MCPTT treatment.”

- On **4G:** P-CSCF → **PCRF** (Diameter **Rx**) → PCRF tells the gateway (**Gx**) → the **SMF/UPF** path installs a **dedicated bearer** with the right QCI.  
- On **5G:** the same idea uses **N5** toward the **PCF**, then **SMF**, then **UPF**.

So we are not inventing a private QoS hack. We are using a normal **3GPP policy and charging control (PCC)** loop, with Open5GS and Kamailio modified so they understand MCPTT application identifiers and those QCIs/5QIs.

**What we do *not* sell as the industrial answer:** pre-loading city-wide packet filters in Mongo (“MC Critical” / `mc-rules`). That can help a lab demo when radio traces are hard, but it is a **temporary fallback**, not the architecture you want to ship.

---

## Diagram A — Boxes on the lab host

```mermaid
flowchart TB
  subgraph Host["Lab host"]
    HUB["Hub :18100 — operator browser"]
    ADMIN["Admin console"]
  end
  subgraph Core["Packet core — Open5GS"]
    MME[MME]
    SMF[SMF]
    UPF[UPF]
    PCRF[PCRF]
    PCF[PCF]
  end
  subgraph IMS["IMS — Kamailio"]
    PCSCF["P-CSCF\nMCPTT SIP + VoLTE"]
  end
  subgraph MC["MCPTT application"]
    AS[mcx-as]
    MEDIA[mcx-media]
    CMS[CMS]
    GMS[GMS]
    IDMS[IdMS]
    DB[(Postgres)]
  end
  UE["Phone + APK"]
  RAN["eNB / gNB"]
  UE --- RAN --> MME
  UE -->|user IP traffic| UPF
  UPF --> PCSCF
  UPF --> CMS
  UPF --> MEDIA
  PCSCF -->|SIP| AS
  PCSCF -->|call-time policy Rx| PCRF
  PCRF -->|install bearer Gx| SMF
  SMF --> UPF
  PCSCF -.->|5G policy N5| PCF
  AS --> MEDIA
  AS --> DB
  CMS --> DB
  HUB --> ADMIN
```

**How to read this.** The phone’s traffic reaches MCPTT servers through the **UPF**. Signalling for calls goes **phone → P-CSCF → application server**. When a call needs priority treatment, **P-CSCF talks to PCRF (or PCF on 5G)**, and the core updates the **UPF** so the radio treats those packets as MCPTT QCIs — not as ordinary internet.

---

## Diagram B — What runs inside the phone vs the servers

```mermaid
flowchart LR
  subgraph Client["On the phone"]
    UI[Screens]
    ENG[Mcx engine]
    MEDC[Audio / video sockets]
  end
  subgraph Id["Who am I / what groups"]
    IDMS2[IdMS login]
    CMS2[CMS profile]
    GMS2[GMS groups]
  end
  subgraph Ctrl["Call control"]
    PCSCF2[P-CSCF]
    AS2[Application server]
  end
  subgraph MP["Talk path"]
    FLOOR[Floor control]
    RTP[Voice / video relay]
  end
  UI --> ENG --> PCSCF2 --> AS2
  ENG --> IDMS2
  ENG --> CMS2
  ENG --> GMS2
  AS2 --> FLOOR
  MEDC --> RTP
  FLOOR --- RTP
```

Floor control and RTP relay live in **one** process (`media_server.py`). CMS and GMS are separate containers from the same software image, started with different roles.

---

## Diagram C — Getting an IP on LTE (before any MCPTT)

```mermaid
sequenceDiagram
  participant UE as Phone
  participant eNB as Cell
  participant MME
  participant HSS
  participant UPF as Gateway / UPF
  participant DNS
  participant MC as MCPTT servers
  UE->>eNB: Radio
  eNB->>MME: Attach
  MME->>HSS: Authenticate SIM
  MME->>UPF: Create session (APN internet)
  UPF-->>UE: IP address + DNS
  UE->>DNS: Look up cms / sip / idms names
  UE->>MC: HTTPS, SIP, RTP
```

Until this finishes, the app cannot register or call.

---

## Diagram D — Same idea on 5G SA

```mermaid
sequenceDiagram
  participant UE as Phone
  participant gNB
  participant AMF
  participant SMF
  participant UPF
  participant PCF
  participant MC as MCPTT servers
  UE->>gNB: Radio
  gNB->>AMF: Register
  AMF->>SMF: Start PDU session
  SMF->>UPF: Program user plane
  SMF->>PCF: Ask for policy
  UPF-->>UE: IP on the data network
  UE->>MC: App traffic
```

This lab’s day-to-day radio demos are usually **LTE**. 5G SA compose files exist; treat a full BladeRF 5G+MCPTT demo as something you verify when that radio path is up.

---

## Diagram E — Logging in (several keys, one goal)

```mermaid
flowchart TB
  A[SIM authenticates to the core] --> B[Phone has an IP]
  B --> C{App sign-in}
  C -->|username + password| D[IdMS gives an access token]
  C -->|SIM button| E[CMS helps exchange a token]
  D --> F[CMS / GMS with that token]
  E --> F
  F --> G[SIP REGISTER at P-CSCF with Bearer]
  G --> H[Application server remembers the user is online]
```

Cellular attach and app login are **different**. Attach gives IP. Login gives the MCPTT identity token used for CMS and SIP.

---

## Diagram F — A normal group call (Hold to talk)

```mermaid
sequenceDiagram
  participant A as Alice
  participant P as P-CSCF
  participant AS as Application server
  participant M as Media
  participant B as Bob
  A->>P: INVITE (group + SDP)
  P->>AS: forward SIP
  AS->>M: create session, add Alice
  AS->>A: 200 OK with media ports
  AS->>B: INVITE (Bob is affiliated)
  B->>AS: 200 OK
  AS->>M: store Bob's ports
  A->>M: Floor Request
  M->>A: Floor Granted
  M->>B: Floor Taken (Alice is talking)
  A->>M: voice RTP
  M->>B: relay RTP
  A->>M: Floor Release
  A->>AS: BYE
```

If Bob presses PTT while Alice still holds the floor, he is **queued** (same group, normal priorities). He does not steal the floor just because his user priority number is higher. Elevating to **emergency** or **imminent peril** is a separate, explicit action.

While this INVITE/SDP is visible at the P-CSCF, the core can open the dedicated MCPTT bearers described in section 1.

---

## Diagram G — Floor states (who may speak)

```mermaid
stateDiagram-v2
  [*] --> Idle
  Idle --> Granted: someone requests and nobody holds
  Granted --> Idle: release, max talk time, or silence too long
  Idle --> Queued: request while someone else holds
  Queued --> Granted: holder releases
  Granted --> GrantedEmergency: emergency / imminent-peril may preempt
```

Implementation: `media_server.py` (`floor_request`, grant, queue, timers).

---

## Diagram H — Audio and video bytes

```mermaid
flowchart LR
  MIC[Mic / camera] --> ENC[Encode]
  ENC --> UDP[UDP to media server]
  UDP --> RELAY[Relay only from floor holder]
  RELAY --> PEER[Peers decode and play]
  FLOOR[Floor messages] --- RELAY
```

---

## Diagram I — Operator changes a user in Hub

```mermaid
sequenceDiagram
  participant Op as Operator
  participant Hub
  participant Adm as Admin API
  participant DB as Database
  participant CMS
  participant UE as Phone
  Op->>Hub: open MCPTT admin
  Op->>Adm: change user flags
  Adm->>DB: save
  Note over UE,CMS: Phone picks up new rights on next profile fetch
  UE->>CMS: get user profile
  CMS->>DB: read
  CMS-->>UE: XML
```

---

## Diagram J — What must be up before calls work

```mermaid
flowchart TB
  prepare[prepare.sh — secrets and certificates] --> compose[docker compose up]
  compose --> DB[mcx-db healthy]
  DB --> CMS
  DB --> GMS
  DB --> AS
  MEDIA[mcx-media] --> AS
  PKI[Lab certificates] --> PCSCF[P-CSCF]
  PKI --> CMS
  compose --> provision[provision_mc.sh]
```

---

## Trust boundaries (short)

| Boundary | What sits inside | What can go wrong outside |
|----------|------------------|---------------------------|
| Phone ↔ core | Attach and IP | Bad radio, wrong APN |
| Phone ↔ MCPTT apps | TLS to CMS, IdMS, SIP | Missing lab CA on the phone |
| P-CSCF ↔ PCRF | Call-time policy | No dedicated bearer → choppy MCPTT |
| App server ↔ media | Internal HTTP | Signalling works, no audio |

---

## When something breaks

| Failure | What you usually see |
|---------|----------------------|
| Database down | CMS / GMS / AS / admin errors |
| Media down | Call may ring; no talk / no floor |
| P-CSCF down | No SIP register or invite |
| IdMS down | Login fails |
| Missing UE route on media host | One-way or silent RTP |
| Policy path not firing | Call works but no MCPTT QCIs on the radio |

Next: [03-Operator-Runbook.md](03-Operator-Runbook.md).
