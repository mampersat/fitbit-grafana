# Google Health API Migration Guide

This guide explains how to migrate this project from the legacy Fitbit Web API to the Google Health API while keeping the same overall workflow (token bootstrap, periodic fetch, historical backfill, InfluxDB writes, Grafana dashboards).

## 1) What Changes in This Project

### API and OAuth changes

- Legacy Fitbit API base:
  - `https://api.fitbit.com`
- Google Health API base:
  - `https://health.googleapis.com/v4`

- Legacy token endpoint:
  - `https://api.fitbit.com/oauth2/token`
- Google token endpoint:
  - `https://oauth2.googleapis.com/token`

- User identity/profile endpoints:
  - Google profile: `GET /users/me/profile`
  - Google settings: `GET /users/me/settings`

- Intraday/detailed data endpoint style:
  - `GET /users/me/dataTypes/{dataType}/dataPoints`

### Data type naming rules

- In endpoint path: use **kebab-case**
  - Example: `heart-rate`, `daily-oxygen-saturation`
- In filter expressions: use **snake_case**
  - Example: `heart_rate`, `daily_oxygen_saturation`

### Scope changes

Google uses URL scopes. Use only what you need. Typical read-only scopes for this project:

- `https://www.googleapis.com/auth/googlehealth.activity_and_fitness.readonly`
- `https://www.googleapis.com/auth/googlehealth.health_metrics_and_measurements.readonly`
- `https://www.googleapis.com/auth/googlehealth.sleep.readonly`
- `https://www.googleapis.com/auth/googlehealth.location.readonly`
- `https://www.googleapis.com/auth/googlehealth.profile.readonly`
- `https://www.googleapis.com/auth/googlehealth.settings.readonly`

## 2) Configure Google Cloud (Client ID / Secret)

1. Open Google Cloud Console:
   - https://console.cloud.google.com
2. Create or select a project.
3. Enable Google Health API for the project from services.
4. Configure OAuth consent screen:
   - Set app information
   - Add required test users if app is in Testing mode
5. Create OAuth credentials:
   - APIs & Services -> Credentials -> Create Credentials -> OAuth client ID
   - Application type: typically Desktop app
6. Add redirect URI exactly:
   - `http://localhost:8080`
7. Save and copy:
   - `GOOGLE_CLIENT_ID`
   - `GOOGLE_CLIENT_SECRET`

## 3) Get Refresh Token Using Parity Tool Settings Menu

Use the parity tool to bootstrap OAuth and extract a refresh token.

- Parity tool:
  - https://developers.google.com/health/migration/parity-tool

### Steps

1. Open parity tool and click **Settings** (gear icon).
2. In OAuth configuration inside Settings:
   - Enter Client ID: your Google OAuth client id
   - Enter Client Secret: your Google OAuth client secret
   - Enter Redirect URI: `http://localhost:8080`
4. Trigger authentication/re-authentication from Settings.
5. Complete the Google consent flow in browser.
6. Return to Settings and copy the refresh token shown by the tool.
7. Get the code part from the url and paste it to get the Acces and Refresh token pair

Notes:

- If you see repeated Unauthorized errors in parity tool, re-authenticate from Settings.
- Refresh tokens can expire in some cases (for example long inactivity or testing-mode policy constraints).

## 4) Update Project Configuration

Use Google mode in `compose.yml` (or shell env vars for local runs).

Required environment variables:

- `HEALTH_API_PROVIDER=google`
- `GOOGLE_CLIENT_ID=<your_client_id>`
- `GOOGLE_CLIENT_SECRET=<your_client_secret>`
- `TOKEN_FILE_PATH=/app/tokens/fitbit.token` (or your preferred location)

Existing variables still required for database/write pipeline:

- `INFLUXDB_VERSION`
- `INFLUXDB_HOST`
- `INFLUXDB_PORT`
- `INFLUXDB_DATABASE` (and auth vars when applicable)

Optional for local validation without DB writes:

- `DRY_RUN_MODE=true`

## 5) First Run / Token Bootstrap

After setting env vars, run one interactive start:

```bash
HEALTH_API_PROVIDER=google \
GOOGLE_CLIENT_ID="<client_id>" \
GOOGLE_CLIENT_SECRET="<client_secret>" \
TOKEN_FILE_PATH="/tmp/google_health_test.token" \
FITBIT_LOG_FILE_PATH="/tmp/google_health_test.log" \
/home/arpan/Documents/fitbit-grafana/.venv/bin/python Fitbit_Fetch.py
```

When prompted, paste the refresh token obtained from parity tool Settings.

The script will refresh access token and persist token metadata to `TOKEN_FILE_PATH`.

## 6) Validate Historical Fetch

Example one-day historical run:

```bash
HEALTH_API_PROVIDER=google \
DRY_RUN_MODE=true \
GOOGLE_CLIENT_ID="<client_id>" \
GOOGLE_CLIENT_SECRET="<client_secret>" \
AUTO_DATE_RANGE=False \
MANUAL_START_DATE="2024-08-20" \
MANUAL_END_DATE="2024-08-20" \
TOKEN_FILE_PATH="/tmp/google_health_test.token" \
FITBIT_LOG_FILE_PATH="/tmp/google_health_test_2024.log" \
/home/arpan/Documents/fitbit-grafana/.venv/bin/python Fitbit_Fetch.py
```

Use `DRY_RUN_MODE=true` while validating API shape/parsing before writing to InfluxDB.

## 7) Common Migration Issues

### `INVALID_DATA_POINT_FILTER_*` errors

Cause:
- Unsupported filter member for a datatype.

Fix:
- Use datatype-appropriate filter members.
- Remember endpoint datatype naming != filter datatype naming:
  - endpoint: kebab-case
  - filter: snake_case

### API returns 200 but script inserts 0 points

Cause:
- Parser mismatch (nested payload/timestamp/value fields), or date mismatch due to timezone and civil vs physical time.

Fix:
- Confirm payload path for that datatype (`heartRate`, `steps`, `exercise`, `dailyOxygenSaturation`, etc.).
- Confirm filter uses supported field member.
- Verify timezone from `users/me/settings`.

### Unauthorized errors after working earlier

Cause:
- Expired/invalid token.

Fix:
- Re-authenticate from parity tool Settings.
- Re-run script and re-enter a valid refresh token.

## 8) Security Notes

- Never commit real client secrets or refresh tokens.
- If secrets were shared in logs/chat, rotate immediately in Google Cloud.
- Keep token file under protected permissions.

## 9) Migration Status Expectations

The migration in this repository is being rolled out in phases.

- Some Google datasets are already mapped and tested.
- Some grouped legacy fetch blocks may still be marked as not finalized in logs while parity work continues.

Use this guide together with project logs to validate each datatype path incrementally.