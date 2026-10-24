# 11 — Source Inventory Appendix

This appendix is the **full phone book** of the tree: services, IPs, modules, patches.  
Prefer it when you need every container name — not when you are learning the story (start with README → 01 → 02).

**Evidence style.** Paths and compose names from a read of this repository. Whether a container is *running on your host right now* is **unverified** until you check `docker ps`.

Launch:

```bash
docker compose -f deploy-all.yaml -f custom_deployments/mcptt/mcptt-overlay.yaml up -d
```

Optional project name used in lab work: `--project-name hppcn_mcptt`.

---

## 1. Top-level layout

| Path | Role |
|------|------|
| `deploy-all.yaml` | Open5GS 4G/5G + Kamailio IMS + DNS / MySQL / PyHSS / RTPEngine / OCS / metrics / Grafana |
| `custom_deployments/mcptt/mcptt-overlay.yaml` | MCPTT overlay (services, IPs, Hub) |
| `base/` | Open5GS image + `base/patches/open5gs/` |
| `ims_base/` | Kamailio image + `ims_base/patches/kamailio/` |
| `custom_deployments/mcptt/` | Product overlay: `android/`, `db/`, `idms/`, `pki/`, `provisioning/`, `services/`, `tools/`, `prepare.sh` |
| Other | `4g-volte-*.yaml`, `sa-*.yaml`, RAN stubs; overlays `open5gs_hss_cx/`, `slicing/`, `with_eupf/` |

---

## 2. MCPTT overlay services

IPs from `.env` / README (subnet typically `172.29.0.0/24`).

| Service | container_name | Image | IP | Host ports | Role in plain words |
|---------|----------------|-------|-----|------------|---------------------|
| mcx-db | hppcn_mcx_db | postgres:16-alpine | 172.29.0.200 | — | MCPTT directory database |
| mcx-idms | hppcn_mcx_idms | keycloak:26.0 | 172.29.0.201 | 18443→8443 | Identity (login tokens) |
| mcx-idms-route | hppcn_mcx_idms_route | hppcn_mcptt_mcxpy | (IdMS netns) | — | UE route helper + IdMS setup |
| mcx-cms | hppcn_mcx_cms | hppcn_mcptt_mcxpy | 172.29.0.202 | — | User profile (CMS) |
| mcx-gms | hppcn_mcx_gms | hppcn_mcptt_mcxpy | 172.29.0.203 | — | Group documents (GMS) |
| P-CSCF (MCPTT SIP) | hppcn_mcx_sip_edge | hppcn_mcptt_kamailio | 172.29.0.204 | — | First SIP hop; can request MCPTT QoS |
| mcx-as | hppcn_mcx_as | hppcn_mcptt_mcxpy | 172.29.0.205 | — | Application server (call logic) |
| mcx-media | hppcn_mcx_media | hppcn_mcptt_mcxpy | 172.29.0.206 | UDP 20000–20999 | Floor + RTP / video |
| mcx-admin | hppcn_mcx_admin | hppcn_mcptt_mcxpy | 172.29.0.207 | 18444→443 | Control UI / API |
| mcx-exporter | hppcn_lab_exporter | hppcn_mcptt_mcxpy | 172.29.0.208 | — | Prometheus exporter |
| speedtest | hppcn_speedtest | hppcn_mcptt_mcxpy | 172.29.0.209 | 18088→80 | Throughput lab |
| mcx-uesim | hppcn_mcx_uesim | hppcn_mcptt_mcxpy | 172.29.0.210 | — | Acceptance (`profile: test`) |
| ocs-ui | hppcn_ocs_ui | hppcn_mcptt_ocs_ui | 172.29.0.211 | 18083→80 | OCS form login |
| hppcn-hub | hppcn_hub | hppcn_mcptt_hub | 172.29.0.212 | 18100→80 | Lab operator portal |
| mcx-fd | hppcn_mcx_fd | hppcn_mcptt_mcxpy | 172.29.0.213 | — | File distribution |

Volumes: often `hppcn_mcptt_mcxdbdata`, `hppcn_mcptt_mcxfddata` (names depend on project name).

**DB migrations note.** SQL under `db/` runs only on **first** Postgres volume init. Existing volumes do **not** auto-apply later files — apply manually or recreate (⚠ data loss). See runbook §10b.

---

## 3. Core / IMS (`deploy-all`) — abbreviated

Open5GS NFs (`hppcn_mme`, `hppcn_upf`, `hppcn_pcrf`, `hppcn_pcf`, …), Kamailio `hppcn_pcscf` / `icscf` / `scscf`, `hppcn_dns`, `hppcn_mysql`, `hppcn_pyhss`, `hppcn_rtpengine`, `hppcn_ocs`, `hppcn_grafana`, `hppcn_webui`, optional `osmomsc` / `osmohlr`. Full IP key names live in repo `.env` (no secret values in this handbook).

---

## 4. mcxpy modules

Entrypoint: `python -m mcx.${MCX_ROLE}` (`services/mcxpy/entrypoint.sh`).

| Module | Purpose |
|--------|---------|
| admin.py / admin_console.html | Provisioning API + Control UI |
| as_server.py | MCPTT application server |
| cms.py / gms.py | CMS / GMS XCAP |
| media_server.py / mbcp.py | Floor + RTP; MBCP codec |
| fd_server.py | File distribution |
| mcdocs.py / xmldocs.py | SDP + MC XML |
| sip.py | Shared SIP stack |
| idauth.py / idms_setup.py / idms_users.py | OIDC + Keycloak helpers |
| simbind.py / ue_locator.py | SIM bind + IP→IMSI |
| lab_exporter.py / speedtest.py / uesim.py | Ops / lab / acceptance |
| config.py / db.py / httpcommon.py | Shared config, Postgres, HTTP auth |

---

## 5. Hub

`services/hppcn_hub/hub.py` + `skin.css`: cookie login, `/go/<app>/` proxy, FA/EN fonts, skin inject. Proxied apps: MCPTT, Charging, Grafana, WebUI, IdMS, Speed, PyHSS (`HUB_UP_*`).

---

## 6. Android

- Project: `custom_deployments/mcptt/android/HppcnMcptt/`  
- Package: `lab.hppcn.mcptt`  
- Activities: Login, Ptt, Groups, Gmc, PrivateCall, Video, Data, Settings, Logs, Debug; `PttReceiver`  
- Sibling: `android/SmsBurst/` (`lab.hppcn.smsburst`)

---

## 7. Patches (QoS-related)

| Tree | Patch | Intent in plain words |
|------|-------|------------------------|
| `base/patches/open5gs/` | 0001 | Normalize QCI/5QI resource types |
| | 0002 | UPF enforces gate / MBR |
| | 0003 | Map AF application id → QCI 65/67/69/70 |
| `ims_base/patches/kamailio/` | 0001 | Put MCPTT AF-Application-Identifier on Rx toward PCRF |

Policy notes for engineers: `docs/adr/` (especially dynamic PCC).

---

## 8. SIP / IMS config homes

| Role | Path |
|------|------|
| MCPTT routes at P-CSCF | `pcscf/route/mcptt.cfg` (+ lab PKI TLS) |
| P / I / S-CSCF | `pcscf/`, `icscf/`, `scscf/` |
| SMSC | `smsc/` |
| DNS | `dns/` |

MCPTT SIP for the phone terminates at the **P-CSCF**; the application server trusts CSCF / P-CSCF. There is no separate MCPTT-only SIP edge on the happy path.

---

## 9. DB migrations

`01_schema` → `07_imminent_peril` (see doc 07 data models). Seed users in `02_seed.sql`.

---

## 10. Env key classes (no values)

- **Infra / IPs / ports:** repo `.env` (`DOCKER_HOST_IP`, `MCX_*_IP`, `MCX_*_PORT_*`, `UE_IPV4_INTERNET`, `UPF_IP`, `DNS_IP`, `HPPCN_*_HOST_PORT`, phone serials, …).  
- **Secrets:** `custom_deployments/mcptt/.mcptt_env` — `MCX_DB_PASSWORD`, `MCX_IDMS_ADMIN_PASSWORD`, `MCX_AS_CLIENT_SECRET`, `MCX_SIM_CLIENT_SECRET`, `MCX_PW_*` only.

Readable catalogue with meanings: [05-Configuration-and-Variable-Catalogue.md](05-Configuration-and-Variable-Catalogue.md).

---

## Back to the story

Return to [README](README.md) for reading order, or [02 Architecture](02-System-Architecture-and-Call-Flows.md) when a call flow is unclear.
