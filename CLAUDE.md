# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Alexa skill (Python 3.11, AWS Lambda + DynamoDB, `eu-west-1`) that speaks the next Irish Rail **Commuter** trains due at a station (nationwide: Dublin commuter incl. Dundalk/Longford/Portlaoise/Gorey, Cork, Waterford-Limerick Junction), using the NTA GTFS-Realtime API. Store name **4NextLocalTrain**, invocation **four next local train**. Sibling of 4NextBus / 4NextTram / 4NextDart / 4NextLocalTrain, which share the same structure, so fixes often apply across all of them.

## Dev server / port

There is **no dev server and no port**. This is a Lambda skill. Run it locally via the proof-of-concept script or tests, or invoke the deployed Lambda with `tools/deploy.py test`.

## Commands (Windows / PowerShell)

```powershell
python -m venv .venv
.\.venv\Scripts\python -m pip install -r requirements-dev.txt
copy .env.example .env      # NTA_API_KEY (+ ALEXA_SKILL_ID, ALERT_EMAIL for deploys)

.\.venv\Scripts\python tools\poc_next_local_train.py Maynooth --direction city --no-realtime
.\.venv\Scripts\python -m pytest -q                                   # all tests
.\.venv\Scripts\python -m pytest tests\test_stations.py::test_name -q   # single test
.\.venv\Scripts\python tools\build_voice_model.py                     # regenerate interaction model from GTFS
python tools\make_skill_icons.py                                       # icons from 4NextLocalTrainIcon.jpg

python tools\deploy.py all          # first-time stack + upload (prints Lambda ARN)
python tools\deploy.py code         # later code changes
python tools\deploy.py test --station Pearse --direction southbound   # smoke-test deployed Lambda
python tools\deploy.py ingest-key   # key for GitHub secrets AWS_ACCESS_KEY_ID / AWS_SECRET_ACCESS_KEY (optional vars `AWS_REGION`, `FOURNEXTLOCALTRAIN_TABLE`)
python -m ingest.build_timetable    # timetable load into DynamoDB
```

TLS: if pip fails with `CERTIFICATE_VERIFY_FAILED` (antivirus/proxy), bootstrap once with `--trusted-host pypi.org --trusted-host files.pythonhosted.org --upgrade pip truststore`. Never use `verify=False`.

## Architecture

- `src/skill/app.py`: ask-sdk-core handlers and the Lambda `handler`. `src/skill/service.py`: application service with per-container caches.
- `src/common/`: `stations` (station-name/direction matching), `predictions` (timetable + realtime merge), `speech`, `gtfs_time` (Europe/Dublin), `realtime` (NTA fetch), `rt_cache`, `store` (single-table DynamoDB, low-level boto3 client for fast cold starts).
- `ingest/`: DuckDB over the TFI `GTFS_Realtime.zip` into DynamoDB. Filter: `route_type 2` plus a Commuter **corridor allowlist** (the live zip labels most rail as `rail`, not `Commuter`). Only Commuter trains are spoken, even at stations with DART/InterCity. Do not ingest standalone per-operator GTFS zips; stop/trip IDs must match GTFS-R v2.
- `skill-package/`: `skill.json` and `interactionModels/custom/en-GB.json` (pasted into the Alexa console JSON editor). Icons in `skill-package/assets/images/`.
- `docs/`: GitHub Pages (privacy page and icons).
- `template.yaml`: CloudFormation. AWS names: `FourNextLocalTrain` (table/stack), Lambda `FourNextLocalTrain-Skill`, SSM `/4nextlocaltrain/nta_api_key`.
- Timetable ingest via `.github/workflows/ingest.yml` (`workflow_dispatch`); check the file for any schedule.

## Domain rules worth knowing

- **Shared NTA rate limit:** NTA issues one GTFS-R key per developer account, shared by all the 4Next skills. Before calling NTA, `rt_cache` claims the same DynamoDB lock item as the bus skill (`FourNextBus` table, `META/RTLOCK`) so all skills combined make at most one TripUpdates call per 60 s. Losers use their cached feed or fall back to timetable ("scheduled"). Set `NTA_LOCK_TABLE` if the bus table isn't `FourNextBus`.
- A favourite is one direction at a named station; directions are split by destination corridor.
- Dialog is `SKILL_RESPONSE`: the Lambda elicits `station` / `direction` itself.
- Never reuse another 4Next skill's Alexa skill ID.
