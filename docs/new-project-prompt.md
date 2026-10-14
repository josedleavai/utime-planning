# Prompt for adding the scanner feature to an existing project

This is for a project that already has its own PocketBase + SvelteKit stack, structure and
CI/CD. The prompt only says what to build. Claude works out where it goes from your codebase.

First, copy the skill into the project:

    git clone -b claude/biometrics-scanner-portal-config-g5qhqc https://github.com/josedleavai/utime-planning.git /tmp/utime-planning
    mkdir -p .claude/skills && cp -r /tmp/utime-planning/.claude/skills/zkteco-scanner .claude/skills/

Then start `claude` in the project and paste the text below the line.

---

Add ZKTeco fingerprint scanner support to this project so it can replace ZKTeco BioTime 8 ("utime").

Use the `zkteco-scanner` skill (.claude/skills/zkteco-scanner/) for the protocol, rules and BioTime API. Read it fully first.

Fit everything into this project's existing structure, conventions and deploy setup. Don't add new tooling or change the stack. Start by looking at the codebase, then tell me where each piece will go and wait for my OK.

What to build, in order. Stop after each step so I can review it:
1. **Collections**: employees, devices, punches, biometrics, device_commands, and settings (for last_synced), as described in the skill. Only logged-in staff can access them.
2. **Scanner endpoints** (/iclock/*): the handshake, receiving punches and users/fingerprints, the command queue, and command results. Only accept known scanner serial numbers, and always reply with plain-text OK.
3. **Fake scanner test script**, so I can test without a real device. Sending the same punch twice must give exactly one record.
4. **Read-only BioTime sync**: employees hourly, punches every minute, switchable with SYNC_ENABLED. Also a small probe script I can run to check BioTime's real field names.
5. **UI pages**:
   - today's punches, updating live
   - employees
   - devices, showing online/offline
   - a punch report with CSV export
   - device actions (send a user to a scanner, backfill punches, reboot with a confirmation step)
6. **Cut-over runbook** (docs/cutover.md): move one pilot scanner first, then the rest, plus how to roll back.

Rules:
- Never write to BioTime. Don't contact the real BioTime or scanners, or deploy, without asking me first.
- Scanner times have no timezone. Convert them using SCANNER_UTC_OFFSET.
- Add the new env vars to the project's existing env setup (BIOTIME_URL, BIOTIME_USER, BIOTIME_PASSWORD, SYNC_ENABLED, SCANNER_UTC_OFFSET, ALLOWED_DEVICE_SNS). Ask me for their values.
- If something you observe disagrees with the skill, trust what you observe and tell me.
