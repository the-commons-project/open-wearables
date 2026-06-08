# End-to-End Test: Oura -> OW -> JHE

A new-developer-friendly walkthrough that verifies the Oura -> Open Wearables (OW) -> JupyterHealth Exchange (JHE) data pipeline running on Fly.io.

Pre-req: the OW backend is already deployed per [../README.md](../README.md). This doc picks up from there and walks all the way to "FHIR Observations visible in JHE for a connected patient".

You need **zero prior knowledge** of OW internals or the JHE data model. Every command is copy-pasteable from Windows Git Bash (or any POSIX shell).

---

## Prerequisites

| What | Where | Value (PoC) |
| --- | --- | --- |
| Windows + Git Bash (or any POSIX shell) | local | `MSYS_NO_PATHCONV=1` prefix is required on Windows for `flyctl ssh` |
| `flyctl` logged in to org `jupyterhealth` | local | `flyctl auth login` |
| OW app deployed | Fly | `ow-poc.fly.dev` |
| JHE app deployed | Fly | `jhe.fly.dev` |
| OW API key (in JHE secrets) | Fly | `sk-1a9980ecae33ef2e8b8abddb0289d8e0` |
| JHE admin login | JHE UI | `admin@example.com` / `Jhe1234!` |
| JHE seeded patient | JHE seed | `ll_patient_peter@example.com` / `Jhe1234!` |
| JHE practitioner (sends invites) | JHE seed | `manager_mary@example.com` / `Jhe1234!` |
| Oura developer app | dash.ouraring.com | redirect URI `https://ow-poc.fly.dev/api/v1/oauth/oura/callback` |

---

## High-level data flow

```
Patient browser
  -> JHE invitation link (/clients/ow/?code=jhe.fly.dev_<token>)
  -> JHE launch.html redeems invite, exchanges PKCE code for access token
  -> JS calls JHE POST /api/v1/ow/users          (creates OW user via OW API key)
  -> JS calls JHE GET  /api/v1/ow/oauth/oura/... (gets OW-generated Oura authorize URL)
  -> Patient consents on Oura
  -> Oura -> OW callback (/api/v1/oauth/oura/callback) stores tokens in OW
  -> OW pulls historical data on connect + Oura webhooks push deltas
  -> OW parses payloads into `data_point_series`
  -> JHE `manage.py ow_poll` reads OW timeseries via API key and writes FHIR Observations
```

---

## One-time JHE configuration

If the JHE app is freshly deployed and these aren't done, do them once. Skip the ones already done.

### 1. Enable the OW module

```bash
MSYS_NO_PATHCONV=1 flyctl ssh console -a jhe -C 'python /code/manage.py shell -c "
from core.models import JheSetting
from django.core.cache import cache
s,_=JheSetting.objects.update_or_create(key=\"module.ow\", setting_id=None, defaults={\"value_type\":\"boolean\"})
s.set_value(\"boolean\", True); s.save()
cache.delete(\"jhe_setting:module.ow\")
print(\"module.ow=\", s.get_value())
"'
```

### 2. Set the public site URL (used to build invitation links)

```bash
MSYS_NO_PATHCONV=1 flyctl ssh console -a jhe -C 'python /code/manage.py shell -c "
from core.models import JheSetting
from django.core.cache import cache
s,_=JheSetting.objects.update_or_create(key=\"site.url\", setting_id=None, defaults={\"value_type\":\"string\"})
s.set_value(\"string\", \"https://jhe.fly.dev\"); s.save()
cache.delete(\"jhe_setting:site.url\")
print(\"site.url=\", s.get_value())
"'
```

If this is wrong (e.g. left as `http://localhost:8000` from the seed), invitation codes will embed the wrong host and the patient browser will fail with "TypeError: Failed to fetch".

### 3. Create the JHE OAuth2 "OW Client" application

Done via the Django admin UI: `https://jhe.fly.dev/admin/oauth2_provider/application/`. Required field values:

| Field | Value |
| --- | --- |
| Client type | Public |
| Authorization grant type | Authorization code |
| Redirect URIs | `https://jhe.fly.dev/clients/ow/` |
| Skip authorization | yes |
| Algorithm | RS256 |
| PKCE required | yes |

Save and note the generated `client_id`.

### 4. Set the invitation URL template

```bash
MSYS_NO_PATHCONV=1 flyctl ssh console -a jhe -C 'python /code/manage.py shell -c "
from core.models import JheSetting
from django.core.cache import cache
s,_=JheSetting.objects.update_or_create(key=\"client.invitation_url\", setting_id=None, defaults={\"value_type\":\"string\"})
s.set_value(\"string\", \"https://jhe.fly.dev/clients/ow/?code=CODE\"); s.save()
cache.delete(\"jhe_setting:client.invitation_url\")
print(\"client.invitation_url=\", s.get_value())
"'
```

### 5. Link the OW Client to the Oura data source

Done via JHE admin: `/admin/core/clientdatasource/add/`. Pick the OW Client app and the Oura DataSource.

### 6. Confirm OW credentials are set on `jhe` app

```bash
MSYS_NO_PATHCONV=1 flyctl secrets list -a jhe | grep OW_
```

You should see `OW_API_URL`, `OW_API_KEY`, and friends.

---

## Run the test

### Step A: practitioner sends Peter an invite

1. Log in to JHE as `manager_mary@example.com` / `Jhe1234!`.
2. Navigate to Patients -> Peter -> "Invitations" tab.
3. Click "Send invitation". Copy the generated link. It should look like `https://jhe.fly.dev/clients/ow/?code=jhe.fly.dev_<token>` (host part = `jhe.fly.dev`, not `localhost`).

### Step B: Peter consents on Oura

1. Open the invitation link in a private window.
2. The launch page shows "Redeeming invitation..." -> "Loading consents..." -> "Redirecting to Oura...".
3. On the Oura page, log in (you can use any real Oura account, including a developer test account).
4. Approve the requested scopes.
5. You should land back on the JHE OW page showing "Connected: Oura".

### Step C: verify Oura -> OW (data landed in OW)

```bash
MSYS_NO_PATHCONV=1 flyctl ssh console -a ow-poc -C 'python -c "
import asyncio
from sqlalchemy import text
from app.database import AsyncSessionLocal
async def go():
    async with AsyncSessionLocal() as s:
        for q in [
            \"select count(*) from \\\"user\\\"\",
            \"select user_id, provider, created_at from user_connection\",
            \"select count(*) from data_point_series\",
            \"select std.code, std.unit, count(*) from data_point_series d join series_type_definition std on std.id=d.series_type_definition_id group by std.code, std.unit order by 3 desc\",
            \"select min(recorded_at), max(recorded_at) from data_point_series\",
        ]:
            r = await s.execute(text(q)); print(\"->\", r.fetchall())
asyncio.run(go())
"'
```

Expected (counts depend on the Oura account, but format is the same):

```
-> [(1,)]
-> [(UUID('ba10df71-...'), 'oura', datetime(...))]
-> [(664,)]
-> [('heart_rate', 'bpm', 249), ('heart_rate_variability_sdnn', 'ms', 249), ('energy', 'kcal', 55), ('distance_walking_running', 'meters', 55), ('steps', 'count', 55), ('skin_temperature_deviation', 'celsius', 1)]
-> [(datetime(2026, 3, 5, 10, 0, ...), datetime(2026, 5, 26, 12, 21, 27, ...))]
```

If `data_point_series` is 0: OW didn't receive any Oura webhook. Check `flyctl logs -a ow-poc | grep -iE "oura|webhook"`. Common cause: Oura redirect URI in the dev portal does not exactly match `https://ow-poc.fly.dev/api/v1/oauth/oura/callback`.

### Step D: verify OW API authenticates with the API key

```bash
curl -H "X-Open-Wearables-API-Key: sk-1a9980ecae33ef2e8b8abddb0289d8e0" \
  https://ow-poc.fly.dev/api/v1/users | jq .
```

Expected: a JSON object with `items` containing the connected user and `has_active_connection: true`. **Note the header name** - it is `X-Open-Wearables-API-Key`, not `Authorization: Bearer`.

### Step E: run JHE `ow_poll` to pull OW data into FHIR observations

The default `POLL_WINDOW` is 1 day. For a first-time backfill of older Oura history, monkey-patch to 90 days:

```bash
MSYS_NO_PATHCONV=1 flyctl ssh console -a jhe -C 'python /code/manage.py shell -c "
from datetime import timedelta
from core.management.commands import ow_poll
ow_poll.POLL_WINDOW = timedelta(days=90)
from django.core.management import call_command
call_command(\"ow_poll\")
"'
```

Expected output:

```
... GET /api/v1/users/<ow_user_id>/timeseries?types=heart_rate&start_time=... 200 None
Poll completed for jhe_user=10006 patient=40001 created=50
OW poll complete (mode=normalized). Created 50 observations.
```

The `created=50` cap is `ow_poll`'s batch limit per run. Subsequent runs will return `created=0` because `ow_poll` uses a high-water-mark strategy (`start_time = max(start_time, last_obs.last_updated - POLL_OVERLAP)`). This means historical data older than the most recently ingested observation is skipped. Backfill is a known limitation - see follow-up issue [jupyterhealth/jupyterhealth-exchange#448](https://github.com/jupyterhealth/jupyterhealth-exchange/issues/448).

### Step F: verify FHIR observations exist in JHE

```bash
MSYS_NO_PATHCONV=1 flyctl ssh console -a jhe -C 'python /code/manage.py shell -c "
from core.models import Observation
qs = Observation.objects.filter(subject_patient_id=40001)
print(\"total=\", qs.count())
print(\"min=\", qs.order_by(\"effective_date_time\").values_list(\"effective_date_time\", flat=True).first())
print(\"max=\", qs.order_by(\"-effective_date_time\").values_list(\"effective_date_time\", flat=True).first())
"'
```

Expected: `total >= 50`, plus the first and last observation timestamps.

### Step G: see the data in the JHE UI

1. Log in to JHE as `admin@example.com`.
2. Navigate to Patients -> Peter (id 40001) -> "Data" tab.
3. You should see heart-rate observations dated within the polled window.

Pipeline verified end-to-end.

---

## What is and isn't validated by this PoC

| Stage | Validated? |
| --- | --- |
| Patient invitation link -> JHE launch page | yes (HTTPS, correct host) |
| JHE OAuth2 PKCE token exchange | yes |
| JHE -> OW user provisioning (API key) | yes |
| Oura OAuth + token storage in OW | yes |
| Oura webhook -> OW parsing -> `data_point_series` | yes |
| OW API (`/users`, `/timeseries`) auth | yes |
| JHE `ow_poll` -> FHIR Observation create | yes (heart_rate only) |
| **Multi-type OW timeseries support in `ow_poll`** | **no** (only `heart_rate` requested) |
| **Historical backfill beyond the high-water-mark** | **no** (one-time gap on first connect) |
| **Tigris raw-payload archival** | **no** (bucket stays empty in this build) |

The "no" items are tracked in [jupyterhealth/jupyterhealth-exchange#448](https://github.com/jupyterhealth/jupyterhealth-exchange/issues/448).

---

## Troubleshooting

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| Invitation link page: "TypeError: Failed to fetch" | Old launch.html cached, or `site.url` JheSetting is still `http://localhost:8000` | Hard-refresh; rerun the JheSetting update in step 2 above |
| Invitation page request goes to `http://jhe.fly.dev/...` (not `https`) | `launch.html` hardcoded `http://` (fixed in `feat/ow-poc`) | Redeploy JHE |
| OW `/api/v1/users` -> 401 "Authentication required" | Wrong header | Use `X-Open-Wearables-API-Key`, not `Authorization: Bearer` |
| Oura callback succeeds but `data_point_series` is empty | Oura webhook subscription not configured for the test account | Trigger a manual sync in OW or wait for next periodic poll |
| `ow_poll` always reports `created=0` after first run | Incremental high-water-mark | Monkey-patch `POLL_WINDOW` AND delete prior observations, OR wait for #448 |
| `ModuleNotFoundError: No module named 'app.db'` on OW SSH | OW uses `app.database`, not `app.db` | Use `from app.database import AsyncSessionLocal` |
| `ModuleNotFoundError: No module named 'core.tasks'` on JHE SSH | `ow_poll` is a management command, not a Celery task | Run `python /code/manage.py ow_poll` directly |
| `env: '|': No such file or directory` on Windows | `MSYS_NO_PATHCONV=1` doesn't escape pipes inside SSH | Wrap remote command in `bash -lc "..."` and escape pipes |
| Fly machine lease "currently held" on deploy | Previous deploy hasn't released the lease | Wait 60s and retry `flyctl deploy` |

---

## Quick smoke test (fastest possible verification)

If you just want a 30-second "is the pipe alive" check after a fresh deploy:

```bash
# 1. Does OW know about a patient with an active Oura connection?
curl -H "X-Open-Wearables-API-Key: sk-1a9980ecae33ef2e8b8abddb0289d8e0" \
  https://ow-poc.fly.dev/api/v1/users | jq '.items[] | {email, has_active_connection, last_synced_provider}'

# 2. Did ow_poll create at least one observation?
MSYS_NO_PATHCONV=1 flyctl ssh console -a jhe -C 'python /code/manage.py shell -c "
from datetime import timedelta
from core.management.commands import ow_poll
ow_poll.POLL_WINDOW = timedelta(days=90)
from django.core.management import call_command
call_command(\"ow_poll\")
"'
```

If both succeed, the pipeline is alive.
