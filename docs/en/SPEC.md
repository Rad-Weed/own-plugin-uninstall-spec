# Uninstall Specification for My Own Plugins

### Rules My Own AI Must Follow When Building Plugins: Installation, Uninstallation, and External Side Effects (v1.0 · First version)

> **Scope**: **rules I (Rad-Weed) impose on my own AI assistant when it builds DSH plugins**. The AI must follow them while creating, packaging, installing, and uninstalling plugins — especially those with install scripts, desktop shell patches, external files, or system-level changes.
> **Origin**: a real uninstallation incident on 2026-10-19 (`dsh-whale-widget` and `dsh-restart-button`): the official removal path took 4 min 10 s, produced a misleading `not-removable`, left stale `node_modules` junctions behind, and the shell patch could only be reverted after DSH fully exited.
> **Status**: this is **a checklist I set for my own AI's plugin-development behavior**. It is **not** an official DSH standard. Others are free to reference, quote, or adapt it. Existing official and marketplace conventions are summarized in §2; this document only fills the gap they do not cover: uninstallation, rollback, external side effects, and crash recovery.
> **Version**: v1.0 (first version) · 2026-10-19

---

## 0. Disclaimer (must read)

- This specification is **for my own AI**: I require it to follow these rules while building, packaging, installing, and uninstalling DSH plugins for me.
- It targets **my own AI's plugin-development behavior**; it is not an official DSH standard, marketplace admission condition, or review criterion.
- Other authors may freely reference, quote, or adapt it without permission; adoption is entirely voluntary.
- The incident review and rules reflect my personal experience only and target no specific plugin, author, or project.

## TL;DR (10 rules I require my own AI to follow)

1. **Installation and uninstallation must be symmetric**: every install action needs a re-entrant, idempotent reverse action; running an uninstall script twice must have no side effects.
2. **External side effects must be declared**: if anything outside the plugin bundle is changed — `app.asar`, system configuration, desktop shell, services, or user directories — the README / `package.json` must state what was changed and how to revert it.
3. **Shell-patch plugins must ship an orchestrator**: detect whether DSH is running → wait for it to fully exit → modify/restore files → verify SHA256 → restart. **Never write `app.asar` while DSH is running.**
4. **Waiting must run in an independent process**: DSH terminates its child-process tree on exit. Use WMI / Scheduled Tasks to escape; never rely on a normal child process of DSH.
5. **After uninstall, self-check and clean the DSH quartet**: `package.json` (dependencies/bundles), `pnpm-lock.yaml`, `node_modules/.modules.yaml`, `cordis.yml` — plus any leftover `node_modules/<pkg>` junction.
6. **Keep user data by default**: uninstallation must not touch user data; purging requires an explicit flag (`--purge`) and the data path must be documented.
7. **Give a time budget and a log**: DSH's remove path can be silent for minutes due to pnpm registry retries (measured: 4 min 10 s). Scripts must log, set timeouts, and never look hung.
8. **Prefer in-host designs**: if Cordis host services/slots can do it, do not touch the installation directory. Use `ctx.effect` to own every cleanup, no daemons, no raw child processes.
9. **Follow the official dependency and compatibility contract**: host interface packages only in `peerDependencies`; declare `engines.dsh`; always bump the version; entry points/exports must really exist.
10. **Follow the marketplace security and disclosure contract**: declare `cloud/network/api_keys/permissions` truthfully; no raw `child_process` (P202); do not shadow `single` slots (P401).

---

## 1. Why my AI needs this specification (real incident summary)

| Symptom | Evidence | What I can prevent |
|---|---|---|
| Uninstall looks "hung" | `pnpm remove` triggered registry retries (10 s → 1 min backoff); both removals reported `Done in 4m 10.6s` with no output | Add logging and a time-budget note to uninstall scripts; avoid network-dependent uninstall paths |
| Misleading error | In the window right after pnpm rewrote `package.json` (bundle already gone from the list), a retried removal returned `not-removable` with no diagnostic | Document "wait until the previous operation finishes before retrying"; disable first, then remove |
| Stale directory junctions | After a successful uninstall, `node_modules/dsh-whale-widget` and `node_modules/dsh-restart-button` junctions remained | Self-check and clean them at the end of the uninstall script (§3.4) |
| Shell patch cannot be reverted online | `app.asar` is exclusively locked by the running DSH process | Build the "exit → apply → restart" orchestration into the install/uninstall scripts (§3.3, §4) |
| Half-applied state after interruption | The user closed the uninstall window twice → pnpm interrupted → three overlapping remove operations; config and dependencies out of sync | Idempotent + re-entrant + retryable scripts with explicit recovery guidance |
| Manual recovery required | A standalone background script finally did it: wait for exit → official restore → SHA verification → delete backup → auto-restart; completed in 4 seconds | Ship this orchestration inside the plugin's own install/uninstall scripts |

> Conclusion: **the plugin manager only owns the "Cordis rows + pnpm dependencies" layer.** Anything beyond that layer (shell patches, external files, system settings) is outside its uninstall contract and must be symmetrically cleaned up by the AI that builds the plugin.

---

## 2. Existing conventions in the ecosystem

| Document | Layer | Key content | Relation to this spec |
|---|---|---|---|
| Official "Packaging and Installing Plugins" [publish.zh.md](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/user/develop/basic/publish.zh.md) | Packaging / install contract | bundle vs. profile manifests, `dsh.bundle.patch`, `engines.dsh`, peer dependencies, layer order, `dsh plugin remove` removes both dependency and layer; `prepare` + `allowBuilds` for Git installs | **Official authority for installation**; does not cover external side-effect cleanup |
| Official "Plugins and Lifecycle" [framework/index.zh.md](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/user/develop/framework/index.zh.md) | Runtime lifecycle | Fibers, `ctx.effect`, reverse-order disposers, HMR/reload | Official basis for runtime resource cleanup (§3.9) |
| DSH Plugin Marketplace [STANDARD.md](https://github.com/bradeGithub/DSH-Plugins-Marketplace/blob/main/STANDARD.md) | Recognition / install / disclosure / verification | 10-step type detection, minimal cordis `package.json`, anti-patterns (root-level install scripts, self-registered patches, host packages in `dependencies`…), DISCLOSURE v0.2, verification field contract, self-test checklist | **Marketplace "recognition layer" authority**; its uninstall quick-reference is only one line ("script installs cannot be rolled back") |
| [dsh-plugin-verify](https://github.com/qing3a/dsh-plugin-verify) | Runtime verification / review | 7/7 waterfall + tools/result runtime verdicts, P202 (raw child process) / P401 (single slot) safety rules, [plugin-standards.md](https://github.com/qing3a/dsh-plugin-verify/blob/main/docs/plugin-standards.md), [review-checklist.md](https://github.com/qing3a/dsh-plugin-verify/blob/main/docs/review-checklist.md); its submission conditions already require an README with **install/uninstall** instructions | **De-facto verification standard**; §5 of this document can complement it |
| [dsh-design-conventions.md](https://github.com/qing3a/dsh-plugin-verify/blob/main/docs/source/dsh-design-conventions.md) | Official design conventions digest | no daemons, host exit SIGKILLs the whole child tree, `ctx.subprocess`, `ctx.storageDomain`, slot rules, error-code contract, i18n/doc rules | Basis for §3.9 |
| [dsh-agent-teams dsh-plugin-development Skill](https://github.com/NanmiCoder/dsh-agent-teams/blob/main/.dsh/skills/dsh-plugin-development/SKILL.md) | Development checklist | runtime-surface decisions, bundle contract, effect ownership, client build, HMR/restart boundaries, validation matrix, definition of done | Implementation-level best practices |
| [dshbase](https://dshbase.com/) / [awesome-dsh-plugin](https://github.com/awesome-dsh-plugin/awesome-dsh-plugin) | Directory / grading | admission thresholds, install-tested badges, security disclaimers | Distribution-side requirements |

**Findings**:

- The ecosystem **already has** rules for "how to be recognized/installed/updated" (marketplace STANDARD);
- It **already has** rules for "can it be trusted" (dsh-plugin-verify, disclosure contract);
- It **already has** rules for "how not to write a bad plugin" (official docs + community skills);
- **But no document defines an author contract for uninstall, rollback, external side effects, or interruption recovery.** The marketplace docs even state that script-type installs "cannot be rolled back automatically — the author must provide reverse-operation instructions", with no executable requirements.

§3 of this document turns that gap into an executable minimal contract **for my own AI's plugin development**.

---

## 3. Rules I require my own AI to follow (M = mandatory · R = recommended · A = anti-pattern)

### 3.1 ⛔ M1 — Installation and uninstallation must be symmetric, idempotent, re-entrant

- Every install action must have a matching uninstall action; running the script twice must have no side effects (second run prints "already in target state" and exits 0).
- The uninstall script must converge safely from **any intermediate state**: dependency removed or not, shell patch reverted or not, backup deleted or not.
- Failures must be **retryable**, and must preserve the scene (backup, logs, required files) instead of leaving state half-modified.
- Provide `-DryRun` / `--dry-run`: print what would be done without changing anything.

### 3.2 ⛔ M2 — External side effects must be declared with reverse operations

Document in the README (and optionally a custom `package.json` field):

| Item | Description |
|---|---|
| Files touched | Semantic, not machine-specific, e.g. `DSH install dir/resources/app.asar` |
| What was changed | Shell patch / tray / file associations / scheduled tasks / env vars / ports |
| Online revertible? | If not, state "requires DSH to fully exit before reverting" |
| Backup and verification | Backup directory, SHA256, original size |
| Reverse action | Uninstall command / script path / manual steps |
| User data location | Data directory, kept by default, how to purge |

**Draft manifest field** (for future marketplace/host consumption; writing it into the README works today):

```jsonc
{
  "dsh": {
    "bundle": { "patch": "./cordis.patch.yml" },
    "install": {
      "effects": ["app.asar"],            // app.asar | profile | user-data | system | service
      "requiresFullExit": true,           // must DSH fully exit to uninstall?
      "uninstall": "./scripts/uninstall.ps1",
      "backup": { "dir": "resources/.dsh-myplugin-backup", "sha256": true }
    },
    "data": { "path": "~/.dsh/my-plugin", "purgeFlag": "--purge" }
  }
}
```

### 3.3 ⛔ M3 — Dedicated contract for shell patches / system-level changes

- **Never write `app.asar` while DSH is running**: the file is exclusively locked and truncating it can corrupt the installation.
- **Always back up first**: original header + `meta.json` (`originalSize`, `originalSha256`, `patchedSha256`, `headerBytes`). Keep the backup in the install directory until restoration is verified.
- **Double SHA verification**: current file == patched SHA → write → result == original SHA. Refuse and keep the backup if either check fails.
- **Scripts must orchestrate themselves**: detect DSH processes → wait for full exit (zero processes **and** the file can be opened exclusively, confirmed repeatedly) → modify → verify → restart DSH.
- **The waiting process must be independent** (WMI `Win32_Process.Create` or Scheduled Tasks): DSH synchronously terminates its child tree on exit, so a normal child process would be killed with it.
- **On failure**: popup + log + keep the backup; never silently continue or restart into a broken state.
- **Prefer solutions that never touch the install directory**: host services, capabilities supported by `desktopNext`, pure Cordis plugins, or a small external process.

### 3.4 ⛔ M4 — After uninstall, assert and clean the "quartet + junction"

Before the uninstall script exits, assert and clean:

```powershell
$profile = "$env:USERPROFILE\.dsh\profiles\desktop"   # replace with the actual profile
# 1. package.json: our package absent from dsh.profile.bundles and from dependencies
# 2. pnpm-lock.yaml / node_modules\.modules.yaml: no reference
# 3. cordis.yml: no rows of this package
# 4. node_modules\<pkg> junction left behind → use rmdir (removes only the link)
cmd /c rmdir "$profile\node_modules\<pkg>"
# 5. print the final inventory as evidence that uninstall succeeded
```

Recommend including this verification snippet in the README so users can check that "it is really gone".

### 3.5 ⛔ M5 — Keep user data by default; purge only explicitly

- Store data in `~/.dsh/<plugin-scope>/...` or via `ctx.storageDomain`; never scatter it across `cwd`, the Desktop, or the install directory.
- Uninstall **keeps** user data by default; deleting it requires `--purge` or a separate command, documented in the README.
- After uninstall, print the data location (what was kept and how to remove it).

### 3.6 ⛔ M6 — Waiting / timeouts / logs / exit codes for uninstall scripts

- Parameters: `-TimeoutSeconds` (default ≥ 900 s; ≥ 3600 s for shell-patch plugins), `-Force`, `-SkipPlugin`, `-DryRun`, `-PurgeData`.
- Log everything with timestamps (pnpm output, checksums, error stacks); document the log path in the README.
- While waiting for exit, show a clear instruction ("quit DSH from the tray completely; do not click Restart") and explain that "multiple minutes of silence during removal is normal".
- Exit codes: `0` success; `1` failure; `2` bad arguments; `3` wait timeout; `4` verification failure. Document them.

### 3.7 ⭐ R1 — Network and time expectations

- Keep install/uninstall paths **network-independent** where possible (local `link:`, tarballs, committed build artifacts); when network is required, offer an offline mode or a timeout.
- State in the README that DSH's built-in remove path goes through pnpm and may take minutes due to registry retries; **closing the terminal interrupts it and leaves a half-applied state**.
- If removal seems stuck, check `.plugin-manager/logs/operation-*/pnpm.log` before clicking again.

### 3.8 ⭐ R2 — Host dependencies, versions, package metadata

- `@deepseek-ai/*` host interface packages belong in `peerDependencies` only (build-time copies go to `devDependencies`) to avoid shadowing the host (marketplace §6.6 real case).
- Declare `engines.dsh` (e.g. `">=0.2.0 <0.3.0"`); follow semver and bump the version on every release.
- `main` / `exports` / `files` must point to files that really exist; prefer artifact-type packages (commit `lib/`), and for source-type packages provide a self-contained `prepare`.
- Fill in `repository`; README install commands must be verified on a fresh profile.

### 3.9 ⭐ R3 — Runtime resource cleanup (host/client)

- Every long-lived resource (route, listener, timer, watcher, DOM, React root, socket, temporary service) must be owned by the current fiber via `ctx.effect(() => disposer)` / `ctx.on()`, so dispose/HMR cleans them in reverse order.
- **No daemons**: DSH is a foreground process; long-running/daemon work belongs to an external host. Child processes must go through `ctx.subprocess` (host exit reaps the whole tree).
- Have HMR/dispose safety tests: registry, DOM, styles, and controllers are all cleaned up after unload/reload.
- Do not shadow `single` slots (P401); do not use raw `child_process.spawn` (P202).

### 3.10 ⭐ R4 — Security and disclosure

- Declare per marketplace STANDARD §9: `cloud`, `network`, `offline_mode`, `api_keys` (env/storage), `permissions`, `jurisdiction`, `retention`.
- Minimize permissions; read credentials from env vars only, never persist them or log them; local HTTP endpoints stay on loopback with a method allowlist.
- Script-based installs must ask the user for explicit confirmation; Git installs should be commit-pinned.

### 3.11 ⛔ A — Anti-patterns (do not do)

| Anti-pattern | Consequence |
|---|---|
| Root-level `install.ps1` plus a `dsh` capability declaration | Marketplace may mis-classify; I should put scripts under `scripts/` (STANDARD §6.1) |
| Registering rows into the profile `cordis.patch.yml` from the plugin itself | Double-loading crash (issue #39) |
| Putting host interface packages into `dependencies` | An old copy shadows the host; tool calls break |
| Modifying `app.asar` while DSH runs | File corruption / startup failure |
| Uninstall script without wait-for-exit logic | Half-applied state after interruption; users think "uninstall is broken" |
| Successful uninstall but stale `node_modules` junction | Weird states on the next install/update |
| Deleting user data by default | Irreversible data loss |
| Silent failures / English-only stack traces | Users cannot recover on their own |

---

## 4. Reference implementation: idempotent uninstall script skeleton (PowerShell)

> Key points: independent process waiting for exit, double SHA verification, preserve the scene on failure, auto-restart. For a full implementation see `dsh-restart-button`'s `uninstall-restart-key.ps1` plus the `restore-on-exit.ps1` used for this incident.

```powershell
param(
  [string]$InstallDir = 'D:\DSH',
  [int]$TimeoutSeconds = 900,
  [switch]$SkipPlugin, [switch]$Force, [switch]$DryRun, [switch]$PurgeData
)
$ErrorActionPreference = 'Stop'
$profileDir = Join-Path $env:USERPROFILE '.dsh\profiles\desktop'
$backupDir  = Join-Path $InstallDir 'resources\.dsh-myplugin-backup'
$log = Join-Path $env:USERPROFILE ".dsh\myplugin-uninstall.log"

function Log($m) { Add-Content $log "$(Get-Date -Format s)  $m" -Encoding UTF8 }
function Popup($m,$icon) { (New-Object -ComObject WScript.Shell).Popup($m,0,'DSH plugin uninstall',$icon) | Out-Null }

# 1) Remove the plugin package (official channel; -SkipPlugin skips it)
#    dsh plugin --profile desktop remove dsh-my-plugin   (may take minutes; capture output)
# 2) Wait for DSH to fully exit: zero processes AND app.asar openable exclusively, 3 consecutive checks
#    If this script was started from inside DSH, it MUST run as an independent process (WMI / Scheduled Task)
# 3) Revert external side effects (example: app.asar)
#    node <official restore tool> restore --asar <app.asar>
# 4) Double SHA verification: current == patchedSha256 -> restore -> result == originalSha256
# 5) Clean the leftover junction: cmd /c rmdir "$profileDir\node_modules\dsh-my-plugin"
# 6) Assert the manager quartet (see §3.4)
# 7) User data: kept by default; delete only with $PurgeData
# 8) Restart DSH: Start-Process "$InstallDir\DeepSeek Harness.exe"
# 9) On failure: Popup + keep backup + non-zero exit code
```

**How to start an independent waiting process from inside DSH** (so it survives DSH exit):

```powershell
Invoke-CimMethod -ClassName Win32_Process -MethodName Create -Arguments @{
  CommandLine = 'powershell.exe -NoProfile -ExecutionPolicy Bypass -WindowStyle Hidden -File "C:\path\restore-on-exit.ps1"'
}
```

(Note: if the script contains non-ASCII text, Windows PowerShell 5.1 requires **UTF-8 BOM**, otherwise parsing fails. This was a real pitfall in this incident.)

---

## 5. My AI's pre-delivery self-test checklist

**Install**
- [ ] Fresh profile + fresh `DSH_HOME`: one-shot install per README succeeds
- [ ] `dsh --profile <p> --dump-config` shows this plugin's layer
- [ ] Repeated installs have no side effects (idempotent)

**Uninstall**
- [ ] Official `dsh plugin --profile <p> remove <pkg>` removes dependency and layer
- [ ] Uninstall script is re-entrant: second run only prints "already clean"
- [ ] Interrupted uninstall converges when re-run (simulate Ctrl+C)
- [ ] After uninstall, all §3.4 quartet + junction assertions pass
- [ ] `-DryRun` changes nothing
- [ ] Failure keeps the backup and logs and is retryable

**External side effects (if any)**
- [ ] Refuses to touch `app.asar` while DSH runs, with a clear message
- [ ] Wait-for-exit logic runs in an independent process; timeout/verification failures produce a popup and a log
- [ ] After restore, SHA256 == original value
- [ ] DSH starts normally after restore

**Network / compatibility / security**
- [ ] Uninstall still succeeds offline (or fails with a clear reason)
- [ ] `engines.dsh`, peerDependencies, and version numbers are correct
- [ ] Disclosure fields complete; no raw `child_process`; no `single` slot shadowing
- [ ] Uninstall leaves user data intact by default; `--purge` behavior documented

**Verification tools**
- [ ] `npx dsh-plugin-verify <plugin path> --repo <DSH checkout>` passes (7/7 waterfall + tools/result)
- [ ] If published to a marketplace: repo has the `dsh-plugin` topic, LICENSE, README (install/uninstall/minimal example), and `repository` field

---

## 6. Appendix: Ecosystem observations (reference only)

> The following are my personal observations of the ecosystem, shared for reference only.

1. **Plugin manager**
   - Support an `uninstall` hook / external side-effect declaration (§3.2's `dsh.install.effects`), applied or rolled back by the host at restart;
   - Make "removing / removed / partially applied" explicit states; attach diagnostics and advice to `not-removable` (the current retry window misreports it);
   - Let remove use a resolved-lock offline fast path, avoiding registry retries for unrelated packages;
   - Reconcile `node_modules` after pnpm finishes; reconcile orphaned junctions at startup;
   - Stream pnpm output in the GUI with a "may take several minutes" hint.
2. **Plugin marketplace STANDARD**
   - Add an "uninstall / rollback" chapter (this document's §3 can serve as reference);
   - Show `effects` / `requiresFullExit` / data locations on cards, so users know before installing that "this plugin patches app.asar and uninstalling needs DSH to exit".
3. **Official docs publish.zh.md**
   - Add an "uninstallation and external side effects" section: clarify the boundary of `dsh plugin remove` (it only removes dependencies and layers; it does not handle shell patches or external files).

---

## 7. References

**Official**
- "Packaging and Installing Plugins" <https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/user/develop/basic/publish.zh.md>
- "Plugins and Lifecycle" <https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/user/develop/framework/index.zh.md>
- Official repository <https://github.com/deepseek-ai/deepseek-harness>

**Marketplace / verification / community conventions**
- DSH-Plugins-Marketplace `STANDARD.md`: <https://github.com/bradeGithub/DSH-Plugins-Marketplace/blob/main/STANDARD.md>
- dsh-plugin-verify (runtime verification + review checklist): <https://github.com/qing3a/dsh-plugin-verify>
- DSH design conventions digest: <https://github.com/qing3a/dsh-plugin-verify/blob/main/docs/source/dsh-design-conventions.md>
- dsh-plugin-development Skill: <https://github.com/NanmiCoder/dsh-agent-teams/blob/main/.dsh/skills/dsh-plugin-development/SKILL.md>
- dshbase (community directory/tutorials): <https://dshbase.com/>
- awesome-dsh-plugin: <https://github.com/awesome-dsh-plugin/awesome-dsh-plugin>

---

## Appendix A: Incident timeline (reproducible evidence)

| Time | Event |
|---|---|
| 00:02:33 | `dsh-whale-widget` removal operation starts (pnpm remove) |
| 00:02:35 → 00:06:44 | pnpm registry request for `@dsh-ops/bash-win32-x64` fails, retries with 10 s / 1 min backoff; `Done in 4m 10.6s` |
| 00:06:5x | A second removal call returns a misleading `not-removable` (bundle already gone from the list, pnpm already rewrote `package.json`) |
| after 00:06:44 | `node_modules/dsh-whale-widget` junction remains (pointing to the 0.3.18 package on the Desktop) |
| 00:15:00 → 00:19:09 | `dsh-restart-button` removal, again `Done in 4m 10.6s`; `node_modules/dsh-restart-button` junction remains |
| 00:14 / 00:15 | The author's own uninstall script got stuck twice at the "remove plugin package" step (user closed the window → interrupted) |
| 00:25:03 | Independent background script starts (WMI escape), waits for DSH to fully exit |
| 00:27:26 → 00:27:30 | Exit detected → official restore → double SHA verification passes → backup deleted → DSH auto-restarted |
| 00:29 | Recheck: `app.asar` has the original SHA256, no patch markers, profile clean, no plugin entries in the Loader |
---

## AI and My Own Research

This document was generated by AI together with my own research on some of the referenced documents; it may not be comprehensive.
