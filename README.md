# utime-planning

Planning notes for replacing ZKTeco BioTime ("utime") with our own PocketBase + SvelteKit
attendance app.

## Claude Code skill: `zkteco-scanner`

`.claude/skills/zkteco-scanner/` holds everything needed to connect the ZKTeco fingerprint
scanners to the new app: rollout phases, env vars, collections, PocketBase hook patterns, the
scanner (ADMS/PUSH) protocol and the BioTime read-only API.

To use it in another project, copy the folder into that repo:

```bash
mkdir -p /path/to/your-project/.claude/skills
cp -r .claude/skills/zkteco-scanner /path/to/your-project/.claude/skills/
```

Claude Code picks it up automatically in that project (restart the session if it was open) and
loads it when you work on the scanner integration. You can also call it directly with
`/zkteco-scanner`.
