# 03 — Operator Runbook

This is the day-to-day cookbook: **start the lab, put SIMs and users in place, check that things are healthy, run a two-phone demo, and recover when something breaks.**

Commands come from `custom_deployments/mcptt/README.md`, `docs/mcx-e2e-runbook.md`, and the compose files. Steps that can destroy state are marked ⚠. **Never paste secret values** from `.mcptt_env` into tickets or into this handbook.

**Project name used in recent lab work:** `hppcn_mcptt`  
**Typical host:** `DOCKER_HOST_IP=192.168.1.51`

```bash
export COMPOSE="docker compose -f deploy-all.yaml -f custom_deployments/mcptt/mcptt-overlay.yaml --project-name hppcn_mcptt"
```

---

## 1. What you need before you start

| Need | Why |
|------|-----|
| Docker + Compose v2 | Everything runs as containers on this host |
| `.env` at the repo root | Core/IMS IPs and passwords for Open5GS |
| `custom_deployments/mcptt/.mcptt_env` | Created by `prepare.sh` — holds MCPTT secrets |
| Optional BladeRF + srsRAN | Real radio for two-phone E2E demos |
| Android SDK / `adb` | Install and debug the APK |
| Lab CA on the phones | So HTTPS to CMS / IdMS / SIP TLS is trusted |

If prepare was never run, stop and do §2 first.

---

## 2. First-time prepare (safe to re-run)

```bash
custom_deployments/mcptt/prepare.sh
# Optional password rotation (⚠ recreates IdMS users):
# custom_deployments/mcptt/prepare.sh --rotate-user-passwords
```

**What it does.** Creates or refreshes certificates under `pki/out`, IdMS realm pieces, and the secret file `.mcptt_env`.

**Healthy.** Exit code 0; `pki/out/ca.crt` exists.  
**Unhealthy.** Missing `openssl`, or permission errors under `pki/`.

---

## 3. Bring the stack up

```bash
$COMPOSE up -d
custom_deployments/mcptt/tools/check_namespace.sh
```

The check script confirms the MCPTT containers are on the expected Docker network (lab isolation).

**After you change MCPTT Python/Hub code and rebuild:**

```bash
$COMPOSE build mcx-admin mcx-as mcx-media mcx-cms mcx-gms hppcn-hub
$COMPOSE up -d --force-recreate --no-deps mcx-admin mcx-as mcx-media hppcn-hub
```

⚠ `--force-recreate` drops in-memory call and floor state on AS/media. Active demos will die; that is expected.

---

## 4. Subscriber / SIM provisioning

```bash
custom_deployments/mcptt/provisioning/provision_mc.sh lab
# or device-oriented helpers:
custom_deployments/mcptt/provisioning/provision_lab_sims.sh
```

| Command idea | Effect |
|--------------|--------|
| `add <imsi> <ki> --op/--opc …` | Open5GS subscriber + sessions |
| `mc-rules <imsi\|all>` | ⚠ Preloads static MC packet filters — **dev fallback only**, not the production QoS story |
| `remove-mc <imsi\|all>` | ⚠ Removes those static rules |
| `show <imsi\|all>` | Lists what is stored |

IMS / PyHSS identities are separate from the MCPTT Postgres users — follow the top-level `README.md` for those.

---

## 5. Radio (LTE BladeRF example)

From `docs/mcx-e2e-runbook.md`:

```bash
docker compose -f srsenb.yaml up -d
docker logs -f hppcn_srsenb
custom_deployments/mcptt/tools/device/device_info.sh
```

**Expect.** Cell on air; phone `rmnet` address in `192.168.100.0/24`; PLMN **901/70**.

No radio? You can still exercise containers and Hub; you cannot prove end-to-end PTT over the air.

---

## 6. Android APK

```bash
cd custom_deployments/mcptt/android/HppcnMcptt
./gradlew :app:assembleDebug

custom_deployments/mcptt/tools/device/install_ca.sh
custom_deployments/mcptt/tools/device/provision_apn.sh
custom_deployments/mcptt/tools/device/install_app.sh
# or:
adb install -r app/build/outputs/apk/debug/app-debug.apk
```

Lab users (passwords **only** in `.mcptt_env` as `MCX_PW_*`): alice, bob, carol, dave, erin — see seed `db/02_seed.sql`.

Grant microphone (and camera for video) when Android asks.

---

## 7. Health checks (read-only)

```bash
# Are the important boxes up?
docker ps --format '{{.Names}} {{.Status}}' | grep -iE 'mcx|hub|mme|upf|pcscf'

# Hub portal
curl -s http://127.0.0.1:18100/healthz

# Admin / AS / Media
curl -sk https://127.0.0.1:18444/healthz
curl -s http://172.29.0.205:8080/healthz
curl -s http://172.29.0.206:8081/healthz

# Live sessions (empty {} is fine when nobody is in a call)
curl -s http://172.29.0.206:8081/v1/sessions | head -c 400; echo
curl -sk https://127.0.0.1:18444/v1/status
```

| Result | Meaning |
|--------|---------|
| Hub `{"ok": true` | Portal process is up |
| Admin status JSON with domain/users | Database reachable |
| Media `{}` or a session map | Media process up; empty = no calls |
| healthz fail | Jump to [08 Troubleshooting](08-Troubleshooting-and-Observability.md) |

---

## 8. Where to look at logs

```bash
docker logs --tail 100 hppcn_mcx_as
docker logs --tail 100 hppcn_mcx_media
docker logs --tail 100 hppcn_mcx_sip_edge    # P-CSCF MCPTT edge (same container name historically)
docker logs --tail 50 hppcn_hub
# On the phone:
adb logcat -s MCX
```

---

## 9. Acceptance suite (profile `test`)

```bash
$COMPOSE --profile test run --rm mcx-uesim
# subset: mcx-uesim TC-013 TC-016
# report lands in: custom_deployments/mcptt/reports/mcx-acceptance.json
```

This is automated signalling/media smoke — not a substitute for a two-phone radio demo.

---

## 10. Open Hub in a browser

1. Go to `http://192.168.1.51:18100/` (or your `DOCKER_HOST_IP`).  
2. Log in with the shared lab Hub account (`HUB_USER` / `HUB_PASSWORD` — **do not publish**).  
3. Open **HPPCN MCPTT Control**.  
4. Direct Admin on port `18444` may need the Hub proxy and a trusted lab CA in the browser.

---

## 10b. Postgres migrations on an *existing* volume

SQL files under `custom_deployments/mcptt/db/` run **only when the data volume is empty** (first init). If the lab DB was created before later migrations (`03_sim_bind.sql` … `07_imminent_peril.sql`), those files are **not** applied automatically.

**Check (read-only):**

```bash
docker exec hppcn_mcx_db psql -U mcx -d mcx -c "\d mc_user" | head
# Look for columns such as allow_imminent_peril_call, lab_password, sim_bind
```

**Apply a missing migration (⚠ writes schema):** copy the needed `.sql` into the container and run with `psql`, or recreate the volume only if you accept data loss.

---

## 11. Backup / restore (guidance)

| Data | Where | Notes |
|------|-------|-------|
| MC database | Docker volume (often `mcxdbdata`) | ⚠ `pg_dump` as user `mcx` |
| Certificates | `custom_deployments/mcptt/pki/out` | Protect private keys |
| Secrets | `.mcptt_env` | Never commit |
| Open5GS Mongo | mongo volume | Subscribers |

Example dump (**read**):

```bash
docker exec hppcn_mcx_db pg_dump -U mcx mcx > /tmp/mcx-$(date +%F).sql
```

⚠ Restore overwrites live users and groups — isolate the lab first.

---

## 12. Rebuild Open5GS or Kamailio images

After changing patches under `base/patches` or `ims_base/patches`:

```bash
docker build -t hppcn_mcptt:latest ./base
docker build -t hppcn_mcptt_kamailio:latest ./ims_base
$COMPOSE up -d --force-recreate --no-deps pcrf pcf pcscf mme smf upf   # adjust the list to what you rebuilt
```

Confirm the patched strings/symbols still appear in the new images (see notes under `docs/adr/` for QoS work).

---

## 13. Safe demo sequence (voice)

1. Both phones show **Registered** in the app.  
2. Both are affiliated to `grp-ops` (or your demo group).  
3. Alice starts a group call → Bob auto-answers.  
4. Alice holds PTT → she sees **Granted**; Bob sees **Taken** and hears audio.  
5. Alice releases; Bob holds PTT; then Hangup.

Helpers: `tools/device/ptt.sh <serial> call|down|up|hang`.

---

## 14. Rollback without panic

- **APK:** `adb install -r` a previous known-good APK.  
- **Containers:** redeploy previous image tags if you tagged them; otherwise `git checkout` the known revision and rebuild.  
- ⚠ Avoid `docker compose down -v` unless you *intend* to wipe volumes.

---

## Quick “is the lab usable?” checklist

- [ ] `prepare.sh` already succeeded once  
- [ ] `$COMPOSE` containers for AS, media, CMS, Hub are Up  
- [ ] Hub `/healthz` and Admin `/v1/status` respond  
- [ ] Phones have lab CA + APN and can resolve `*.mcx.hppcn.lab`  
- [ ] Two users login and show Registered  
- [ ] One PTT cycle grants floor and carries audio  

Next: [04 — Hub and Android manual](04-WebUI-and-Android-User-Manual.md).
