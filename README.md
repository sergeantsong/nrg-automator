# Tesla Energy Automator

Self-hosted app for a home with Tesla solar, a Powerwall 3, a Wall Connector and a 2018 Model X on
SCE TOU-PRIME. It:

- tracks live and historical energy and costs every interval at TOU-PRIME rates,
- sends surplus solar into the Model X instead of exporting it, while making sure the Powerwall is full before peak,
- never pulls EV charging energy from the grid during peak (4-9pm),
- optionally tops the EV up overnight in the cheapest TOU slots when solar won't reach your minimum SoC by departure,
- lets you toggle automation and set the charging amps by hand next to live solar output,
- replays your history to show what you would save by charging from solar, from off-peak grid, or both.

Polling is conservative. The 2018 Model X is never woken just to read data, and Fleet API spend stays under a
monthly budget that you set.

## How it works

```
Tesla Fleet / Owner API ─┐                    ┌─> controller ──> set amps / start / stop / Powerwall settings
Wall Connector (LAN) ────┼─> adaptive poller ─┤
SCE Green Button CSV ────┘        │           └─> SQLite (readings, sessions, intervals, commands, api usage)
                                  │                       │
                               tariff engine ─────> reports + savings simulator ─> FastAPI ─> React dashboard
```

- `backend/app/tesla/`: `TeslaClient` interface with Fleet API, Owner API (teslapy) and mock implementations, plus the Wall Connector local reader.
- `backend/app/tariff/`: TOU engine and `rates.yaml` (editable from Settings).
- `backend/app/poller.py`: adaptive polling, API usage and budget guard, charge session tracking.
- `backend/app/controller/solar_follow.py`: charging decisions (details below).
- `backend/app/analytics/`: Tesla history backfill, Green Button import, cost reports, savings simulator.
- `frontend/`: dashboard with Live, Reports, Savings and Settings pages.

### Controller

Runs every 30 s using the latest stored reading. It makes no API calls of its own except commands.

| Situation | Behavior |
| --- | --- |
| Peak (on/mid-peak, 4-9pm) | EV charges only from solar beyond house load (and Powerwall charging if not full). Any grid import above the tolerance immediately lowers amps or stops charging. If data is stale, charging stops. |
| Daytime, off-peak | `surplus = solar - house load - Powerwall need`, where the Powerwall need is the power required to reach its target SoC by 4pm. Amps = surplus / volts, clamped to 5-48 A. Starts after 10 min of surplus and stops after 10 min without it. Amps change only in steps of at least 2 A, no more often than every 5 min (increases) or 2 min (decreases). |
| Solar + off-peak top-up mode | If the EV is below its minimum SoC, the controller picks the cheapest non-peak 15-minute slots before departure and charges at max amps during them. |
| Manual mode | Your amps and start/stop choices apply. The peak guard still stops charging unless "Peak override" is on. |
| Stale data (> 15 min) | Holds the current state and never increases amps. |

Every command is logged with its reason (Live page, "Recent commands"). **Automation starts disabled and in dry-run
mode.** Watch the dry-run decisions for a day or two before turning dry run off.

If you enable "Let automation change Powerwall settings", the app sets the operation mode and backup reserve at peak
start, restores the reserve afterwards, and applies the export rule (`pv_only` by default) while the EV absorbs solar.

Turn off any **scheduled charging in the car**, or it will fight the controller.

### Polling budget (Fleet API)

| Source | Interval |
| --- | --- |
| Energy site live status | 3 min (daylight, EV plugged in, automation on) / 10 min (daylight) / 30 min (night) |
| Vehicle state (does not wake the car) | 12 min (24 min with a Wall Connector) |
| Vehicle charge state | 10 min, only while the car is already awake and plugged in |
| Wall Connector local API | 30 s, free |
| Energy history | once per night (1:30am) |

The app projects monthly spend from the last 24 hours and the month so far. If the projection is over budget
(default $10, about Tesla's monthly credit), all intervals stretch up to 6x. Usage is shown on the Settings page.

## Quick start (mock data)

```bash
cp .env.example .env        # leave TESLA_API_MODE=mock
docker compose up -d --build
open http://localhost:8080
```

Mock mode simulates the site and car and generates 30 days of history, so every page can be tried before
connecting Tesla.

## Connecting your Tesla account

Set `TESLA_API_MODE` in `.env` to `fleet` (recommended) or `owner`, then restart with `docker compose up -d`.

### Option A: Official Fleet API

1. **Create an app** at [developer.tesla.com](https://developer.tesla.com) (Tesla account, then Dashboard, then Create Application).
   - OAuth grant type: Authorization Code and Machine-to-Machine.
   - Allowed origin: the domain you will host the public key on (step 2), e.g. `https://yourname.github.io`.
   - Allowed redirect URI: `http://localhost:8080/api/auth/tesla/callback`, or your LAN or HTTPS URL. It must match `FLEET_REDIRECT_URI`.
   - Scopes: Vehicle Information, Vehicle Charging Management, Energy Product Information, Energy Product Settings.
   - Copy the Client ID and Client Secret into `FLEET_CLIENT_ID` and `FLEET_CLIENT_SECRET`.
2. **Host a public key.** Tesla requires one at
   `https://<your-domain>/.well-known/appspecific/com.tesla.3p.public-key.pem`:
   ```bash
   openssl ecparam -name prime256v1 -genkey -noout -out private-key.pem
   openssl ec -in private-key.pem -pubout -out public-key.pem
   ```
   GitHub Pages works: create a repo `<you>.github.io` and commit `public-key.pem` to
   `.well-known/appspecific/com.tesla.3p.public-key.pem`. Add a `.nojekyll` file so the dot-folder is served.
   Keep `private-key.pem` private. This app doesn't need it, because a 2018 Model X accepts legacy (unsigned)
   commands and energy commands are never signed.
3. Set `FLEET_DOMAIN=<your-domain>` (no scheme), restart, open **Settings**, and click **Register partner domain**
   (one time per app).
4. Click **Sign in with Tesla** and approve the scopes. Tokens are encrypted at rest with `APP_SECRET_KEY`.
5. Set a **monthly budget** under Settings, then Polling. Fleet API pricing (per request: data about $0.002,
   commands about $0.001, wakes about $0.02) is configurable in `.env` in case Tesla changes it.

### Option B: Owner API (unofficial)

Set `TESLA_API_MODE=owner` and `OWNER_EMAIL=you@example.com`. In **Settings**, click **Sign in to Tesla**, log in
on the page that opens, and paste the final `https://auth.tesla.com/void/callback?...` URL back into the app. Tesla is
phasing this API out, so expect it to stop working at some point. API costs are reported as $0 in this mode.

### Wall Connector (recommended)

A Gen 3 Wall Connector exposes `http://<ip>/api/1/vitals` on your LAN. Set `WALL_CONNECTOR_IP` (give it a
DHCP reservation). The app then reads plug-in state and charging power every 30 s for free, which also cuts vehicle
polling in half and gives exact EV energy for reports.

## Tariff

`backend/app/tariff/rates.yaml` holds TOU-PRIME seasons, time windows, rates, the daily fixed charge, holidays and
export credits. **The seeded rates are approximate.** Check them against your latest SCE bill or the SCE tariff
book and edit them under Settings, then Tariff. Saved edits go to `data/rates.yaml` and survive upgrades. Add each
year's holidays, because SCE treats them like weekends.

Under the Net Billing Tariff (NEM 3.0), export credits are low and vary by hour, so set `export.default` and
`export.by_period` to your typical credits. Under NEM 2.0, use your import rate minus non-bypassable charges.

## Reports and savings

- **Reports**: daily or monthly cost (import, export credit, fixed charge), cost without solar or battery, kWh by
  TOU period, EV energy split into grid vs. solar/battery with its cost per kWh, and home charging sessions.
- **Savings**: replays every 15-minute interval with a simple self-powered Powerwall model. Each day's EV energy
  is re-placed under three strategies:
  - **Solar-optimized**: into intervals that exported solar.
  - **Off-peak grid**: into the cheapest non-peak slots.
  - **Solar + off-peak**: surplus first, the rest in the cheapest slots.

  Savings are measured against "Modeled actual", which is the same model run on your real charging times, so
  model error cancels out. When no EV data exists yet, EV energy is estimated from load spikes of 5.5 kW or more.
- **History**: the app backfills 30 days of Tesla `calendar_history` on first start (one request per day) and then
  fetches it nightly. Use **Fetch Tesla history** for more days. Import your **SCE Green Button** CSV (sce.com,
  then Data Sharing & Download, then Download My Data, 15-minute CSV) to compare meter data with Tesla's numbers.

## Development

```bash
# backend
cd backend
uv venv --python 3.12 .venv && uv pip install -r requirements-dev.txt --python .venv/bin/python
.venv/bin/pytest
DATA_DIR=./data TESLA_API_MODE=mock .venv/bin/uvicorn app.main:app --reload --port 8000

# frontend (proxies /api to :8000)
cd frontend && npm install && npm run dev
```

## Security notes

- Set `APP_PASSWORD` and a random `APP_SECRET_KEY`. The dashboard uses an HttpOnly session cookie, and login is rate-limited.
- Keep it on your LAN or behind a VPN or reverse proxy with HTTPS. Set `COOKIE_SECURE=true` when serving over HTTPS.
- Tesla tokens are stored encrypted in `data/automator.db`. Back up `data/` and keep it private.
