# ZKTeco ADMS / PUSH protocol (what the scanner sends and expects)

The scanner is always the client. It polls the server over plain HTTP; the server never
connects to the scanner. Every request carries `?SN=<serial>`. All bodies are plain text,
fields separated by **TAB** (`\t`), records separated by newline. Firmware versions differ —
log unknown `table=` values and unknown lines instead of failing.

## 1. Handshake — `GET /iclock/cdata?SN=XXX&options=all&pushver=...`

Sent at boot and periodically. Reply with options (one per line):

```
GET OPTION FROM: XXX
ATTLOGStamp=None
OPERLOGStamp=9999
ATTPHOTOStamp=None
ErrorDelay=30
Delay=10
TransTimes=00:00;14:05
TransInterval=1
TransFlag=TransData AttLog OpLog AttPhoto EnrollUser ChgUser EnrollFP ChgFP UserPic
TimeZone=8
Realtime=1
Encrypt=None
```

- `ATTLOGStamp` / `OPERLOGStamp`: last stamp the server has; the scanner uploads records newer
  than it. `None` (or `0`) = send everything it has → only use for a deliberate backfill; once
  running, return the stamp saved in `devices.attlog_stamp` / `operlog_stamp`.
- `Delay`: seconds between `getrequest` polls. `Realtime=1`: upload each punch immediately.
- `TimeZone`: hours offset (keep in sync with `SCANNER_UTC_OFFSET`).
- Save `Stamp=` from each upload's query string into the device record.

## 2. Uploads — `POST /iclock/cdata?SN=XXX&table=<TABLE>&Stamp=<n>`

Reply `OK` after storing (some firmwares accept `OK: <count>`; plain `OK` is safe).

### `table=ATTLOG` — punches, one per line
```
<PIN>\t<YYYY-MM-DD HH:MM:SS>\t<Status>\t<Verify>\t<WorkCode>\t<Reserved>\t<Reserved>
1001\t2026-10-13 08:01:22\t0\t1\t0\t0\t0
```
- Status (punch state): `0` check-in, `1` check-out, `2` break-out, `3` break-in,
  `4` OT-in, `5` OT-out. Many sites leave it at `0` and derive in/out from time order.
- Verify: `0` password, `1` fingerprint, `2`/`4` card, `15` face (codes vary by model —
  store the raw number, map for display only).
- Time is the scanner's **local** clock, no timezone.

### `table=OPERLOG` — users, templates and admin log, mixed lines; first word is the type
```
USER PIN=1001\tName=Juan Cruz\tPri=0\tPasswd=\tCard=\tGrp=1\tTZ=0000000000000000\tVerify=0
FP PIN=1001\tFID=6\tSize=1184\tValid=1\tTMP=<base64>
FACE PIN=1001\tFID=0\tSize=...\tValid=1\tTMP=<base64>
OPLOG 4\t0\t2026-10-13 08:00:00\t0\t0\t0\t0
```
- `USER` → upsert `employees` by `pin`. `Pri=14` is device admin.
- `FP` / `FACE` → upsert `biometrics` by (`pin`, `type`, `FID`). `TMP` is an opaque blob —
  store as-is; it can be sent back to other scanners of the same algorithm version.
- `OPLOG` → device admin actions; store raw or ignore.

### Newer firmware (push protocol 3.x)
May send `table=USERINFO`, `table=BIODATA` (all biometrics, with `Type=` field: 1 fingerprint,
9 visible-light face, etc.) or `table=tabledata&tablename=...`. Same `key=value\t...` line style.
Parse generically: split on `\t`, then on the first `=`.

### `table=ATTPHOTO`
Binary JPEG with a header — ignore unless photo capture is needed.

## 3. Commands — `GET /iclock/getrequest?SN=XXX`

Polled every `Delay` seconds; also the scanner's heartbeat (update `last_seen`).
- Nothing to do → reply `OK`.
- Otherwise reply one or more lines `C:<id>:<command>`; `<id>` is our `device_commands.seq`.
  Mark them `sent`.

Common commands (fields TAB-separated):
```
C:101:DATA UPDATE USERINFO PIN=1001\tName=Juan Cruz\tPri=0\tPasswd=\tCard=\tGrp=1\tTZ=0000000000000000
C:102:DATA DELETE USERINFO PIN=1001
C:103:DATA UPDATE FINGERTMP PIN=1001\tFID=6\tSize=1184\tValid=1\tTMP=<base64>
C:104:DATA UPDATE BIODATA Pin=1001\tNo=0\tIndex=0\tValid=1\tDuress=0\tType=1\tMajorVer=10\tMinorVer=0\tFormat=0\tTmp=<base64>
C:105:DATA QUERY ATTLOG StartTime=2026-10-01 00:00:00\tEndTime=2026-10-13 23:59:59
C:106:CHECK
C:107:INFO
C:108:REBOOT
C:109:CLEAR LOG
C:110:SET OPTION Delay=10
```
- `CHECK` makes the scanner re-handshake and upload anything newer than its stamps.
- `DATA QUERY ATTLOG` re-uploads a date range via `POST cdata?table=ATTLOG` (backfill after a gap).
- `CLEAR LOG` / `CLEAR DATA` are destructive — require explicit admin confirmation in the UI.
- Use `FINGERTMP` on older firmware, `BIODATA` on push 3.x devices; check what the device
  uploaded to decide.
- Send a small batch per poll (e.g. ≤ 20 lines) — some devices have small buffers.

## 4. Command results — `POST /iclock/devicecmd?SN=XXX`

Body, one line per executed command:
```
ID=101&Return=0&CMD=DATA
ID=108&Return=0&CMD=REBOOT
```
- `Return=0` success → `done`; negative → `failed` (keep `return_code`).
- `INFO` results include extra `key=value` lines (firmware, user/FP counts) → store on the device.
- Reply `OK`.
- Commands `sent` but with no result after a few minutes → set back to `pending` (retry).

## 5. Other endpoints some firmwares call
`GET /iclock/ping`, `POST /iclock/registry`, `GET /iclock/push`, `POST /iclock/querydata` —
reply `OK` and log them; add handling only if a real device needs it.
