# 06 — 3GPP Standards and Traceability

This chapter answers a question bosses ask early: **“Are we 3GPP compliant?”**

Honest short answer: this lab implements a **Rel-17-oriented on-network** MCPTT / MCVideo / MCData profile with Open5GS + Kamailio IMS and **call-time policy control** (P-CSCF → PCRF on 4G, PCF on 5G). That is a serious architecture. It is **not** a stamped conformance certificate. Spec titles below are the families the project cites; **clause-by-clause “shall” audits against official PDF text were not re-run in this documentation pass**.

Official portal: https://www.3gpp.org/specifications  

Machine matrix: [06-Standards-Traceability.csv](06-Standards-Traceability.csv).

---

## 1. What this lab claims (and where the claim lives)

| Claim | Where you can see the intent |
|-------|------------------------------|
| Rel-17-oriented on-network MC services | `custom_deployments/mcptt/README.md` |
| Signalling shaped like TS 24.379; floor like TS 24.380 | `mbcp.py`, Android `Mbcp.kt`, `as_server.py` |
| CMS / GMS / IdMS roles (24.484 / 24.481 / 24.482) | `cms.py`, `gms.py`, Keycloak realm |
| MCVideo / MCData lab paths | `CallVideo`, media video ports, `fd_server`, DataActivity |
| Common architecture families (23.280 / 23.179) | ADR series under `docs/adr/` |
| Stage 1 requirements family (22.179) | Educational reports / ADRs |

---

## 2. How to study a specification without drowning

For any spec you open, walk this ladder:

1. **Problem** the industry was solving  
2. **Why** mission-critical needs its own rules  
3. **Big idea** (entities + procedures)  
4. **Entities** (server, client, CMS, …)  
5. **Procedures** (register, affiliate, call, floor)  
6. **Messages** (SIP bodies, MBCP, XCAP)  
7. **shall / should / may** language  
8. **This lab** — what we actually run  
9. **Path** in the repo  
10. **Failure** modes if you skip a step  
11. **Test** you can run  
12. **Evidence limits** — what you still have not proven on air  

### A. Service and architecture

- **TS 22.179** — what operators expect (latency, availability, priority classes).  
- **TS 23.179 / 23.280** — functional entities. *This lab collapses many boxes into `mcx-as` + CMS/GMS/IdMS/media.*

### B. Protocols

- **TS 24.379** — SIP/XML MCPTT signalling (`mcptt-info`, PSI, affiliation).  
- **TS 24.380** — floor messages (our MBCP encoder in `mbcp.py`).

### C. Security

- **TS 33.179 / 33.180** — MC security. Lab uses TLS + OAuth + SIM bind. **Not** the full media-security suite (no SRTP / GMK here).

### D. IMS and packet core

- **TS 23.228 / 24.229** — IMS / SIP.  
- **TS 23.401 / 23.501 / 23.502 / 23.503** — EPS / 5GS + policy.  
  Dynamic **Rx** (4G) / **N5** (5G) is the intended path for mission-critical bearers (classes 65 / 67 / 69 / 70).

---

## 3. What the standard leaves free (and what we chose)

| Topic | Standardized? | Our lab choice |
|-------|---------------|----------------|
| Separate OS process per MC network function | Not required | One `mcxpy` image; `MCX_ROLE` picks the job |
| Hub portal | **Not** a 3GPP entity | Custom `hppcn_hub` |
| Admin REST `/v1/users` | Product API | Custom |
| Where the phone sends MCPTT SIP | Product choice | Straight to **P-CSCF** (same IMS front door as VoLTE). Call-time QoS uses P-CSCF → PCRF/PCF |
| Static Mongo TFT preload (`mc-rules`) | Not the 3GPP call-time model | Dev fallback only |
| Floor inactivity 6 s | Timer values often implementation-defined | `RTP_INACTIVITY_S` |
| G.711 A-law voice / H.264 320×240 | Codec policy | Lab choices in `CallMedia` / `CallVideo` |
| Imminent peril vs emergency | Distinct in the architecture/signalling families | Separate CMS flags + MBCP indicator bits |

---

## 4. Open5GS / Kamailio patches (standards-adjacent)

| Patch area | Intent | Spec relationship |
|------------|--------|-------------------|
| Open5GS 0001… | Normalize QCI/5QI resource types | Correct GBR / non-GBR treatment |
| Open5GS 0002… | UPF QER gate / MBR enforce | User-plane QoS enforcement |
| Open5GS 0003… | AF-App-Id → QCI 65/67/69/70 | Dynamic PCC families (23.203 / 23.503) |
| Kamailio 0001… | Rx AF-Application-Identifier from MCPTT ICSI | AF behaviour toward PCRF |

These support **standardized PCC ideas**. They are still **local patches** — treat upgrades carefully.

---

## 5. How to strengthen evidence later (when the boss asks for proof)

1. Freeze a release (e.g. Rel-17) and the PDF versions.  
2. Extract *shall*s for on-network group call + floor + CMS authorization only.  
3. Map each to a PCAP + log + `uesim` test case.  
4. Update the CSV `assessment` / `test_evidence` columns.  
5. Do **not** mark the system “compliant” until that campaign finishes.

---

## 6. Related project docs

- `docs/adr/ADR-001` … `ADR-011`  
- `docs/mcptt-standards-used-in-chats.md`  
- `docs/hppcn-mcptt-educational-report-EN.md`

Next: [07 — Protocols, APIs, ports](07-Protocols-APIs-Ports-and-Data-Models.md).
