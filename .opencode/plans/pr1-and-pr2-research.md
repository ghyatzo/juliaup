## PR1 message sketch

**Title:** Centralize where paths are discovered

**Body:**

Alternative to #886, supersedes #915.

This PR carries forward @davidanthoff's centralization idea from #915 and @ghyatzo's `current_exe().canonicalize()` from #886, rebased onto current main and fixed.

**What changed**

Two new unconditional fields in `GlobalPaths` — `juliaupselfexecfolder` and `juliaupselfexec` — computed once via `dunce::canonicalize` in `get_paths()` and reused everywhere, replacing scattered `current_exe()` + parent() duplications.

- `juliaupselfexecfolder` = the bin directory (where the running binary lives). Used by: self-update download target, BundledJulia path, PATH shell-script edits, `julia`→`julialauncher` symlink (install + post-update restore).
- `juliaupselfexec` = the path to the `juliaup` binary itself. Used by: launcher spawning juliaup for init/setup/self-update/auto-install, cron/schtasks entries for background self-updates, the `_post-update` hook after self-update.

**Regressions from #915 fixed**

- The `_post-update` hook is **preserved** (#915's original rebase dropped it, which would have re-opened #1513 — the `julia` symlink being wiped during self-update on Unix).
- Compile errors in `command_selfupdate.rs` (typo `juliupselfexecfolder`, missing `paths` variable) and `julialauncher.rs` (missing `paths` argument at `run_selfupdate` call sites) corrected.

**Design decisions**

- **D1 (feature gate):** `juliaupselfexecfolder`/`juliaupselfexec` are unconditional, not gated behind `selfupdate`. Soft fallback if `dunce::canonicalize` fails.
- **D2 (symlink dir):** `get_bin_dir()` keeps the undocumented `JULIAUP_BIN_DIR` env var for users who need to place channel symlinks (`julia-<channel>`) separately from the binary location. The fallback now sources from `paths.juliaupselfexecfolder` instead of a duplicate `current_exe()` call, and the `~/.local/bin` fallback for system-wide installs is preserved.
- **`juliaupselfhome` removed:** The field was redundant (always equalled `juliaupselfexecfolder.parent()`) and only consumed by the uninstaller. The uninstaller now derives the install root locally.
- **Comment corrected:** The `parent().join("juliaup")` pattern is documented as "the launcher and juliaup are siblings in the bin dir, so when the launcher calls `get_paths()`, `current_exe()` returns the launcher's path, not juliaup's" — not the incorrect ".exe suffix on Windows" hypothesis from the original #915 thread.

**Follow-up (separate PR / RFC)**

The `JULIAUP_DEPOT_PATH`-as-sole-authority direction (unifying `~/.juliaup` and `~/.julia/juliaup`) is deliberately deferred. See [PR2 URL — TBD] for the RFC.

---

## PR2 research — "Unify install layout under JULIAUP_DEPOT_PATH" (draft/RFC)

### Three possible approaches

**Approach A: Env-var authority (rustup model)**
- Make `JULIAUP_DEPOT_PATH` (or its default `~/.julia/juliaup`) the sole authority for *both* binary location and data.
- `juliaupselfhome = juliauphome`, `juliaupselfexecfolder = juliaupselfhome/bin`, `juliaupselfexec = juliaupselfexecfolder/juliaup`.
- Drop `current_exe()` entirely for self-location discovery.
- **Breaking:** Installer layout changes from `~/.juliaup/bin/juliaup` to `~/.julia/juliaup/bin/juliaup`. Existing `~/.juliaup` installs need migration (move binaries, update PATH, update shell scripts).
- **System-wide symlink problem:** If binary is at `/opt/juliaup/bin/juliaup` but `JULIAUP_DEPOT_PATH` is `~/.julia/juliaup`, the binary is NOT under the depot. This case breaks. Solution: require either a custom `JULIAUP_DEPOT_PATH` matching the install location, or a separate mechanism for the binary location.
- **Pros:** Matches rustup's proven model. No more `current_exe()` fragility.

**Approach B: Install receipt (uv model)**
- Keep the current two-directory layout (`~/.juliaup` for binary+selfconfig, `~/.julia/juliaup` for data).
- At install time, write a receipt file (`juliaup-install.json`) recording both paths.
- At runtime, read the receipt instead of deriving from `current_exe().parent().parent()`.
- `current_exe()` is used only to *validate* the receipt (uv's `check_receipt_is_for_this_executable`), not to *discover* the install root.
- **Pros:** Non-breaking, keeps the layout split, more robust than parent-of-parent.
- **Cons:** Adds a new file to maintain. If the receipt is deleted or corrupted, fall back to `current_exe()`.

**Approach C: Env var + receipt hybrid**
- Introduce `JULIAUP_SELF_HOME` or `JULIAUP_INSTALL_DIR` env var as the authority.
- Default to the receipt (if found), then to `current_exe().parent().parent()` as fallback.
- The installer always writes the env var or the receipt.
- **Pros:** Backward compatible, authoritative, no layout migration.
- **Cons:** Adds env var + receipt complexity.

### System-wide install ("the crux")

All approaches share the same problem for system-wide installs: channel symlinks (`julia-<channel>`) need to go to a user-writable, PATH-located directory. Neither `juliaupselfexecfolder` (system dir, not writable by user) nor `juliaupselfhome`/`juliauphome` (data dir, possibly not on PATH) work. The current `~/.local/bin` fallback in `get_bin_dir()` handles this, but only if the binary is outside `$HOME`.

A proper solution would either:
- Keep `JULIAUP_BIN_DIR` (which we did in PR1).
- Or introduce `JULIAUP_SYMLINK_DIR` with a clearer name and document it.
- Or make `get_bin_dir` check if `juliaupselfexecfolder` is on PATH + writable, and fall through `~/.local/bin` → user-configurable env var → error.

### Recommendation for PR2

Approach B (install receipt) as a draft RFC. It's the least breaking path that eliminates the `current_exe().parent().parent()` fragility. The receipt is conceptually simple: written at install time, read at runtime, validated against `current_exe()`. If the user moves their binary, the validation fails and we recover by re-deriving from `current_exe()`.

The env-var authority (Approach A) is more elegant but requires a layout migration that touches installer, uninstaller, PATH scripts, and every existing install — high risk for low return (since the `current_exe().parent().parent()` derivation is already correct for the default install layout).

The system-wide symlink question can be addressed separately (or never — `JULIAUP_BIN_DIR` already exists for power users).
