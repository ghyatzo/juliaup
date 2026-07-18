# RFC: Unify install layout under JULIAUP_DEPOT_PATH

**Status:** Draft — seeking design feedback before implementation.

## Problem

juliaup currently derives its install root (where the binary and self-config live) from `current_exe().parent().parent()` at runtime. This is fragile:

- It hard-codes a two-level nesting assumption (`install_root/bin/juliaup`).
- It breaks if the binary is placed at a different depth (e.g. `/usr/bin/juliaup` → install root = `/`).
- It re-derives the location on every invocation instead of remembering where it was installed.
- It is the outlier among version managers: rustup uses `CARGO_HOME` as a single env-var authority; uv uses an install receipt written at install time.

The two-directory split (`~/.juliaup` for binary+self-config, `~/.julia/juliaup` for data) was not a deliberate architectural decision — it was an accident of commit `5ada99e` (Jan 2022) placing `juliaupself.json` relative to the binary rather than relative to the data dir.

## Proposed solution

A hybrid approach combining env-var authority (rustup model) with an install receipt (uv model):

### 1. `JULIAUP_DEPOT_PATH` as data authority (existing, unchanged)

`juliauphome` continues to derive from `JULIAUP_DEPOT_PATH` (default `~/.julia/juliaup`). This already works and governs `juliaup.json`, versiondb, lockfile, Julia version installations, completions.

### 2. Install receipt as install-location authority (new)

The installer (`juliainstaller.rs`) writes a receipt file at install time, recording:

```json
{
  "install_location": "/home/user/.juliaup",
  "juliauphome": "/home/user/.julia/juliaup",
  "version": "1.21.0"
}
```

At runtime, `get_paths()` reads the receipt to find `install_location` and `juliaupselfconfig`, instead of deriving from `current_exe().parent().parent()`. The `--path`/`-p` flag continues to work as before — the installer simply records wherever it put things.

### 3. `JULIAUP_BIN_DIR` promoted to binary-location override (enhanced)

`JULIAUP_BIN_DIR` is promoted from symlink-placement-only to a binary-location override. When set, `get_paths()` uses it as the bin dir (`juliaupselfexecfolder`) instead of the receipt's `install_location/bin`.

### 4. Precedence

```
JULIAUP_BIN_DIR (env var)       ← wins for bin dir when set
JULIAUP_DEPOT_PATH (env var)    ← wins for data dir when set (existing)
        ↑ else
receipt values                   ← persistent, written at install time
        ↑ else (receipt missing)
current_exe derivation           ← last-resort fallback for existing installs
```

This keeps backward compatibility: existing installs without a receipt fall back to the current `current_exe()` behavior. New installs get the more robust receipt-based discovery.

### 5. Drop the `juliaup` subdir

`juliauphome` becomes `$JULIAUP_DEPOT_PATH` (not `$JULIAUP_DEPOT_PATH/juliaup`). The bin folder is `$JULIAUP_DEPOT_PATH/bin`. This simplifies the path structure. (Note: this changes `juliauphome` for everyone — data currently at `$JULIAUP_DEPOT_PATH/juliaup` would move to `$JULIAUP_DEPOT_PATH`. This is a breaking change requiring migration.)

## Why not env-var-only (rustup model)?

An env-var-only approach (`JULIAUP_DEPOT_PATH` as the sole authority for both binary and data) is simpler but:
- **The `--path` flag breaks**: the installer would need to set `JULIAUP_DEPOT_PATH` in the user's environment, requiring shell rc file edits or a wrapper script. The receipt approach lets `--path` keep working as-is.
- **No persistence across shells/GUI launches**: editors/IDEs that don't source `.bashrc` wouldn't inherit the env var. A receipt is discovered automatically on every invocation.
- **No multi-install detection**: env vars can't tell you "this binary doesn't match the canonical install." A receipt can (uv's `check_receipt_is_for_this_executable` pattern).

## Why not receipt-only (uv model)?

A receipt-only approach (no env var override) is what uv does, but:
- **juliaup supports multi-depot via `JULIAUP_DEPOT_PATH`** (used heavily in tests and by power users). uv has no equivalent. The env var must remain as an override that wins over the receipt.
- **uv refuses self-update with no receipt.** juliaup can't do that — existing installs don't have a receipt. The `current_exe` fallback is needed for migration.

## Breaking changes

1. **`juliauphome` path change**: `$JULIAUP_DEPOT_PATH/juliaup` → `$JULIAUP_DEPOT_PATH`. Data moves up one level. Requires migration.
2. **Existing installs need migration**: first run of the new binary detects the old layout and relocates files. The "Tolerate missing juliaupself.json" PR (prerequisite) ensures this doesn't hard-brick.
3. **Test layout changes**: `command_selfupdate_test.rs` needs rewriting to mirror the unified layout.

## Migration strategy (sketch)

On first run of the new binary (in `julialauncher.rs` or `juliaup` main, before `load_config_db_lockfree`):

1. Check if `juliaupself.json` exists at the old `current_exe().parent().parent()` location.
2. If yes and no receipt exists: write a receipt recording the old `install_location` and `juliauphome`.
3. If the new `juliauphome` path differs from the old one (due to dropping the `juliaup` subdir), relocate data files.
4. Refresh shell PATH blocks via `refresh_existing_shell_init_blocks`.

This is a one-shot migration that runs once and is idempotent.

## Open questions for maintainers

1. **Receipt location**: should the receipt live in `juliauphome` or in a platform config dir (like uv's `~/.config/uv/`)?
2. **`JULIAUP_BIN_DIR` rename**: should it be renamed to `JULIAUP_INSTALL_DIR` to reflect its promoted role? Or keep the name for backward compat?
3. **Fallback deprecation**: should the `current_exe` fallback be permanent (for backward compat) or time-limited (deprecation window)?
4. **Migration timing**: one-shot on first run of new binary, or installer-driven (re-run installer to migrate)?
5. **`juliaup` subdir removal**: is dropping the `juliaup` subdir from `juliauphome` worth the migration cost, or should we keep `$JULIAUP_DEPOT_PATH/juliaup` as `juliauphome` and put the bin at `$JULIAUP_DEPOT_PATH/juliaup/bin`?

## Comparison with other version managers

| Tool | Discovery authority | Self-update | Multi-depot |
|------|---------------------|-------------|-------------|
| rustup | `CARGO_HOME` env var (single root) | download `rustup-init`, run `--self-replace` | No |
| uv | Install receipt (written at install time) | `axoupdater` or re-run installer | No |
| mise/asdf | Package manager (no self-update) | n/a | n/a |
| juliaup (current) | `current_exe().parent().parent()` | download tarball, extract, `_post-update` | Yes (via `JULIAUP_DEPOT_PATH`) |
| juliaup (proposed) | Receipt + env var overrides + `current_exe` fallback | unchanged | Yes (preserved) |

## References

- PR1 (centralize paths): `centralize-paths-v2` branch
- Small Fix PR (tolerate missing self config): `tolerate-missing-selfconfig` branch
- rustup self_update source: https://github.com/rust-lang/rustup/blob/master/src/cli/self_update.rs
- uv self_update source: https://github.com/astral-sh/uv/blob/main/crates/uv/src/commands/self_update.rs
- axoupdater receipt module: https://docs.rs/axoupdater/latest/src/axoupdater/receipt.rs.html
