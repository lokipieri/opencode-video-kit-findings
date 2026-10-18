# Upstream Findings — opencode-video-kit

Environment: Windows 11, PowerShell 5.1, fresh machine, opencode Desktop 2.0.32.
Kit extracted to `C:\Users\User\_kits\opencode-video-kit`; all 5 stages completed (Verdict: PASS).
Every finding below was reproduced during a clean run of the kit.

## 1. `verify-all.ps1` aborts on native-command stderr (stage 05 cannot complete unpatched)
- Stage-05 runs `& ffmpeg @genArgs 2>&1` and `& whisper-transcribe-mcp $clip 2>&1` under
  `$ErrorActionPreference='Stop'`. PowerShell 5.1 converts stderr output from native
  commands into a terminating `NativeCommandError`, so the verifier aborts on harmless
  ffmpeg banner output.
- Observed: first stage-05 run aborted mid-suite. After a relax-EAP patch (`try/finally`
  around the native calls), all 34 steps passed.
- Suggested fix: keep the native calls wrapped in try/finally with relaxed EAP, or on
  PS7 set `$PSNativeCommandUseErrorActionPreference = $false`.

## 2. `donsetch` is declared but never installed → a dead MCP server on every fresh install
- `01-core-config/config-payload.json` ships a `donsetch` MCP entry and the docs treat it
  as keyless, but no stage installs the package. The stage-05 "MCP resolution" check still
  passes because it only resolves `command[0]` (`cmd.exe`), not the real payload. Result:
  the kit reports PASS while one of its 7 MCP servers is dead.
- Observed: `donsetch` absent after stages 01–05; installed manually from npm
  (`donsetch@4.10.1`; its postinstall fetches a checksum-verified platform binary).
- Suggested fix: add an install step for it (with install scripts allowed), and strengthen
  the resolution check (see #3).

## 3. Stage-05 MCP check overstates what it verifies
- Docs claim it "verifies each MCP server starts"; the check only resolves the first token
  of the command. `cmd.exe /c npx -y <pkg>` always resolves, so missing packages — and even
  crashed servers — pass.
- Suggested fix: spawn each server and require an MCP `initialize` response; at minimum,
  resolve every token / confirm the package exists.

## 4. Python `.cmd` wrapper hardcodes the WindowsApps path
- `03-video-tooling/install-video-tooling.ps1` (line ~68) writes a wrapper that hardcodes
  `%LOCALAPPDATA%\Microsoft\WindowsApps\python.exe -m <tool>`.
- On machines where that `python` is the Microsoft Store stub (or Python is installed
  elsewhere), the wrapper breaks even though a working Python exists — exactly what
  happened on this machine.
- Suggested fix: resolve Python via `Get-Command python` / `py -3`, bake in the resolved
  absolute path, or emit a shim that prefers `uv tool run`.

## 5. Piper voice download: single hardcoded URL, no checksum/retry
- `install-video-tooling.ps1` (line ~85) hardcodes one Hugging Face URL for
  `en_US-lessac-medium.onnx`. No hash verification, no mirror/fallback — a flaky
  connection or a 404 leaves stage 03 without a voice.
- Suggested fix: verify SHA-256, add retry, and/or document a mirror. (Voice was
  prefetched manually here: 63,201,294 B.)

## 6. `docs/stages.md` references the wrong verifier name
- Line 21 documents stage 04's verifier as `verify-comfyui.ps1`; the shipped file is
  `verify-generation.ps1`. Doc-only fix.

## 7. Minor: JSONC comment loss when writing config
- The config-write path re-serializes with `ConvertTo-Json`, dropping any comments a user
  added to `opencode.jsonc`. Cosmetic; consider a comment-preserving writer or a separate
  overlay file.

## 8. Minor: duplicate report rows after an aborted-and-rerun stage 05
- Re-running `write-setup-report.ps1` after an aborted run appends duplicate rows for
  repeated steps. Consider deduplication or a fresh report per run.

## 9. Documentation gap: first-session npx downloads vs. the MCP connect timeout
- On a fresh machine, several shipped MCP servers are npx-based (rea, github, context7,
  sequential-thinking). Their first-ever `npx -y` download can exceed the client's MCP
  connect timeout (~50 s), and a timed-out download can leave a corrupted npx cache.
- Observed: on this machine only `whisper` connected at first boot; two npx caches needed
  deleting and re-downloading.
- Suggested fix: pre-warm each npx package during stage 01/03 (e.g. run the package once
  in a throwaway command), and/or document the expected first-run flakiness and its remedy.

## How to report
No upstream repository URL ships with the kit, and no GitHub CLI is present on this
machine, so these findings are stored locally instead of filed online. Attach or send this
file to whoever distributed the kit.
