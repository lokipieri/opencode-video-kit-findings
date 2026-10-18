# Post-Reboot Addendum — Repairs & Live Verification

**Date:** 2026-10-18 · **Kit:** opencode-video-kit (extracted to `C:\Users\User\_kits\opencode-video-kit`)
**Base report:** `SETUP-REPORT.md` — Verdict: PASS, 34/34 recorded steps
**Status after this addendum:** all 7 MCP servers connected and answering real tool calls.

## 1. Repairs applied beyond the kit

| # | Problem observed | Fix applied |
|---|---|---|
| 1 | `ffmpeg-mcp-lite` fails against `mcp` 2.x (`ModuleNotFoundError: mcp.server.fastmcp`) | `uv tool install ffmpeg-mcp-lite --force --with "mcp<2"` → 8 tools live |
| 2 | Two npx caches corrupted when first-boot downloads were killed at the ~50 s MCP connect timeout (`ERR_MODULE_NOT_FOUND`; missing `ajv-formats`) | Deleted the two folders under `%LOCALAPPDATA%\npm-cache\_npx\` (context7, sequential-thinking), then clean re-download |
| 3 | `donsetch` declared in the kit config but never installed by any stage | Installed from npm (`donsetch@4.10.1`; postinstall fetched its checksum-verified binary) → 4 tools live |
| 4 | Stage-05 `verify-all.ps1` aborts on native-command stderr under `$ErrorActionPreference='Stop'` (kit bug) | Patched the native call sites (relax-EAP `try/finally`); original kept as `verify-all.ps1.orig` |
| 5 | Piper voice: single hardcoded URL, no retry/checksum fallback | Voice prefetched manually (`en_US-lessac-medium.onnx`, 63,201,294 B + JSON) |

Cleanup: the Desktop helper `restart-opencode-fresh.cmd` (used to restart app + daemon with a fresh PATH before the reboot) was deleted after the reboot made it obsolete.

## 2. Live verification (after restart, 18:22 local)

| Server | Tools | Evidence |
|---|---|---|
| `rea` | 207 | `opencode.log` connect wave + live tool catalog |
| `github` | 26 | `opencode.log` + catalog |
| `ffmpeg-mcp-lite` | 8 | log + catalog + **live call**: ffprobe of the smoke clip → 5.000 s, 66,284 B |
| `donsetch` | 4 | log + catalog |
| `whisper-transcribe-mcp` | 3 | log + catalog |
| `context7` | 2 | log + catalog + **live call**: resolve-library-id → FFmpeg docs (3,843 snippets) |
| `sequential-thinking` | 1 | log + catalog + **live call**: structured round trip OK |

Additional checks: kit test suite **ALL TESTS PASS** (8 files) · Piper synthesis produced a ~108 KB WAV · ffmpeg rendered a 5 s clip · all 9 command-line tools resolve in a fresh shell.

## 3. Playbook (if anything misbehaves later)

- **Diagnose MCP:** inspect `%USERPROFILE%\.local\share\opencode\log\opencode.log` for `mcp connect failed` / `mcp transport error`.
- **npx-based servers** (rea, github, context7, sequential-thinking): if one stops connecting, delete its folder under `%LOCALAPPDATA%\npm-cache\_npx\` and restart the app — the next run re-downloads (give it a minute).
- **ffmpeg-mcp-lite:** keep the `mcp<2` pin. Reinstall: `uv tool install ffmpeg-mcp-lite --force --with "mcp<2"`.
- **donsetch:** reinstall with the script allowed: `npm install -g --allow-scripts=donsetch --foreground-scripts donsetch`.
- **npm install-scripts gate:** this machine's npm asks before running package install scripts; when a reinstall needs one, pass `--allow-scripts=<pkg> --foreground-scripts` (as was done for donsetch).
- **Full app + daemon restart** (only if ever needed): quit opencode fully, verify no `opencode-cli.exe serve --service` process remains, then relaunch.
- **Remotion studio:** `cd 03-video-tooling\remotion` then `npm run studio` (dependencies already installed).

## 4. Upstream findings

Kit-author bug list is in `UPSTREAM-FINDINGS.md` (same folder): 9 findings, each reproduced on this machine. No upstream repository URL ships with the kit, so they are stored here, ready to send.

## 5. Addendum update (later the same day)

- GitHub CLI 2.102.0 installed; logged in as lokipieri.
- The 9 upstream findings are filed as issues: https://github.com/lokipieri/opencode-video-kit-findings
- GITHUB_PERSONAL_ACCESS_TOKEN (from GitHub CLI) and CONTEXT7_API_KEY are set as Windows user environment variables; both activate for the MCP servers on the next opencode restart.