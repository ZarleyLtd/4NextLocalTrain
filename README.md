# 4NextLocalTrain

Alexa skill that tells you when the next Irish Rail **Commuter** trains are due at a station, using the
NTA GTFS-Realtime API. Sibling of [4NextBus](https://github.com/ZarleyLtd/4NextBus),
[4NextTram](https://github.com/ZarleyLtd/4NextTram), and [4NextDart](https://github.com/ZarleyLtd/4NextDart).
Store name: **4NextLocalTrain**. Invocation: **four next local train**.

- "Alexa, ask four next local train from Maynooth towards the city"
- "Alexa, ask four next local train from Pearse southbound"
- "Alexa, ask four next local train to set my favourite stop to Dundalk towards Dublin"
- "Alexa, ask four next local train for my favourite stop"
- "Alexa, ask four next local train" (uses your favourite)

Only **Commuter** trains are included, even at stations that also have DART or InterCity
services. Coverage is nationwide: Dublin commuter (including Dundalk, Longford, Portlaoise,
and Gorey), Cork (Mallow, Cobh, Midleton), and Waterford–Limerick Junction. A favourite is
**one direction at a named station** (GTFS uses one stop id per station; directions are split
by destination corridor).

## You do this

The code and AWS scripts live in this repo. These steps need you (Amazon console, GitHub
Pages, secrets). Do not reuse the 4NextBus, 4NextTram, or 4NextDart skill IDs.

### Before deploy

1. **Amazon Developer Console** — Create a custom skill named `4NextLocalTrain`, locale **en-GB**,
   invocation **four next local train**. Copy the skill ID (`amzn1.ask.skill....`).
2. Check on amazon.co.uk that nothing already intercepts "four next local train".
3. Reuse the existing AWS account (`eu-west-1`) and the same NTA GTFS-R key as bus/tram/dart.
4. GitHub repo: [ZarleyLtd/4NextLocalTrain](https://github.com/ZarleyLtd/4NextLocalTrain).
5. Optional: drop `4NextLocalTrainIcon.jpg` in the project root, then run
   `python tools/make_skill_icons.py`.

### After the Lambda exists

6. Copy `.env.example` to `.env`. Set `ALEXA_SKILL_ID`, `ALERT_EMAIL`, `NTA_API_KEY`.
7. `python tools/deploy.py all` — creates stack `FourNextLocalTrain`, Lambda `FourNextLocalTrain-Skill`,
   SSM `/4nextlocaltrain/nta_api_key`. It prints the Lambda ARN.
8. First timetable: `python -m ingest.build_timetable`.
9. Alexa console:
   - Build → JSON Editor: paste `skill-package/interactionModels/custom/en-GB.json` → Save & Build.
   - Endpoint: `eu-west-1`, paste the Lambda ARN.
   - Distribution: upload `skill-package/assets/images/en-GB_smallIcon.png` (108×108) and
     `en-GB_largeIcon.png` (512×512). Privacy:
     https://zarleyltd.github.io/4NextLocalTrain/privacy.html
10. **GitHub Pages** on this repo from `/docs` so the privacy page and icons are live.
11. `python tools/deploy.py ingest-key` → GitHub secrets `AWS_ACCESS_KEY_ID` /
    `AWS_SECRET_ACCESS_KEY` (optional vars `AWS_REGION`, `FOURNEXTLOCALTRAIN_TABLE`).
12. Console Test tab / a device: Maynooth towards the city, Connolly towards Drogheda,
    Kent towards Cobh, set favourite, launch with no slots, Help/Stop.
13. Publish when you want it in the store.

Smoke test: `python tools/deploy.py test --station Pearse --direction southbound`.

Later code changes: `python tools/deploy.py code`.

## Layout

```
src/common/        GTFS time handling, predictions, realtime fetch, speech, DynamoDB, station matching
src/skill/         Lambda handler (ask-sdk-core)
ingest/            daily timetable build (DuckDB over the TFI GTFS zip -> DynamoDB)
tools/             local proof-of-concept and helper scripts
skill-package/     Alexa skill manifest and en-GB interaction model
tests/             pytest
template.yaml      CloudFormation: DynamoDB table, Lambda, role, SSM parameter
```

## Voice model

Canonical files:

- Interaction model: [`skill-package/interactionModels/custom/en-GB.json`](skill-package/interactionModels/custom/en-GB.json)
- Store listing: [`skill-package/skill.json`](skill-package/skill.json)
- Regenerate the model from GTFS: `python tools/build_voice_model.py`

Paste the JSON into the Alexa console **Build → Interaction Model → JSON Editor**, then Save
and Build. A new unpublished skill can start with `en-GB` only.

### Invocation vs display name

| What users see / say | Value |
| --- | --- |
| Skill name (store and console) | `4NextLocalTrain` |
| Invocation name (spoken) | `four next local train` |

### Custom intents

| Intent | Slots | Purpose |
| --- | --- | --- |
| `NextLocalTrainIntent` | `station` (`LOCAL_TRAIN_STATION`), optional `direction` (`LOCAL_TRAIN_DIRECTION`) | Times at a station, or at the favourite |
| `SetFavouriteStopIntent` | same | Save one favourite direction at a station |
| `GetFavouriteStopIntent` | none | Read back the favourite |

Direction phrases, where they apply at that station:

- Cardinal: northbound / southbound / eastbound / westbound
- City: towards the city (suburban stations only; city hubs elicit a terminus)
- Terminus: towards Dundalk, Maynooth, Portlaoise, Gorey, Cobh, and other named destinations

Dialog is `SKILL_RESPONSE`: Lambda elicits `station` or `direction` when needed.

## Data sources

- Realtime: `https://api.nationaltransport.ie/gtfsr/v2/TripUpdates` (header `x-api-key`), max 1 call / 60 s **per key**.
- Static timetable: `https://www.transportforireland.ie/transitData/Data/GTFS_Realtime.zip`, **route_type 2** and a Commuter corridor allowlist (the live zip labels most rail as `rail`, not `Commuter`).
  Do not ingest standalone `GTFS_Irish_Rail.zip` for production IDs — they must match GTFS-R v2.

## Local setup (Windows / PowerShell)

```powershell
python -m venv .venv
.\.venv\Scripts\python -m pip install -r requirements-dev.txt
copy .env.example .env   # then fill in NTA_API_KEY
.\.venv\Scripts\python tools\poc_next_local_train.py Maynooth --direction city --no-realtime
.\.venv\Scripts\python -m pytest -q
.\.venv\Scripts\python tools\build_voice_model.py
```

If `pip` fails with `CERTIFICATE_VERIFY_FAILED`, bootstrap once with
`--trusted-host pypi.org --trusted-host files.pythonhosted.org --upgrade pip truststore`.

NTA only issues one GTFS-Realtime subscription per developer account. 4NextLocalTrain
shares that key with live 4NextBus, 4NextTram, and 4NextDart. Before calling NTA it claims the
same DynamoDB lock item as the bus skill (`FourNextBus` `META/RTLOCK`), so the
combined skills stay at one TripUpdates call per 60 seconds. The loser uses a
cached feed if that Lambda container has one, otherwise timetable times (“scheduled”).
Set `NTA_LOCK_TABLE` if the bus table name is not `FourNextBus`.

Daily ingest in `.github/workflows/ingest.yml` runs at **00:55 UTC** (after 4NextBus
00:10, 4NextDart 00:25, 4NextTram 00:40). GitHub’s public `schedule` queue is often 6–7 hours late.

## Free tier

Same shape as 4NextBus: Lambda, one provisioned DynamoDB table, CloudWatch Logs, SSM Parameter Store,
GitHub Actions for the daily timetable build.
