# Kick-off prompt for the new attendance project

Before you start, copy the skill into the new project and start Claude Code there:

    git clone -b claude/biometrics-scanner-portal-config-g5qhqc https://github.com/josedleavai/utime-planning.git /tmp/utime-planning
    mkdir -p /path/to/new-project/.claude/skills
    cp -r /tmp/utime-planning/.claude/skills/zkteco-scanner /path/to/new-project/.claude/skills/
    cd /path/to/new-project && claude

Fill in the `<...>` placeholders you already know, then paste everything below the line as
your first message.

---

# Task: build our attendance app (PocketBase + SvelteKit) to replace ZKTeco BioTime 8

## Context
- Today our ZKTeco fingerprint/face scanners push to **ZKTeco BioTime 8** ("utime") on a Windows server (`<WINDOWS_IP>`). BioTime bundles its own Apache, Django and PostgreSQL. Its scanner/web port is usually `8081`. The scanners use the ADMS/PUSH ("iclock") HTTP protocol: they are always the client and poll the server. BioTime is live and must keep running untouched until the cut-over.
- We are replacing it with our own app in THIS repo:
  - Backend: **PocketBase** with JS hooks in `pb_hooks/`. Migrations live in `pb_migrations/` and are committed.
  - Frontend: **SvelteKit in plain JavaScript** (not TypeScript) with `adapter-static`. It builds into `pb_public/` and PocketBase serves it.
  - Delivery: GitHub → GitHub Actions builds a Docker image → pushes to GHCR → deploys over SSH to our Linux server (`<LINUX_HOST>`) with `docker compose pull && docker compose up -d`. `pb_data/` lives on a volume and is backed up daily.
- The Linux server is on a different internal network from the scanners and the Windows server. The networks are routed and can reach each other.
- Domain knowledge is in the **`zkteco-scanner` skill** at `.claude/skills/zkteco-scanner/` (`SKILL.md`, `protocol.md`, `biotime-api.md`). Read all three before writing any scanner or sync code. If the folder is missing, stop and tell me.

## Non-negotiable rules
1. **Never write to BioTime**, whether through its API or its database. Only GET requests are allowed, plus the token login. Don't connect to its Postgres unless I approve it, and then only with a read-only role.
2. Don't contact the real BioTime, change scanner settings, push to `main` or deploy without asking me first.
3. The `/iclock/*` routes are public because scanners can't authenticate:
   - Only accept serial numbers that are in the `devices` collection or `ALLOWED_DEVICE_SNS`.
   - An unknown SN still gets `OK`, but store nothing and log it.
   - Replies are plain text, never JSON.
   - Scanners speak **plain HTTP only**. The `/iclock` path must not be redirected to HTTPS, and it should be reachable only from the scanner subnets.
4. Scanner timestamps have no timezone:
   - Keep the raw local string in `punch_time_local`.
   - Convert it to UTC for `punch_at` using `SCANNER_UTC_OFFSET` (e.g. `+08:00`).
   - Never rely on the server's own timezone.
5. Punches must be idempotent. Put a unique index on `(device_sn, pin, punch_time_local)` and ignore duplicate errors. Always answer scanner uploads with `OK`, otherwise the scanner resends forever.
6. Scanner firmware varies. Log unknown tables, lines and fields raw instead of crashing, and tell me about them rather than guessing.
7. No secrets in git. Provide a `.env.example`. Real values go in the server `.env` and in GitHub Actions secrets.

## Before writing code
1. Explore this repo and summarise its existing structure and conventions. I have my own codebase layout: fit into it rather than imposing a new one. Propose where these go:
   - `pb_hooks/` and `pb_migrations/`
   - the SvelteKit app
   - `tools/`, the `Dockerfile`, `docker-compose.yml` and `.github/workflows/`
2. Pin a PocketBase version (latest stable unless the repo already has one). Check its JSVM API against the official docs, because it changed in v0.23. Check in particular:
   - `routerAdd`, `e.request.url.query()`, reading the raw body (`toString(e.request.body)`) and `e.string()`
   - `require(\`${__hooks}/...\`)`: handlers run in isolated contexts
   - `$app.save`, `findFirstRecordByFilter`, `cronAdd`, `$http.send`, `$os.getenv`
3. Give me a milestone plan based on the list below, adjusted to the repo, and **wait for my OK**.

## Milestones (stop after each one for my review; commit each)
- **M1 Scaffold.**
  - A script that downloads the pinned PocketBase binary.
  - `pb_hooks/` and `pb_migrations/`.
  - The SvelteKit app using `adapter-static` with output to `pb_public/`.
  - A dev setup that runs `pocketbase serve` and `vite dev` together, with Vite proxying `/api` and `/iclock` to `:8090`.
  - `.env.example`, and a `.gitignore` covering `pb_data`, `.env` and the binary.
  - A README with the run steps.
- **M2 Collections, as migration files.**
  - Collections: `users` (auth, with role `admin`/`hr`), `employees`, `devices`, `punches`, `biometrics`, `device_commands`, and a `settings` collection (key/value, for things like `last_synced`). Fields are as in the skill.
  - Unique indexes.
  - API rules: logged-in staff only, nothing public, and `biometrics.template` never exposed to the UI.
- **M3 Scanner endpoints** in `pb_hooks/iclock.pb.js`, with helpers in `pb_hooks/lib/iclock.js`:
  - handshake, using the stamps saved per device
  - ATTLOG ingest
  - OPERLOG `USER`/`FP`/`FACE` lines, plus push 3.x `USERINFO`/`BIODATA`
  - the `getrequest` command queue, at most 20 lines per poll
  - `devicecmd` results
  - `ping` and the other endpoints, which reply `OK` and log
  - `last_seen` heartbeat, SN allowlist and UTC conversion
  - a cron that resets `sent` commands to `pending` when they get no result within 5 minutes
  - an env toggle `ICLOCK_DEBUG=true` that logs every raw request
- **M4 Fake scanner.** `tools/fake-scanner.js` (Node, zero dependencies) hits every endpoint against a running PocketBase and checks that:
  - the handshake text is correct
  - the same punch sent twice gives exactly one record, with the correct UTC `punch_at`
  - an unknown SN gets `OK` and nothing is stored
  - a USER line upserts the employee
  - a command goes pending → sent → done
- **M5 BioTime sync, read-only.**
  - `pb_hooks/biotime-sync.pb.js`, guarded by `SYNC_ENABLED`:
    - JWT login with a cached token and one retry on 401
    - employees hourly
    - punches every minute with a 10-minute overlap, saving `last_synced` in `settings`
    - `source="biotime"`
    - timeouts of 10 s or less; on failure, log and wait for the next run, never blocking `/iclock`
  - `tools/biotime-probe.js`, which logs in and prints one page each of employees, transactions and terminals. I'll run it against the real BioTime so we can confirm the field names before relying on them.
- **M6 UI** (SvelteKit + PocketBase JS SDK):
  - login
  - a live "today" punch board using realtime subscriptions
  - employees
  - devices, showing online/offline from `last_seen`
  - a punch report with a date range, employee/device filters and CSV export
  - device actions:
    - push or delete a user on a scanner, and copy fingerprints from one scanner to another
    - `CHECK`, and a `DATA QUERY ATTLOG` date-range backfill
    - `REBOOT` and `CLEAR LOG`, both behind a typed confirmation
- **M7 CI/CD.**
  - A multi-stage `Dockerfile`: build SvelteKit with Node, then an alpine image with the pinned PocketBase, `pb_hooks`, `pb_migrations` and `pb_public`.
  - `docker-compose.yml` with the `pb_data` volume, `env_file: .env`, a healthcheck on `/api/health` and a restart policy.
  - GitHub Actions:
    - On PRs: build the image, start it, and run `tools/fake-scanner.js` against it.
    - On `main`: push to GHCR and deploy over SSH.
  - The list of required secrets: `SSH_HOST`, `SSH_USER`, `SSH_KEY` and the GHCR auth.
  - A daily `pb_data` backup script/cron with retention.
- **M8 Runbook.** `docs/cutover.md` with our real values, built from the skill's checklist:
  - network test from the scanner subnet
  - one pilot scanner first, then the rest
  - set `SYNC_ENABLED=false`
  - rollback: point the scanner back to `<WINDOWS_IP>:8081`

## Env values (I'll provide the ones I know; ask me for the rest)
| Var | Value | Where to find it |
|---|---|---|
| `BIOTIME_URL` | `http://<WINDOWS_IP>:<PORT>` | The `Listen` line in `<BioTime>\apache\conf\httpd.conf` (or `attsite.ini`). This is also the "Server Address/Port" in the scanner's COMM → Cloud Server menu. |
| `BIOTIME_USER` / `BIOTIME_PASSWORD` | `<...>` | A dedicated user I create in the BioTime web UI. Not the superadmin. |
| `SYNC_ENABLED` | `true` | Set to `false` after the cut-over. |
| `SCANNER_UTC_OFFSET` | `<+08:00>` | The scanners' local timezone. |
| `ALLOWED_DEVICE_SNS` | `<SN1,SN2>` | BioTime → Device page, or on the scanner under Menu → System Info → Device Info. |
| `ICLOCK_DEBUG` | `false` | Set to `true` while testing real firmware. |

## How to work
- Small, focused commits. Run the fake scanner after every hook change, and run the build before saying something works.
- If PocketBase docs, BioTime field names or firmware behaviour disagree with the skill, trust what you actually observe. Then tell me so I can update the skill.
- Keep the code plain and readable. This is an internal tool maintained by a small team.
