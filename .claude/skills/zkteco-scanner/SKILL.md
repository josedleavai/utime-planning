---
name: zkteco-scanner
description: Connecting ZKTeco fingerprint/face scanners (currently on ZKTeco BioTime 8 / "utime") to our PocketBase + SvelteKit attendance project. Use when writing or debugging the /iclock scanner routes (ADMS/PUSH protocol), the BioTime sync, punch/employee/device/command collections, the fake-scanner test script, or the scanner cut-over from BioTime.
---

# ZKTeco scanners → PocketBase + SvelteKit

## The situation

- **Today:** scanners → ZKTeco **BioTime 8** on a Windows server (Apache + Django + bundled
  PostgreSQL, web/scanner port usually `8081`). It is the live system and must not go down.
- **Goal:** our own app — **PocketBase** (JS hooks in `pb_hooks/`) + **SvelteKit** (JS,
  `adapter-static`, served from `pb_public/`) — deployed via GitHub Actions + Docker to our
  Linux server (a different network from the scanners, routed internally).
- Scanners hold their own users/fingerprints/faces; switching servers does not wipe them.

## Rollout phases (respect the current phase)

1. **Shadow (local dev):** app reads employees + punches from the **BioTime API** on a cron.
   Scanners untouched. `/iclock` routes tested only with the fake scanner (see below).
2. **Shadow (deployed):** same, running on the Linux server. BioTime is still the source of truth.
3. **Cut-over:** one scanner's *COMM → Cloud Server* setting is pointed at the Linux server,
   then the rest. `SYNC_ENABLED=false`. Rollback = point scanner back at `<windows-ip>:8081`.

## Hard rules

- **Never write to BioTime** (API or DB). Read-only, always.
- **Never connect to BioTime's bundled Postgres** unless the API is unavailable; if so, use a
  dedicated read-only role.
- `/iclock/*` routes are **public** (scanners can't authenticate) → only accept requests whose
  `SN` is in the `devices` collection (or `ALLOWED_DEVICE_SNS`). Unknown SN → still reply `OK`
  but store nothing (and log it), so a misconfigured scanner doesn't retry-storm.
- Scanners speak **plain HTTP** and expect **plain-text** replies, never JSON. Keep `/iclock`
  off the public internet; the staff UI uses HTTPS as normal.
- Scanner timestamps have **no timezone**. Convert with `SCANNER_UTC_OFFSET` before saving to a
  PocketBase date field (stored as UTC), and keep the raw local string too.
- Punch inserts must be **idempotent** — scanners resend. Unique index on
  `(device_sn, pin, punch_time_local)`; swallow the unique-violation error.
- Always answer a scanner upload with `OK` once it is stored (or deduped). A non-OK reply makes
  the scanner resend forever.

## Env vars

| Var | Example | Notes |
|---|---|---|
| `BIOTIME_URL` | `http://192.168.1.20:8081` | Windows server IP + BioTime port |
| `BIOTIME_USER` / `BIOTIME_PASSWORD` | `api_sync` / … | dedicated BioTime web user |
| `SYNC_ENABLED` | `true` | `false` after cut-over |
| `SCANNER_UTC_OFFSET` | `+08:00` | offset of the scanners' local clock |
| `ALLOWED_DEVICE_SNS` | `ABCD123456,EFGH789012` | optional bootstrap allowlist |

Read them in hooks with `$os.getenv("NAME")`. Secrets live in the server `.env` / GitHub
Actions secrets, never in git. Provide a `.env.example`.

## Collections

| Collection | Key fields |
|---|---|
| `users` (auth) | portal logins (admins/HR) — separate from employees |
| `employees` | `pin` (text, unique — the ID on the scanner), `name`, `card`, `privilege`, `active`, `biotime_id` |
| `devices` | `sn` (unique), `alias`, `ip`, `last_seen` (date), `attlog_stamp`, `operlog_stamp`, `enabled` |
| `punches` | `device_sn`, `pin`, `punch_time_local` (text), `punch_at` (date, UTC), `status` (number), `verify` (number), `workcode`, `source` (`scanner`/`biotime`) — **unique (device_sn, pin, punch_time_local)** |
| `biometrics` | `pin`, `type` (`fp`/`face`), `index`, `size`, `template` (text, base64), `device_sn` |
| `device_commands` | `device_sn`, `seq` (number), `command` (text), `status` (`pending`/`sent`/`done`/`failed`), `return_code`, `sent_at`, `done_at` |

Create collections in the local PocketBase admin UI; PocketBase writes `pb_migrations/*.js`
automatically — commit those. Production applies them on start.

## PocketBase JS hooks — patterns and gotchas

- Route syntax (PocketBase ≥ 0.23): `routerAdd("POST", "/iclock/cdata", (e) => { ... })`.
- Query param: `e.request.url.query().get("SN")`.
- Raw body: `toString(e.request.body)` (older builds: `readerToString(e.request.body)`). Do **not**
  call `e.requestInfo()` first — it consumes the body.
- Reply: `return e.string(200, "OK")`.
- **Each handler runs in an isolated context** — top-level functions/variables in the file are
  NOT visible inside handlers. Put helpers in `pb_hooks/lib/iclock.js` and inside each handler do
  `const lib = require(`${__hooks}/lib/iclock.js`)`.
- Records: `new Record($app.findCollectionByNameOrId("punches"))`, `rec.set(k, v)`, `$app.save(rec)`.
- Lookups: `$app.findFirstRecordByFilter("devices", "sn = {:sn}", { sn })` (throws if none);
  `$app.findRecordsByFilter("device_commands", "device_sn = {:sn} && status = 'pending'", "seq", 20, 0, { sn })`.
- Cron (BioTime sync): `cronAdd("biotime-sync", "* * * * *", () => { ... })`.
- Outbound HTTP: `$http.send({ url, method, headers, body, timeout })` → `res.statusCode`, `res.json`.

Skeleton:

```js
// pb_hooks/iclock.pb.js
routerAdd("GET", "/iclock/cdata", (e) => {
  const lib = require(`${__hooks}/lib/iclock.js`)
  const sn = e.request.url.query().get("SN")
  const device = lib.touchDevice(sn, e.request.remoteAddr) // null if not allowed
  return e.string(200, device ? lib.handshake(sn, device) : "OK")
})

routerAdd("POST", "/iclock/cdata", (e) => {
  const lib = require(`${__hooks}/lib/iclock.js`)
  const q = e.request.url.query()
  const sn = q.get("SN")
  const body = toString(e.request.body)
  if (lib.touchDevice(sn, e.request.remoteAddr)) lib.ingest(sn, q.get("table"), body)
  return e.string(200, "OK")
})

routerAdd("GET", "/iclock/getrequest", (e) => {
  const lib = require(`${__hooks}/lib/iclock.js`)
  const sn = e.request.url.query().get("SN")
  if (!lib.touchDevice(sn, e.request.remoteAddr)) return e.string(200, "OK")
  return e.string(200, lib.nextCommands(sn) || "OK")
})

routerAdd("POST", "/iclock/devicecmd", (e) => {
  const lib = require(`${__hooks}/lib/iclock.js`)
  const sn = e.request.url.query().get("SN")
  lib.commandResults(sn, toString(e.request.body))
  return e.string(200, "OK")
})

// some firmwares also hit /iclock/ping, /iclock/registry, etc.
routerAdd("GET", "/iclock/ping", (e) => e.string(200, "OK"))
```

Protocol details (handshake text, line formats, command syntax, status/verify codes) are in
[protocol.md](protocol.md). BioTime API calls for the shadow phase are in
[biotime-api.md](biotime-api.md).

## Testing without a real scanner

Run PocketBase locally (`./pocketbase serve`), add a device with `sn=TEST001`, then:

```bash
B=http://127.0.0.1:8090
curl "$B/iclock/cdata?SN=TEST001&options=all&pushver=2.4.1"            # handshake
printf '1001\t2026-10-13 08:01:22\t0\t1\t0\t0\t0\n' | \
  curl -X POST --data-binary @- "$B/iclock/cdata?SN=TEST001&table=ATTLOG&Stamp=1"   # punch
curl "$B/iclock/getrequest?SN=TEST001"                                  # pending commands
printf 'ID=1&Return=0&CMD=DATA\n' | \
  curl -X POST --data-binary @- "$B/iclock/devicecmd?SN=TEST001"        # command result
```

Send the same punch twice → exactly one record. Keep these as `tools/fake-scanner.js` (Node,
no deps) so it can run in CI against a PocketBase container.

## Cut-over checklist

1. From a PC on the scanners' network: `curl http://<linux-server>:<port>/iclock/ping` → `OK`.
2. `devices` record exists for the scanner's SN (Menu → System Info → Device Info).
3. Scanner: COMM → Cloud Server → server address/port → save; watch `devices.last_seen` update.
4. Punch on the scanner → appears in `punches` within ~10 s, with the correct local time.
5. Queue `DATA QUERY ATTLOG ...` for the last few days if anything was missed during the switch.
6. Repeat per scanner; then `SYNC_ENABLED=false`. Keep BioTime installed but idle as fallback.
