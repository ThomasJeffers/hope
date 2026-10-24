# 01 — MCPTT Master Handbook (Textbook)

> Progressive textbook for the **HPPCN_MCPTT** lab in this repository.  
> Evidence tags: see [README.md](README.md).  
> Companion diagrams: [02-System-Architecture-and-Call-Flows.md](02-System-Architecture-and-Call-Flows.md).

---

## Chapter 0 — How to use this book

You will see the same idea three times: **intuition → example → code**. When a claim is about *this* lab, a path appears (e.g. `custom_deployments/mcptt/services/mcxpy/mcx/as_server.py`). If you cannot open that path, the claim is outdated — fix the doc, not your memory.

**Acronym rule.** Every acronym is expanded at first use in a chapter. Glossary: [10-Practical-Labs-Glossary-and-Exam.md](10-Practical-Labs-Glossary-and-Exam.md).

---

## Chapter 1 — What problem MCPTT solves

### Intuition

Ordinary mobile voice is one-to-one (or conference bridges). Mission-critical push-to-talk (MCPTT) is closer to a **disciplined radio net**: many listeners, one talker at a time, fast push-to-talk, group affiliation, and priority for emergencies — over a cellular IP network instead of a dedicated LMR (Land Mobile Radio) channel.

### Example

Police ops channel “grp-ops”: Alice presses PTT; Bob and Carol hear her. If Bob presses while she is still talking, he **waits in line** — he does not shove her off the air just because his profile priority number is higher. Only an explicit **emergency** or **imminent peril** escalation may take the floor. That rule is **floor control** (who may speak). Separately, the **packet core** should carry Alice’s voice on a mission-critical radio class so the cell does not treat it like casual internet traffic — that is **network QoS** (chapter 6).

### Technical

3GPP Stage 1 requirements live in **TS 22.179**; architecture in **TS 23.179** / common MC services **TS 23.280**; on-network signalling **TS 24.379**; media-plane control **TS 24.380**. This lab implements a **Rel-17-oriented on-network** profile (see overlay README), not off-network IOPS.

### In this project (**Verified in source**)

| Concept | Where it lives |
|---------|----------------|
| Client | `custom_deployments/mcptt/android/HppcnMcptt/` package `lab.hppcn.mcptt` |
| Call control AS | `mcx/as_server.py` container `hppcn_mcx_as` |
| Floor + RTP | `mcx/media_server.py` + `mcx/mbcp.py` |
| User/group config | Postgres `mc_user` / `mc_group` + CMS/GMS XCAP |
| “Hub” | **Not** a 3GPP MCPTT entity — lab SSO iframe portal `hppcn_hub` |

---

## Chapter 2 — IP and service networking (lab-shaped)

### Intuition

Every container and every phone has addresses. Packets only arrive if **routing**, **DNS**, and **firewalls/NAT** agree. In this lab, MC servers must talk *to the UE’s PDN address*, so UPF NAT for that range is disabled.

### Example

UE gets `192.168.100.2`. Media server `172.29.0.206` must route `192.168.100.0/24` via UPF (`MCX_ROUTE_TO_UPF`, idms-route entrypoint). If that route is missing, SIP REGISTER may work (if hairpinned oddly) but RTP fails — classic “signalling OK, audio dead”.

### Concepts you need

| Term | Meaning here |
|------|----------------|
| Host IP | e.g. `DOCKER_HOST_IP=192.168.1.51` published ports |
| Docker bridge IP | e.g. `172.29.0.x` on compose network |
| UE IP | Allocated on `internet` DNN/APN |
| Advertised FQDN | `cms.mcx.hppcn.lab` via lab BIND `DNS_IP` |
| Listening socket | Process bound to 0.0.0.0:port inside container |
| NAT | Normally UPF SNATs UE→Internet; **MC exemption** `MCX_NO_NAT_IPV4` |

### Observe

```bash
docker exec hppcn_mcx_media ip route | head
docker exec hppcn_upf ip route | head   # name may vary — check `docker ps`
# From host: packet capture toward media ports
sudo tcpdump -ni any udp portrange 20000-20999 -c 20
```

---

## Chapter 3 — LTE/EPC fundamentals (enough to attach)

### Intuition

The phone (**UE**) + **SIM** prove identity to the **HSS** via the **MME**. The gateway (**SMF/UPF**, or older SGW/PGW names) gives the UE an IP on an **APN**. Until that default bearer exists, no MCPTT app traffic flows.

### Example (this lab)

PLMN **901/70**, APN **internet** (everyday data) + **ims** (VoLTE signalling). The MCPTT app’s HTTPS and media use the data path; VoLTE stays available for ordinary IMS audio. Provisioning: `custom_deployments/mcptt/provisioning/provision_mc.sh`.

### Key nodes (Open5GS layout in `deploy-all.yaml`)

| Function | Role in plain words |
|----------|---------------------|
| MME | Runs attach / mobility on 4G |
| HSS / PyHSS | Subscriber database (and IMS identities) |
| SMF / UPF | Creates the IP session and carries user packets |
| PCRF / PCF | Decides *which* QoS a session may get when asked |
| eNB | Radio (srsRAN / BladeRF in the E2E runbook) |

### Attach → IP (simplified)

1. UE camps on cell → Attach / PDN Connectivity.  
2. MME authenticates with HSS (EPS-AKA).  
3. Default bearer on APN `internet` → UE IP.  
4. DNS to lab BIND → resolve `*.mcx.hppcn.lab`.  

**Verified in runtime (lab practice):** `docs/mcx-e2e-runbook.md` expects `rmnet` in `192.168.100.0/24` and voice reg in service.

---

## Chapter 4 — 5G System (what differs)

### Intuition

5G SA replaces many Diameter/GTP-C control conversations with **HTTP/2 Service-Based Interfaces**. User plane is still tunnelled (N3 GTP-U) to a **UPF**. PDU Session ≈ old PDN connection; **DNN** ≈ APN; **5QI** ≈ QCI.

### Mapping (conceptual, not identical)

| 4G | 5G |
|----|-----|
| MME | AMF (+ some SMF) |
| HSS | UDM/AUSF/UDR |
| PGW-C/SMF-like | SMF |
| PGW-U | UPF |
| PCRF | PCF |
| QCI | 5QI |

### In this repo

Compose files for 5G SA exist (`sa-deploy.yaml`, `sa-vonr-*.yaml`). MCPTT addressing stays the same idea as on LTE. Open5GS PCF patches teach the core the MCPTT application identifiers used on the **N5** policy interface. The day-to-day BladeRF runbook still centres on **LTE**; treat full 5G radio + MCPTT as a path you prove when that radio is the one under test.

---

## Chapter 5 — IMS in one sitting

### Intuition

**IMS** (IP Multimedia Subsystem) is the SIP “phone system” sitting on the packet core. Phones **register**, then place **INVITE**s. Messages usually enter at the **P-CSCF** (Proxy Call Session Control Function), may pass other CSCFs, and reach an **application server** that implements the service logic.

### Example

- **VoLTE:** the phone registers to the P-CSCF; ordinary voice uses network class **QCI 1**.  
- **MCPTT in this lab:** the same P-CSCF is also the SIP front door for the Android app. When an MCPTT call starts, that P-CSCF is in the perfect place to tell the **PCRF** “open mission-critical bearers” (classes **65 / 67 / 69 / 70** as needed). Chapter 6 explains why that matters.

### In this lab today

| Piece | What it does for you |
|-------|----------------------|
| Kamailio P/I/S-CSCF | Classic IMS stack (VoLTE and friends) |
| `hppcn_pcscf` | First SIP hop for **both** VoLTE and MCPTT; can request MCPTT QoS from the core |
| `hppcn_mcx_as` | MCPTT application server — groups, invitations, media setup |

The app’s SIP host (`sip.mcx.hppcn.lab`) resolves to this **P-CSCF**. REGISTER uses a Bearer token the P-CSCF checks with the application server. There is no separate “MCPTT-only SIP edge” on the happy path anymore.

### SIP / SDP / RTP (minimum)

- **SIP** — text signalling: REGISTER, INVITE, BYE, …  
- **SDP** — lists IP/ports/codecs inside SIP.  
- **RTP** — the voice/video bytes. This lab uses G.711 A-law voice and H.264 video.  
- **MBCP** — floor control messages (who may talk), carried like RTCP.

---

## Chapter 6 — Making MCPTT traffic important on the radio

### Intuition

Painting the PTT button red does nothing to the cell scheduler. The **radio and core** only treat packets specially if they are mapped to a **bearer** (4G) or **QoS flow** (5G) with the right class (QCI / 5QI).

So “MCPTT quality” has two different meanings people mix up:

1. **Floor priority** — Alice is talking; Bob must wait (or queue). That is decided in the **media server**.  
2. **Network QoS** — Alice’s RTP should ride a mission-critical bearer so the cell does not starve it. That is decided by the **packet core**, when policy says so.

Both matter. Only (2) changes how the eNB/UPF schedule bits.

### The classes we care about

| Class | Use |
|------:|-----|
| **65** | MCPTT voice |
| **67** | MCVideo |
| **69** | MCPTT SIP signalling |
| **70** | MCPTT data (profiles, groups, files, …) |
| **1** | Normal VoLTE audio (keep this for ordinary IMS calls) |

### How the core learns “this call is MCPTT”

**Standards way (what we implement on Open5GS + Kamailio):**

1. Phone places MCPTT SIP through the **P-CSCF**.  
2. P-CSCF sends a policy request to **PCRF** on 4G (**Rx** interface), or to **PCF** on 5G (**N5**).  
3. Policy answers with the MCPTT QCIs/5QIs.  
4. **SMF** programs the **UPF** (and the radio gets matching dedicated bearers / QoS flows).

That loop is ordinary **3GPP policy and charging control**. We patch Open5GS and Kamailio so MCPTT application identifiers map to classes 65/67/69/70 — we are not replacing the core with a side script as the product story.

**Lab-only shortcut (not the product story):**

Pre-install filters in the core database (“MC Critical” / `mc-rules`) so some ports always look important. Useful when you need a quick radio check; **wrong** as the long-term architecture because it is not call-time, not IMS-aware, and does not scale like PCC.

### Where to look in this repository

| Topic | Location |
|-------|----------|
| P-CSCF MCPTT routes + Rx hooks | `pcscf/route/mcptt.cfg`, Kamailio ims_qos patches under `ims_base/patches/` |
| Open5GS PCRF/PCF MCPTT QCI mapping | `base/patches/open5gs/` |
| Decision record (engineers) | `docs/adr/ADR-011-dynamic-pcc-rx-n5.md` |
| Pictures and call flows | [02 — System Architecture](02-System-Architecture-and-Call-Flows.md) |

When you verify on air, you are looking for the core **creating** those dedicated bearers at call time — not for a new design.

---

## Chapter 7 — MCPTT concepts (standard → lab)

### 7.1 Identities

| Identity | Lab example | Where it lives |
|----------|-------------|----------------|
| MCPTT ID | `sip:alice@mcx.hppcn.lab` | `mc_user.mcptt_id` |
| IMSI bind | alice ↔ phone-1 IMSI | `mc_user_imsi`, `sim_bind` |
| IdMS user | Keycloak username `alice` | realm `hppcn-mcx` |
| PSI | `sip:mcptt-cf@mcx.hppcn.lab` | `config.CF_PSI` |

### 7.2 Affiliation vs membership

Think of three layers:

- **Membership** (GMS): who *may* belong to the group document.  
- **Affiliation** (phone PUBLISH + AS state): who is *currently* affiliated.  
- **Implicit affiliation (IA):** the CMS profile lists groups to auto-affiliate — Hub checkbox “IA” on a member (`mc_group_member.implicit_affiliation`).

Adding someone as a member without IA does **not** make their phone affiliate by itself.

### 7.3 Call types

| Type | How the user starts it | What the AS expects |
|------|------------------------|---------------------|
| Prearranged group voice | Home PTT / group call | Prearranged group invite |
| Private voice / video | Private call screen | Private call dialog |
| Group MCVideo | Video screen | Video leg on the group |
| Emergency group | Emergency control | User flag `allow_emergency_call` |
| Imminent peril | Peril control | Distinct from emergency; migration `07_imminent_peril.sql` |

### 7.4 Floor control

**Intuition.** The microphone token: only one talker at a time.  
**Messages.** Floor Request / Granted / Taken / Deny / Release / Idle / Revoke — `mbcp.py`, `Mbcp.kt`.  
**Arbitration.** Idle → grant; busy → queue; **preempt only** when the requester’s priority is ≥ 200 (imminent peril / emergency). Also: max talk (T2), hang timer, RTP inactivity ~6 s.

Bob’s ordinary PTT must **not** shove Alice off the air just because his profile number is a bit higher.

### 7.5 Media

Voice ports **20000–20499**; video **20500–20999**. Only the **current floor holder’s** RTP is fanned out to listeners (`on_rtp` / `on_vrtp`).

### 7.6 Features status (honest)

| Feature | Status |
|---------|--------|
| On-network group / private PTT | Implemented (source + acceptance tests) |
| MCVideo H.264 | Implemented (lab codecs; hardening continues) |
| MCData SDS / FD | Implemented |
| Presence soft-state | Implemented (AS registration TTL) |
| Emergency / imminent peril | Implemented with CMS authorization gates |
| Off-network / IOPS | **Not built** |
| Protected media (SRTP / GMK) | **Not built** (plain RTP) |
| Call-time MCPTT network QoS | Wired in this tree; confirm on radio with a Create-Bearer / QoS-flow trace |

---

## Chapter 8 — End-to-end journey (this APK)

Detail diagrams: doc 02. Stages in order:

1. **Install APK** — `tools/device/install_app.sh`.  
2. **Trust lab CA** — `pki/out/ca.crt`.  
3. **APN** — `provision_apn.sh`.  
4. **Cellular attach** — Open5GS + radio (outside the app).  
5. **Login** — IdMS token.  
6. **CMS profile** — groups, authorisations, N6, monitor mode.  
7. **SIP REGISTER** — phone → P-CSCF → AS registrations.  
8. **Affiliation PUBLISH** — IA list + what the user selects.  
9. **PTT / video** — INVITE with MCPTT / MCVideo XML + SDP.  
10. **AS** opens media participants and invites affiliated members.  
11. **Floor** (MBCP) + **RTP**.  
12. **Hangup** / hang timer / BYE.

---

## Chapter 9 — Who does what (cheat sheet)

| Component | Does | Does **not** |
|-----------|------|----------------|
| Android `Mcx` | Client state, SIP, media, UI | Cellular attach |
| P-CSCF | First SIP hop (VoLTE + MCPTT); TLS; token check; asks core for MCPTT QoS | Floor control or audio mix |
| AS (`mcx-as`) | Dialogs, group logic, media setup | Play audio on the phone |
| media | Floor + RTP / video relay | User provisioning |
| CMS / GMS | XCAP documents | Floor |
| IdMS | OAuth / OIDC tokens | SIP routing |
| Hub | Browser portal + proxy skin | MCPTT call logic |
| mcx-admin | Provisioning API + Control UI | Radio |
| Open5GS | Attach, IP, policy when asked | MCPTT XML |

---

## Chapter 10 — Mental models that prevent bugs

1. **Registered ≠ affiliated ≠ in call ≠ has floor.** Four different states.  
2. **Hub green “AS up” ≠ the phone hears audio.** Check floor holder + RTP counters.  
3. **Painting a QCI in a slide ≠ QoS on the air.** You need call-time PCC (or a working lab fallback) *and* the radio scheduler.  
4. **Emergency ≠ imminent peril.** Different CMS flags and floor indicators.  
5. **Membership ≠ IA.** Membership alone does not auto-affiliate.

---

## Chapter 11 — Knowledge checks (short)

1. Name the container that grants the floor.  
2. Which APN carries MCPTT HTTPS in this lab?  
3. Is `hppcn-hub` a 3GPP network function?  
4. Where is `allow_imminent_peril_call` stored, and how does the phone learn it?  
5. Why might Bob get Floor Granted while Alice sees no video?

*Answers:* (1) `hppcn_mcx_media`. (2) `internet`. (3) No — lab portal. (4) Postgres `mc_user` → CMS profile XML → APK after refresh. (5) TX not armed / no IDR / decoder stuck / bad remote video path — see doc 08.

---

## Next

Read [02 — System Architecture and Call Flows](02-System-Architecture-and-Call-Flows.md) for Mermaid diagrams and sequenced flows with file citations.
