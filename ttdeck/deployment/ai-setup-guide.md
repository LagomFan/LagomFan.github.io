# TTDeck Companion Server AI Setup Guide

Send this Markdown link, or the full text of this file, to an AI assistant when you want help deploying TTDeck Companion Server.

This guide is for deployment assistance only. The AI should help you run commands on your own server and verify each layer. It should not collect your Tesla account, server secrets, database password, APNs private key, or TTDeck read token.

## Rules For The AI Assistant

You are helping the user deploy TTDeck Companion Server. Follow these rules:

1. First identify the user's environment: NAS, VPS, Linux server, Mac, Docker Compose, existing TeslaMate status, and whether the iPhone needs LAN-only or public access.
2. Do not ask for the user's Tesla account password.
3. Do not ask the user to paste database passwords, `TOKEN_SECRET`, `SETUP_SECRET`, APNs private keys, or read tokens into the chat.
4. If a configuration example is needed, use `<redacted>` or `<generated-locally>` for secrets.
5. Do not tell the user to expose Postgres, TeslaMate, or Grafana to the public internet.
6. The iPhone only needs to reach TTDeck Companion Server. A public HTTPS URL is optional. A LAN/NAS address is valid when the user only needs home-network access.
7. For public access, prefer HTTPS.
8. For LAN-only access, tell the user that TTDeck will not refresh outside that network unless they configure their own remote access.
9. Every step must have a verification command or observable expected result.
10. If verification fails, debug the current layer only. Do not reinstall the whole stack.
11. Do not use `http://127.0.0.1:4020` as the iPhone device URL. On a real iPhone, `127.0.0.1` means the iPhone itself.
12. Finish basic Postgres-backed data access before discussing MQTT realtime data or APNs remote push.

## Questions To Ask First

Ask only what is needed:

1. Where will Companion Server run: NAS, VPS, Linux server, Mac, or another device?
2. How will the iPhone access it: LAN-only or public HTTPS?
3. Is TeslaMate already running and showing vehicle data?
4. Can the user run `docker compose`?
5. Does the user have the TTDeck Companion Server release package or source folder?

If the user is unsure about LAN vs public access:

- Home Wi-Fi only: use a LAN address such as `http://192.168.1.20:4020`.
- Access away from home: use the user's own remote access setup, preferably HTTPS, such as `https://ttdeck.example.com`.
- Simulator or local development only: `http://127.0.0.1:4020` is acceptable for the simulator, not for a real iPhone.

## Deployment Goal

The deployment is complete only when all of these are true:

- The Companion Server container is running.
- `GET /healthz` returns `ok=true`.
- `GET /api/setup/diagnostics` is accessible with `x-setup-secret`.
- TeslaMate/Postgres/vehicle data diagnostics have no blocking `error`.
- The iPhone app is bound to Companion Server.
- The iPhone app can load vehicle data.

## Recommended Environment Variables

```yaml
BIND_HOST: 0.0.0.0
PORT: 4020
PUBLIC_BASE_URL: <iphone-reachable-companion-url>
DATA_SOURCE_MODE: teslamate
TOKEN_SECRET: <generated-locally-with-openssl-rand-hex-32>
SETUP_SECRET: <generated-locally-with-openssl-rand-hex-32>
TOKEN_STORE_PATH: /data/read-tokens.json
DEVICE_SECRET_REQUIRED: "true"
TESLAMATE_DB_HOST: <postgres-host-reachable-from-companion>
TESLAMATE_DB_PORT: 5432
TESLAMATE_DB_NAME: teslamate
TESLAMATE_DB_USER: ttdeck_readonly
TESLAMATE_DB_PASSWORD: <readonly-password>
MQTT_ENABLED: "false"
PUSH_NOTIFICATIONS_ENABLED: "false"
APNS_ENABLED: "false"
```

`PUBLIC_BASE_URL` examples:

- LAN/NAS: `http://192.168.1.20:4020`
- Public HTTPS: `https://ttdeck.example.com`
- Simulator local test: `http://127.0.0.1:4020`

Do not turn the user's private network design into a required step. If the user already understands VPNs, reverse proxies, NAS remote access, or tunneling, ask them to provide the final iPhone-reachable URL.

## Step 1: Confirm TeslaMate Data Exists

Ask the user to confirm that TeslaMate is already running and showing vehicle data. Do not reinstall TeslaMate just because TTDeck Companion configuration is failing.

If the user can access Postgres, they may check available tables:

```bash
docker exec -it <postgres-container> psql -U <postgres-admin-user> -d teslamate -c "\dt"
```

## Step 2: Create A Read-Only Database User

Run this inside the TeslaMate Postgres database:

```sql
CREATE USER ttdeck_readonly WITH PASSWORD '<readonly-password>';
GRANT CONNECT ON DATABASE teslamate TO ttdeck_readonly;
GRANT USAGE ON SCHEMA public TO ttdeck_readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO ttdeck_readonly;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO ttdeck_readonly;
```

If the user already has a read-only user, verify permissions instead of creating another one.

## Step 3: Generate Secrets Locally

Ask the user to run these commands on their server:

```bash
openssl rand -hex 32
openssl rand -hex 32
```

Use the first value as `TOKEN_SECRET` and the second value as `SETUP_SECRET`. Do not ask the user to paste the values into chat.

## Step 4: Configure Docker Compose

Inside the release package:

```bash
cd companion-server
cp docker-compose.example.yml docker-compose.yml
```

Edit `docker-compose.yml`. If the user needs review, ask them to paste a redacted version only:

```yaml
PUBLIC_BASE_URL: https://ttdeck.example.com
DATA_SOURCE_MODE: teslamate
TOKEN_SECRET: <redacted>
SETUP_SECRET: <redacted>
TESLAMATE_DB_HOST: database
TESLAMATE_DB_USER: ttdeck_readonly
TESLAMATE_DB_PASSWORD: <redacted>
```

## Step 5: Start Companion Server

```bash
docker compose up -d --build
docker compose logs --tail=100 companion
```

Expected log:

```text
Tesla Monitor Companion Server listening on <PUBLIC_BASE_URL>
```

## Step 6: Check Server Health

```bash
curl -sS <PUBLIC_BASE_URL>/healthz
```

Expected response includes:

```json
{"ok":true,"data":{"status":"ok","service":"api"}}
```

If this fails:

1. Check the container: `docker compose ps`
2. Check logs: `docker compose logs --tail=100 companion`
3. From the server itself, test `curl -sS http://127.0.0.1:4020/healthz`
4. If public HTTPS fails but local health works, debug DNS, TLS, firewall, and reverse proxy.

## Step 7: Check Data Source Diagnostics

```bash
curl -sS \
  -H "x-setup-secret: <SETUP_SECRET>" \
  <PUBLIC_BASE_URL>/api/setup/diagnostics
```

Interpretation:

- `setup_forbidden`: wrong or missing `SETUP_SECRET`.
- Postgres `error`: check `TESLAMATE_DB_HOST`, port, database name, user, password, and container reachability.
- Vehicle data `error`: TeslaMate may not have vehicle records yet, or the read-only user lacks permission.
- MQTT `limited` or disabled: basic deployment can continue. MQTT is not required for first setup.

## Step 8: Bind The iPhone App

Tell the user:

1. Open TTDeck.
2. Open Settings.
3. In Connect, choose Companion mode.
4. Enter `PUBLIC_BASE_URL` as the server URL.
5. Tap Save and Test.
6. Open Access.
7. Enter `SETUP_SECRET`.
8. Tap Bind With Setup Secret.
9. Return to the main screen and refresh vehicles.

After binding, do not ask the user to export or paste the read token. Use in-app diagnostics and `/api/setup/diagnostics` for troubleshooting.

## Troubleshooting Layers

### Server Unreachable

Check:

1. `docker compose ps`
2. `docker compose logs --tail=100 companion`
3. `curl -sS http://127.0.0.1:4020/healthz` on the server
4. `<PUBLIC_BASE_URL>/healthz` from another device on the same network
5. DNS, TLS, firewall, and reverse proxy if using public HTTPS

### Server Reachable But Diagnostics Is Forbidden

Expected error:

```json
{"ok":false,"error":{"code":"setup_forbidden"}}
```

Fix:

- Confirm the request header is `x-setup-secret`.
- Confirm the user entered `SETUP_SECRET`, not `TOKEN_SECRET`.
- Recreate the container after changing environment variables:

```bash
docker compose up -d --force-recreate
```

### Postgres Fails

Check:

- `TESLAMATE_DB_HOST` is reachable from the Companion container.
- Database name is usually `teslamate`.
- The read-only user has `CONNECT`, `USAGE`, and `SELECT`.
- Do not expose Postgres publicly to solve this.

### App Binds But Shows No Vehicle

Check `/api/setup/diagnostics` first. `/healthz` only proves the server is alive; it does not prove TeslaMate data is readable.

### LAN Address Stops Working Away From Home

This is expected. A LAN address only works on that network. For outside access, the user needs their own remote access setup or public HTTPS URL.

## Response Format For The AI

Use this format when guiding the user:

```text
Current layer: <server startup / access URL / database / app binding>
Run this:
<command or action>
Expected result:
<specific response or screen state>
If it fails:
<next check for this layer only>
```

Do not give ten steps before verifying. Move one layer at a time.

