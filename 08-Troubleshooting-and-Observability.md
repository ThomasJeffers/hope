# 08 — Troubleshooting and Observability

When a demo fails, the instinct is “reinstall the APK.” Resist that. Walk the **layers** until the first one that fails, then gather evidence that distinguishes causes.

---

## 1. Master diagnostic ladder

1. Is the container **running**? (`docker ps`)  
2. Is it **ready**? (`/healthz`, DB `pg_isready`)  
3. Are **dependencies** up (db, IdMS, media for AS)?  
4. Is the **phone attached** (MME logs, `device_info.sh`)?  
5. Does the phone have the **expected IP** (`192.168.100.0/24` in this lab)?  
6. Does **DNS** resolve `cms.` / `sip.` / `idms.` from the phone?  
7. Does **TLS** to CMS/IdMS work? Does **SIP TLS** to the P-CSCF work?  
8. Did **login** obtain an IdMS token?  
9. Did **SIP REGISTER** return 200? (logcat `MCX`, AS registrations)  
10. Was **affiliation** accepted?  
11. Did **INVITE** succeed on both sides?  
12. Did **floor** Grant / Taken appear in media logs?  
13. Are **RTP** counters rising (phone + media `rx`/`tx`)?  
14. Does the phone **play** audio / show video (focus, surface)?  

Stop at the first failing layer.

---

## 2. Where the logs live

| Component | How to read it |
|-----------|----------------|
| AS | `docker logs hppcn_mcx_as` |
| Media | `docker logs hppcn_mcx_media` |
| P-CSCF (MCPTT SIP) | `docker logs hppcn_mcx_sip_edge` |
| CMS / GMS | `docker logs hppcn_mcx_cms` / `_gms` |
| Hub | `docker logs hppcn_hub` |
| Admin | `docker logs hppcn_mcx_admin` |
| MME / SMF / UPF | `docker logs hppcn_mme` (names may vary — check `docker ps`) |
| Phone | `adb logcat -s MCX` |
| Acceptance | `custom_deployments/mcptt/reports/mcx-acceptance.json` |

---

## 3. Playbooks

### 3.1 Container will not stay up

**Symptom.** Exit / restart loop.  
**Gather.** `docker ps -a`, `docker logs <name>`, health status.  
**Common causes.** Missing `.mcptt_env`, missing PKI files, host port already bound, database not healthy yet.  
**Verify.** `/healthz` returns success after the fix.

### 3.2 Phone will not attach

**Symptom.** No service bar / no `rmnet`.  
**Layer.** Radio + MME + HSS keys.  
**Gather.** srsenb logs, MME auth failures, IMSI / OPC mismatch.  
⚠ Do not dump Ki / OPc into chats.

### 3.3 Attached, but the app cannot reach servers

**Gather.** UE IP; resolve and curl CMS FQDN; UPF routes; `MCX_NO_NAT`; BIND.  
**Classics.** Phone DNS is not lab BIND; NAT still on for the UE prefix; wrong APN (IMS-only when you needed `internet`).

### 3.4 Login fails

**Gather.** IdMS logs, realm name, clock skew, whether the lab CA is trusted on the device.  
**SIM login.** Admin `ue-by-ip` mapping + CMS sim-login path.

### 3.5 Registered but cannot call

**Gather.** Affiliation PUBLISH/NOTIFY, selected group, AS rejection reasons (not a member / not authorised for emergency or peril).  
**Code home.** Authorization branches in `as_server.py`.

### 3.6 Floor never granted

**Gather.** Media logs for Floor Request; current holder; priority; queue.  
**Client.** Is PTT held? Is the selected group the same as the video leg?  
**Server.** Does a media session exist? Did the participant join with a remote address?

### 3.7 One-way or no audio

**Gather.** Media `rx`/`tx` per participant; phone RTP counters; `tcpdump` UDP 20000–20999; routes from media toward the UE.  
**Distinguish.** No uplink vs no fan-out vs player muted (monitor / presentation gains).

### 3.8 Video: A sees B, but B’s transmit never appears on A

**Gather.** `docker logs hppcn_mcx_media | grep video` — is Bob’s stream `relayed=`?  
- If **no packets from Bob:** client TX not armed / no IDR / remote video path null.  
- If **relayed ≥ 1:** Alice decoder or surface — check `V tx/rx` on both UIs.  
**Known hardening areas.** Decoder reset on talker change; arm PTT for video; floor TX while video-focused.

### 3.9 Hub iframe blank

**Gather.** Logged into Hub? Does `/go/mcptt/` return 200? Admin TLS trusted? Browser network tab.  
Accessibility tools often cannot see inside the iframe — use Network / curl with a session cookie.

### 3.10 QoS / QCI not what you expected

**Ask first.** Are you expecting **call-time** P-CSCF → PCRF/PCF, or the old **static** `mc-rules` shortcut?  
**Gather.** PCRF / PCF logs for AF application identifiers; Create-Bearer / QoS-flow messages on air.  
**Product story.** Dynamic PCC (see textbook chapter 6). Static Mongo TFT is fallback only.

---

## 4. Useful read-only commands

```bash
curl -s http://172.29.0.206:8081/v1/sessions | python3 -m json.tool | head
curl -sk https://127.0.0.1:18444/v1/live | python3 -m json.tool | head
docker logs hppcn_mcx_media 2>&1 | grep -iE 'granted|video|floor' | tail -40
adb logcat -s MCX | grep -iE 'floor|video|REGISTER|INVITE'
```

---

## 5. Observability extras

- **Grafana** via Hub → Ops — lab exporter (`mcx-exporter` / `lab_exporter.py`).  
- **Admin Live** tab — human-friendly AS/media view during a demo.  
- Health endpoints prove **process liveness**, not full dependency health.

Next: [09 — Security and reliability](09-Security-and-Reliability.md).
