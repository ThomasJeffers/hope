# 04 — Hub and Android User Manual

This chapter is for people who **click buttons**: operators in a browser, and users on the phone.  
It explains what each screen *means*, not every line of source code. Paths are cited so you can verify.

Grounded in `hppcn_hub/hub.py`, `mcx/admin_console.html`, `mcx/admin.py`, and Android activities under `lab.hppcn.mcptt`.

---

## A. HPPCN Hub (browser)

### What Hub is (and is not)

Hub is a **lab portal**: one login, a sidebar, and an iframe that loads Admin, Grafana, Keycloak, and other tools with a shared look.

Hub is **not** a 3GPP network function. Deleting Hub would annoy operators; it would not stop MCPTT calls by itself.

| | |
|--|--|
| **Who** | Lab operators / demo hosts |
| **URL** | `http://<DOCKER_HOST_IP>:18100/` |
| **Login** | Form POST `/login` — `HUB_USER` / `HUB_PASSWORD` (not IdMS) |

### Chrome controls

| Control | What it does |
|---------|----------------|
| Collapse | Hides the sidebar |
| Language | `/lang/fa` or `en` — switches fonts (Carlito vs IranSansWeb) |
| Log out | Clears the session cookie |
| App buttons | Loads `/go/<id>/…` into the main iframe |

### Apps you can open from the sidebar

| id | Title | Upstream setting |
|----|-------|------------------|
| mcptt | MCPTT Control | `HUB_UP_MCPTT` |
| charging | Charging / OCS UI | `HUB_UP_CHARGING` |
| ops | Grafana | `HUB_UP_GRAFANA` |
| core | Open5GS WebUI | `HUB_UP_WEBUI` |
| idms | Keycloak | `HUB_UP_IDMS` |
| speed | Speed lab | `HUB_UP_SPEED` |
| hss | PyHSS Swagger | `HUB_UP_PYHSS` |

Skin CSS is injected into proxied pages (`skin.css`). The MCPTT Control UI keeps its own font via `hppcn-own-font`.

---

## B. MCPTT Control (Admin)

Served as `admin_console.html` from `admin.py` `GET /`.  
**Tabs:** Live | Users | Groups | Audit.

### Live tab — “what is happening right now?”

| Widget | Where data comes from | Meaning |
|--------|----------------------|---------|
| Registered clients | `/v1/live` ← AS | Who is registered, contact IP, TTL, affiliations |
| Active calls & floor | `/v1/live` ← Media | Session id, floor holder, rx/tx, queue |
| AS / Media pills | same fetch | Green = Admin could reach that HTTP API |

The page polls live data about every 3 seconds. Static tables refresh on load or when you change something.

**Remember:** “AS up” on Hub does **not** prove the phone has audio. Check floor holder + RTP counters.

### Users table — directory of people

| Column | What it controls |
|--------|------------------|
| User / Display | Username; display name via PATCH |
| IMSI / SIM | Bound IMSI(s); `sim_bind` any\|bound (CMS authorization) |
| Password | Show / Reset / Set — talks to IdMS; Show opens a sticky bar (plain text once) |
| Prio | Application priority for floor arbitration |
| N6 | Max simultaneous calls the client may receive |
| Monitor | `off` / `soft` / `full` multi-group audio behaviour |
| Authorisations | Emergency / imminent peril / cancel peril — CMS profile bits |
| Role | Dispatcher flag |
| Private list | Who appears in Private Call contacts |
| Delete | Removes user (`?force=1` needed for seed users) |

**Add user** → `POST /v1/users` (creates an IdMS password returned **once** — copy it offline).

**Important.** Saving a flag updates Postgres immediately. The **phone must refresh its CMS profile** (re-login / re-register) before the device behaves differently. The console help text says the same thing.

### Groups table

| Column | Meaning |
|--------|---------|
| Name / Prio / T2 / hang | Group identity and floor timers |
| Allow peril | Whether this group may run imminent peril |
| Members | Chips + IA (implicit affiliation) toggle + remove; Add with optional IA |
| Delete | Removes the group |

### Audit

`GET /v1/audit` — recent admin actions (not live-polled). Useful after “who changed Bob’s peril flag?”

### Open5GS WebUI “MC Critical”

Documented in `docs/webui-mc-demo.md`. It applies or removes **static** MC packet filters (`mc-rules`). Treat that as a **lab shortcut**, not the production QoS story (see textbook chapter 6 and doc 02).

---

## C. Android client (`lab.hppcn.mcptt`)

### Installation

See runbook §6. Grant **Microphone**, **Camera** (for MCVideo), and notifications when prompted. Keep the screen on during demos — Android may kill background audio.

### Screens (what each activity is for)

| Screen | Role in plain words |
|--------|---------------------|
| Login | IdMS password or SIM-based login |
| Home (Ptt) | Selected group, Hold-to-talk, emergency / peril, status line |
| Groups | Affiliate / select which group is active |
| Private call | One-to-one voice or video contacts |
| Video | MCVideo preview + remote + hold-to-transmit |
| Data | Short data (SDS) and file distribution (FD) |
| GMC | Group-management helpers |
| Settings | Server / lab profile knobs |
| Logs / Debug / NetDiag | Diagnostics |

### Status phrases — do not over-read them

| UI phrase | Means | Does **not** prove |
|-----------|-------|---------------------|
| Registered | SIP REGISTER succeeded | Floor or audio |
| Affiliated | PUBLISH accepted for a group | Already in a call |
| GRANTED / TALKING | This phone holds the floor | Peer decoded your audio |
| TAKEN | Someone else holds the floor | |
| AS up (Hub) | Admin reached AS HTTP | The UE path works |

### How PTT and video feel on the device

- **Voice:** Hold PTT on Home → client sends a floor Request → media Grants or queues you.  
- **Video:** open MCVideo → hold transmit; the app arms video TX and starts or joins the video leg.  
- **Hangup:** ends the presented call / video dialog.

**Emergency** and **Imminent peril** are separate buttons/flows. Emergency is not a synonym for peril. Both need the matching CMS authorisation flags, or the application server will refuse.

### Permissions and background

Battery “optimizations” can delay SIP refresh — if REGISTER flaps on an OEM phone, check those settings. Export logs via Logs screen or `adb logcat -s MCX`.

---

## D. Realistic operator tasks (practice)

1. **Enable imminent peril for bob:** Hub → Users → Peril checkbox → bob re-logins so CMS refreshes.  
2. **Add carol to Jungle with IA:** Groups → Add member + IA checked.  
3. **Reset alice password:** Reset → copy from sticky bar → give to the device operator offline.  
4. **Watch floor live:** Live tab during PTT; confirm the holder matches who is talking.

Next: [05 — Configuration catalogue](05-Configuration-and-Variable-Catalogue.md).
