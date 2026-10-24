# 10 — Practical Labs, Glossary, and Exam

Hands-on modules for a dedicated lab host. Prefer **read-only** exercises first. Steps marked ⚠ can disrupt demos.

---

## Module L0 — Map the repositories

**Goal.** Find the overlay, the patches, the APK, and the decision records.  
**Do.** `ls`, open `custom_deployments/mcptt/README.md`, skim `docs/adr/`.  
**Exercise.** Draw your own box diagram; compare to doc 02 Diagram A.  
**Mastery.** Name five containers and their IPs from memory (inventory appendix / README table).

---

## Module L1 — Containers and health

```bash
docker ps --format '{{.Names}} {{.Status}}' | sort
curl -s http://127.0.0.1:18100/healthz
curl -sk https://127.0.0.1:18444/v1/status
```

**Expect.** Hub ok; Admin JSON with a `domain`.  
**Quiz.** What is the difference between Hub health and media health?

---

## Module L2 — IP topology

**Inspect.** `mcptt-overlay.yaml` `ipv4_address` lines; UPF no-NAT notes.  
**Exercise.** From `hppcn_mcx_media`, show the route to `192.168.100.0/24`.  
**Mastery.** Explain in one paragraph why NAT on that path kills MBCP/RTP return traffic.

---

## Module L3 — Phone attach path

**Use.** `docs/mcx-e2e-runbook.md` + `device_info.sh`.  
**Observe.** IMSI, APN, `rmnet` IP.  
**Quiz.** Which APN carries MCPTT HTTPS in this lab?

---

## Module L4 — Login and CMS profile

**Inspect.** `Http.kt`, `Mcx.applyProfile`; fetch CMS XML (docker exec or phone capture).  
**Exercise.** Toggle imminent peril in Admin; prove the XML field flips; note that the phone must refresh.  
**Mastery.** Trace `allow_imminent_peril_call` from DB → CMS → APK.

---

## Module L5 — Group call + floor

**Demo.** Two phones on `grp-ops`, PTT.  
**Evidence.** `docker logs hppcn_mcx_media | grep -i granted`.  
**Challenge.** Predict Taken vs Granted on each phone *before* you look at the UI.

---

## Module L6 — Media packets

⚠ `tcpdump` on the lab host only.  
**Exercise.** Capture UDP during PTT; match ports to the media session JSON.  
**Mastery.** Explain “only the floor holder’s RTP is fanned out” using `on_rtp` in the media server.

---

## Module L7 — Hub setting → runtime

**Exercise.** Change a display name under Users; find the audit row; confirm the DB.  
**Code homes.** `admin.py` PATCH user; `admin_console.html` `patchUser`.

---

## Module L8 — Config source of truth

**Exercise.** Pick `MCX_SIP_EDGE_URI`; find compose env, `config.py`, and the Android consumer.  
**Quiz.** What happens if compose and the phone settings disagree?  
**Note.** In this lab that URI points at the **P-CSCF** (IMS front door).

---

## Module L9 — Standards vs freedom

**Read.** Doc 06 freedom table + textbook chapter 6 (QoS).  
**Exercise.** Write ten lines: “Hub is / is not standardized because…”.  
**Mastery.** Orally defend why city-wide static TFT preload is rejected as the product story.

---

## Module L10 — ⚠ Simulated failure (disposable lab only)

1. `docker stop hppcn_mcx_media` → attempt PTT → observe.  
2. Start media again → a **new** call is required.  
3. Write an incident note: symptom → layer → evidence → fix → verify.

---

## Glossary (selected)

| Term | Expansion | Role here |
|------|-----------|-----------|
| MCPTT | Mission Critical Push To Talk | The service under study |
| MBCP | Floor-control messages (lab RTCP APP) | Who may talk |
| CMS | Configuration Management Server | User profile over XCAP |
| GMS | Group Management Server | Group documents over XCAP |
| IdMS | Identity Management Server | Keycloak OIDC in this lab |
| PSI | Public Service Identity | `mcptt-cf` / `mcptt-pf` URIs |
| ICSI | IMS Communication Service Identifier | Feature tagging on SIP |
| QCI / 5QI | QoS Class Identifier | Bearer / flow class on the radio |
| PCC | Policy and Charging Control | PCRF / PCF decisions |
| Rx / N5 | AF↔PCRF / AF↔PCF | Call-time QoS interfaces |
| AS | Application Server | `mcx-as` call logic |
| Hub | HPPCN Hub | **Custom** portal — not a 3GPP NF |
| IA | Implicit Affiliation | Auto-affiliate list in the CMS profile |
| N6 | Multi simultaneous calls param | Lab multi-receive cap |
| SDP | Session Description Protocol | Ports and codecs inside SIP |
| RTP | Real-time Transport Protocol | Audio / video bytes |

---

## Question bank

### Easy

1. Which container grants the floor?  
2. Default Hub port?  
3. Is media SRTP in this lab?

### Intermediate

4. Sequence from PTT down to the peer hearing audio (five steps).  
5. Where is `sim_bind` enforced?  
6. How do emergency and imminent peril differ *in this code*?

### Advanced

7. Why does proving Create-Bearer on air require looking at the **P-CSCF → PCRF** path, not Hub?  
8. How would you prove Bob’s video reached Alice at each hop?  
9. Name three implementation-defined timers or codecs.

### Oral scenarios

- “Boss: are we 3GPP compliant?” — answer with evidence levels (README tags).  
- “One-way audio after UPF restart” — layered debug aloud.  
- “Can we delete the Hub?” — what breaks operationally vs what the standard requires.

### Sample answers (brief)

1. `hppcn_mcx_media`. 2. 18100. 3. No.  
4. PTT down → MBCP Request → Grant → RTP uplink → fan-out → decode.  
5. CMS / sim-login + AS registration path using IMSI mapping.  
6. Separate CMS flags; AS gates; MBCP indicator bits; peril cancel authz.  
7. Rx fires at the P-CSCF when SIP/SDP for the call is seen; Hub never asks the PCRF.  
8. Bob TX counter → media video relayed log → Alice RX counter → surface.  
9. RTP inactivity ~6 s; hang / T2 from group; PCMA / H.264 lab choices.

---

## Final practical assessment

**Incident.** Alice and Bob show Registered. Alice PTT shows GRANTED. Bob hears silence. Hub Live shows Alice as holder with rising Alice `tx`, Bob `rx=0`.

**Deliverable (written):**

1. Hypotheses ranked.  
2. Exact read-only commands / logs you would collect.  
3. Most likely layer.  
4. Fix + verification.  
5. What you would *not* do first.

**Passing bar.** Correctly suspects the path Bob ← media (route / NAT / remote RTP) before blaming IdMS; cites media session JSON fields.

---

## Keeping this handbook honest

When you change AS / media / APK / QoS patches, update: doc 02 flows, doc 05 CSV rows, doc 06 CSV assessments, and this lab’s expected observations.

Next: [11 — Source inventory appendix](11-Source-Inventory-Appendix.md).
