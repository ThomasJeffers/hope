# 07 — Protocols, APIs, Ports, and Data Models

This chapter is the **phone book**: which port, which protocol, which JSON path.  
Use it when something “should connect” but does not — or when you need to know whether an interface is 3GPP-shaped or a project REST helper.

---

## 1. Host-published ports (typical lab)

| Port | Service | Protocol |
|------|---------|----------|
| 18100 | hppcn-hub | HTTP |
| 18444 | mcx-admin | HTTPS |
| 18443 | mcx-idms | HTTPS (Keycloak) |
| 18083 | ocs-ui (overlay) | HTTP |
| IMS / VoLTE ports | per `deploy-all.yaml` | SIP / RTP |

The **phone** usually reaches MCPTT by **FQDN on Docker IPs** (lab DNS), not by typing these host ports.

---

## 2. In-network MCPTT addresses

| Container | IP | Ports that matter |
|-----------|-----|-------------------|
| mcx-db | 172.29.0.200 | 5432 |
| mcx-idms | 172.29.0.201 | 8443 |
| mcx-cms | 172.29.0.202 | 443 |
| mcx-gms | 172.29.0.203 | 443 |
| P-CSCF (MCPTT SIP front door) | 172.29.0.204 | 5061 TLS, 5060 |
| mcx-as | 172.29.0.205 | 5060 SIP, **8080** HTTP |
| mcx-media | 172.29.0.206 | **20000–20999** UDP, **8081** HTTP |
| mcx-admin | 172.29.0.207 | 443 |
| mcx-fd | 172.29.0.213 | 443 |

(IPs come from overlay / `.env`; confirm on your host if you customized them.)

---

## 3. Protocols by path (story form)

| Path | Protocol | 3GPP-shaped? |
|------|----------|--------------|
| Phone → IdMS | HTTPS OAuth2 / OIDC | IdMS role (TS 24.482 family) |
| Phone → CMS / GMS | HTTPS XCAP | TS 24.484 / 24.481 family |
| Phone → P-CSCF | SIP over TLS | TS 24.379 family |
| P-CSCF → AS | SIP (UDP/TCP as configured) | Project routing on IMS |
| AS ↔ media | HTTP JSON on `:8081` | **Project** control plane |
| Phone ↔ media | RTP + MBCP | Floor ≈ TS 24.380; codecs are lab choices |
| Hub → admin | HTTP reverse proxy | **Project** |
| P-CSCF → PCRF | Diameter **Rx** | 3GPP PCC |
| AF → PCF (5G) | **N5** | 3GPP PCC |

---

## 4. Application server HTTP (`as_server.py`)

| Method | Path | Role in plain words |
|--------|------|---------------------|
| POST | `/v1/auth/introspect` | P-CSCF asks “is this token still good?” |
| POST | `/v1/registrations` | Side-effect of REGISTER / presence |
| POST | `/v1/gms/changed` | GMS tells AS a group document changed |
| POST | `/v1/media/events` | Media reports hang-timer and friends |
| GET | `/v1/status` | Ops snapshot |
| GET | `/v1/presence` | Who is registered |
| GET | `/healthz` | Process alive? |

---

## 5. Media HTTP (`media_server.py`)

| Method | Path | Role |
|--------|------|------|
| POST | `/v1/sessions` | Create a call’s media session |
| GET | `/v1/sessions` | List sessions |
| PATCH | `/v1/sessions/{sid}` | Flags (emergency / peril) |
| DELETE | `/v1/sessions/{sid}` | Tear down |
| POST | `/v1/sessions/{sid}/participants` | Allocate ports for a participant |
| PUT | `…/participants/{pid}` | Join + remote address |
| DELETE | `…/participants/{pid}` | Leave |

Floor arbitration itself happens on the **UDP** floor channel, not on these HTTP verbs.

---

## 6. Admin HTTP (`admin.py`)

| Method | Path | Role |
|--------|------|------|
| GET | `/` | Control UI |
| GET | `/v1/status` | Domain counts |
| GET | `/v1/live` | AS + media snapshot for Live tab |
| GET/POST/PATCH/DELETE | `/v1/users…` | Provision people |
| POST | `/v1/users/{uid}/password` | IdMS password ops |
| PUT | `/v1/users/{uid}/private-contacts` | PrivateCallList |
| GET/POST/PATCH/DELETE | `/v1/groups…` | Groups |
| POST/PATCH/DELETE | `/v1/groups/{gid}/members…` | Membership / IA |
| GET | `/v1/audit` | Audit trail |
| POST | `/v1/ues` | UE provision helper |
| GET | `/v1/ue-by-ip/{ip}` | SIM login: map UE IP → IMSI |

---

## 7. SIP multipart content-types (`config.py`)

- `application/sdp`  
- `application/vnd.3gpp.mcptt-info+xml`  
- `application/vnd.3gpp.mcvideo-info+xml`  
- `application/vnd.3gpp.mcdata-signalling+xml`  

ICSIs: `ICSI_MCPTT` / `MCVIDEO` / `MCDATA`.

---

## 8. Postgres tables (migrations)

| Table / change | Introduced | Role |
|----------------|------------|------|
| mc_org, mc_user, mc_group, mc_group_member | 01_schema | Core directory |
| mc_user_imsi | 01 / 03 | SIM binding |
| mc_service_config, mc_audit | 01 | Config / audit |
| mc_sds_message, mc_fd_object | 04 | MCData |
| mc_private_contact | 05 | PrivateCallList |
| lab_password column | 06 | Display / reset helper |
| imminent peril columns | 07 | Peril authorization |

Remember: migrations run only on **first** volume init — see runbook §10b.

---

## 9. Floor messages (MBCP)

Encoded in `mbcp.py` / `Mbcp.kt`: Floor Request, Granted, Taken, Deny, Release, Idle, Revoke, Ack, queue fields, priority, user id, and floor indicator bits (normal / emergency / imminent peril in this lab’s mapping).

---

## 10. Android media classes

| Class | Codec | Ports |
|-------|-------|-------|
| CallMedia | PCMA (G.711 A-law) payload type 8, 8 kHz | RTP base; floor at base+2 |
| CallVideo | H.264 payload type 96, 320×240 | Separate video socket |

Next: [08 — Troubleshooting](08-Troubleshooting-and-Observability.md).
