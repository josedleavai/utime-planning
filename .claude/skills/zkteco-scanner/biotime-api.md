# BioTime 8 API (read-only, shadow phase)

Base: `BIOTIME_URL` (e.g. `http://192.168.1.20:8081`). Requires the API to be available on the
BioTime license; if these return 404 / license errors, fall back to a read-only Postgres role on
BioTime's DB (tables `personnel_employee`, `iclock_transaction`, `iclock_terminal`).
**Never call POST/PUT/PATCH/DELETE on anything except the token endpoint.**

Field names below are typical for BioTime 8 — confirm against a real response before relying on
them, and log unexpected shapes instead of crashing.

## Auth
```bash
curl -X POST "$BIOTIME_URL/jwt-api-token-auth/" \
  -H "Content-Type: application/json" \
  -d '{"username":"api_sync","password":"..."}'
# → {"token":"eyJ..."}
```
Header on every call: `Authorization: JWT <token>`. Cache the token; on 401 re-login once.
(Some installs use `POST /api-token-auth/` + `Authorization: Token <token>` instead.)

## Employees
`GET /personnel/api/employees/?page=1&page_size=100`
→ `{ count, next, previous, data: [ { id, emp_code, first_name, last_name, card_no, department, ... } ] }`
Map `emp_code` → `employees.pin`, `id` → `employees.biotime_id`. Follow `next` until null.

## Transactions (punches)
`GET /iclock/api/transactions/?start_time=2026-10-13 00:00:00&end_time=2026-10-13 23:59:59&page_size=200`
→ `data: [ { id, emp_code, punch_time, punch_state, verify_type, terminal_sn, terminal_alias, ... } ]`
Map to `punches`: `terminal_sn` → `device_sn`, `emp_code` → `pin`, `punch_time` →
`punch_time_local`, `punch_state` → `status`, `verify_type` → `verify`, `source = "biotime"`.
The same unique index dedupes these against punches later received directly from scanners.

## Devices
`GET /iclock/api/terminals/` → `data: [ { sn, alias, ip_address, last_activity, ... } ]`
Use to pre-fill the `devices` collection (SN allowlist) before cut-over.

## Sync job (PocketBase cron)
- Every minute when `SYNC_ENABLED=true`: fetch transactions from `last_synced - 10 min` to now
  (overlap is safe thanks to the unique index), then store `last_synced` in a `settings` record.
- Employees: hourly is enough.
- Timeouts ≤ 10 s; a failed sync just logs and tries next minute — never blocks `/iclock` routes.
