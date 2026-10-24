# 05 — Configuration and Variable Catalogue

This chapter answers: **“Where do I change X, and what will break if I get it wrong?”**

Machine-readable twin: [05-Configuration-and-Variable-Catalogue.csv](05-Configuration-and-Variable-Catalogue.csv).  
**No secret values** appear here — only names, locations, and safe examples.

How the list was built: grep of `MCX_*` / `HUB_*` / `HPPCN_*` under `custom_deployments/mcptt`, plus `mcx/config.py`, overlay YAML, media port envs, and Hub. Not every Open5GS YAML key is duplicated — see `mme/`, `smf/`, `upf/` for core QoS knobs.

---

## How to read a row

| Column idea | Meaning |
|-------------|---------|
| Name | Environment variable or DB field |
| Location | File or container where it is set |
| Meaning | What a human should understand |
| Safe example | Non-secret sample |
| Risk | What goes wrong if it is wrong |

---

## 1. Identity and domain

| Name | Location | Meaning | Safe example |
|------|----------|---------|--------------|
| `MCX_DOMAIN` | `config.py`, overlay | DNS suffix for all MC hostnames | `mcx.hppcn.lab` |
| `MCX_IDMS_REALM` | `config.py` | Keycloak realm name | `hppcn-mcx` |
| `MCX_IDMS_URL` | `config.py` | Where phones log in | `https://idms.mcx.hppcn.lab:8443` |
| `CLIENT_ID` | `config.py` | OIDC client id | `mcptt-client` |
| `AUDIENCE` | `config.py` | Token audience | `mcx-services` |

Wrong domain → every FQDN the phone uses is wrong.

---

## 2. Service discovery roots

| Name | Meaning in plain words |
|------|------------------------|
| `MCX_CMS_ROOT` | Where the phone downloads its user profile (XCAP) |
| `MCX_GMS_ROOT` | Where group documents live (XCAP) |
| `MCX_SIP_EDGE_URI` | SIP URI the phone dials for REGISTER / INVITE — in this lab that is the **P-CSCF** (`sip.mcx.hppcn.lab`), not a separate MCPTT-only edge |
| `MCX_FD_ROOT` / `MCX_FD_STORE` | File-distribution URL / disk path |
| `MCX_CA_FILE` | Lab CA path inside containers (`/pki/ca.crt`) |

---

## 3. Addressing / Docker IPs

| Name | Typical role |
|------|----------------|
| `DOCKER_HOST_IP` | Address you type in a browser for Hub / Grafana |
| `MCX_DB_IP` … `MCX_ADMIN_IP` | Fixed IPv4s on the compose network (see inventory appendix) |
| `MCX_AS_IP` / `MCX_MEDIA_IP` | AS ↔ media HTTP |
| `DNS_IP` | Lab BIND for phones and containers |
| `UPF_IP` / `UE_IPV4_INTERNET` | How media finds the phone’s IP range |
| `MCX_NO_NAT_IPV4` | Keep UE MCPTT traffic **without** UPF NAT — critical for RTP return path |
| `MCX_ROUTE_TO_UPF` | Install routes so MC containers reach UE prefixes |

---

## 4. Media ports

| Name | Default idea | Effect |
|------|--------------|--------|
| `MCX_MEDIA_PORT_MIN/MAX` | 20000–20999 | Whole UDP block (must match any TFT / firewall rules) |
| `MCX_VOICE_PORT_MIN/MAX` | 20000–20499 | Voice RTP/RTCP/floor (blocks of 4) |
| `MCX_VIDEO_PORT_MIN/MAX` | 20500–20999 | Video RTP/RTCP |

Allocated by `media_server.py` PortPool. Change these only if you also change core filters and phone expectations.

---

## 5. Roles and process

| Name | Meaning |
|------|---------|
| `MCX_ROLE` | Which program this container runs: `admin` / `cms` / `gms` / `as` / `media` / `fd` / … |
| `MCX_ADMIN_REQUIRE_AUTH` | `0` in the lab overlay = Control API open on the lab network |
| `MCX_ADMIN_HOST_PORT` | Host map, default `18444` |
| `HPPCN_HUB_HOST_PORT` | Default `18100` |
| `MCX_PRESENCE_SOFT_S` | How long AS keeps soft-state presence (if set) |

Wrong `MCX_ROLE` → a container starts the wrong service entirely.

---

## 6. Hub

| Name | Meaning |
|------|---------|
| `HUB_USER` / `HUB_PASSWORD` | Portal login (**secret**) |
| `HUB_PUBLIC_HOST` | Host hint for cookies / display |
| `HUB_UP_MCPTT` … `HUB_UP_PYHSS` | Upstream URLs the iframe proxies to |
| `HPPCN_SHARED_PASSWORD` | Shared demo password across tools (**secret**) |

---

## 7. Secrets (names only)

| Name | Purpose | How it is created / rotated |
|------|---------|-----------------------------|
| `MCX_DB_PASSWORD` | Postgres | `prepare.sh` / recreate DB (⚠ data) |
| `MCX_IDMS_ADMIN_PASSWORD` | Keycloak admin | `prepare.sh` |
| `MCX_AS_CLIENT_SECRET` | AS OAuth client | `prepare.sh` + IdMS |
| `MCX_SIM_CLIENT_SECRET` | SIM login client | `prepare.sh` |
| `MCX_PW_ALICE` … `ERIN` | Lab user passwords | `--rotate-user-passwords` |
| Subscriber `ki` / `op`/`opc` | SIM auth | provisioning scripts — **never commit** |

---

## 8. Protocol constants (`config.py`)

| Symbol | Role |
|--------|------|
| `ICSI_MCPTT` / `MCVIDEO` / `MCDATA` | IMS Communication Service Identifiers (feature tags) |
| `MCPTT_INFO_CT` etc. | SIP multipart content-types |
| `PF_PSI` / `CF_PSI` | Participating / controlling function public service identities |

These rarely change; wrong values break SIP feature tagging and routing.

---

## 9. Android / lab settings

Runtime: `Lab.kt` / Settings — domain and URLs from the lab profile.  
Build: Gradle `applicationId` `lab.hppcn.mcptt`.  
Design tokens: `res/values/colors.xml` + `docs/hppcn-design-tokens.md`.

---

## 10. Who wins when settings disagree?

1. Process environment (compose) overrides image defaults.  
2. `prepare.sh`-generated `.mcptt_env` feeds compose `env_file`.  
3. Database rows override what CMS/GMS put in XML for user/group policy.  
4. The phone caches its profile until it refreshes — **stale until re-fetch**.

So: fix Admin → wait for re-login → then debug the phone.

---

## 11. Validation tips (no secrets printed)

```bash
# List secret *names* present (not values):
grep -E '^[A-Z0-9_]+=' custom_deployments/mcptt/.mcptt_env | cut -d= -f1

# Confirm AS sees the expected domain / PSI:
docker exec hppcn_mcx_as python -c 'from mcx import config; print(config.DOMAIN, config.CF_PSI)'
```

Next: [06 — Standards](06-3GPP-Standards-and-Traceability.md) or [07 — Protocols and ports](07-Protocols-APIs-Ports-and-Data-Models.md).
