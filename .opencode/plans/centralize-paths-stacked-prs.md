# Plan: Two stacked PRs for juliaup path centralization

Using jujutsu (jj v0.41.0). Both PRs target ghyatzo/juliaup fork, opened against JuliaLang/juliaup.

## Baseline (origin/main = `7327e75e`, "Unpin i686 cross image, bump Rust to 1.97")
- `GlobalPaths.juliaupselfbin` exists, gated `#[cfg(feature = "selfupdate")]`; no canonicalize.
- `get_paths()` calls `current_exe()` only under `selfupdate`.
- Scattered `current_exe()`: `bin/julialauncher.rs` (`get_juliaup_path`), `command_selfupdate.rs` (+ its `_post-update` spawn — #1513 fix), `operations.rs` (BundledJulia + `install_background_selfupdate`), `utils.rs::get_bin_dir`, `command_post_update.rs`, `bin/juliaup.rs:62` (Windows MSIX only — leave alone).
- Layout fact: binary+selfconfig in `~/.juliaup` (installer-chosen `install_location`); data in `~/.julia/juliaup` (from `JULIAUP_DEPOT_PATH`). They DIFFER BY DESIGN. `current_exe()` is the correct authority for binary location.
- `dunce` already a dep (Cargo.toml:33).

## Decisions locked
- D1 (feature gate): `juliaupselfexecfolder`/`juliaupselfexec` UNCONDITIONAL; `dunce::canonicalize`; soft fallback to non-canonicalized `current_exe()` if canonicalize fails.
- D2 (`get_bin_dir`): drop undocumented `JULIAUP_BIN_DIR` env var; KEEP `~/.local/bin` fallback for system-wide installs; source folder from `paths.juliaupselfexecfolder`.
- D3 (PR strategy): NEW PR off origin/main from fork; close #915 + #886 after merge.
- D4 (stack): PR1 = centralization (non-breaking). PR2 = stacked on PR1, the breaking `JULIAUP_DEPOT_PATH`-as-sole-authority proposal (draft/RFC-style).

---

## PR1 — "Centralize self-path discovery" (non-breaking)

### jj setup
1. `jj new main` — create empty change on top of origin/main.
2. `jj describe -m "Centralize where paths are discovered"` (or final message later).
3. `jj bookmark set centralize-paths-v2` on this change; delete stale `centralize-paths`/`centralize-paths-2` bookmarks: `jj bookmark delete centralize-paths centralize-paths-2`.

### Phase 1 — Core centralization
4. `src/global_paths.rs`: replace `juliaupselfbin` with `juliaupselfexecfolder` + `juliaupselfexec`; compute via `dunce::canonicalize(current_exe()?)`; unconditional; soft fallback if canonicalize fails. Remove `#[cfg(feature="selfupdate")]` on `my_own_path`/`juliaupselfbin`.
5. `src/bin/julialauncher.rs`: delete `get_juliaup_path()`; add `paths: &GlobalPaths` to `do_initial_setup`, `run_versiondb_update`, `run_selfupdate` (selfupdate variant); fix BOTH call sites to pass `&paths`; keep `#[cfg(not(feature="selfupdate"))]` stub consistent.
6. `src/command_selfupdate.rs`: download to `paths.juliaupselfexecfolder`; PRESERVE the `_post-update` spawn (use `paths.juliaupselfexec` for `new_juliaup`). Critical: rebase of #915 dropped this — don't repeat.
7. `src/operations.rs`: BundledJulia → `paths.juliaupselfexecfolder.join("BundledJulia")`; `install_background_selfupdate(interval, paths)` uses `paths.juliaupselfexec`.
8. `src/bin/juliainstaller.rs` + `src/command_config_modifypath.rs`: rename `juliaupselfbin` → `juliaupselfexecfolder`.

### Phase 2 — get_bin_dir cleanup (D2)
9. `src/utils.rs`: `get_bin_dir(paths: &GlobalPaths)`; remove `JULIAUP_BIN_DIR` branch; fallback uses `paths.juliaupselfexecfolder` then `~/.local/bin` (kept). Update all call sites to pass `paths`.
10. Grep `JULIAUP_BIN_DIR` in docs/tests; remove/adjust.

### Phase 3 — Loose ends
11. Comment at `global_paths.rs` parent-then-join explaining Windows `.exe` suffix + canonicalize-handles-symlinks (closes davidanthoff's open line-comment).

### Phase 4 — Verification
12. `cargo build` (default) AND `cargo build --features selfupdate,binjuliainstaller,binjulialauncher`.
13. `cargo clippy --workspace --features selfupdate,binjuliainstaller,binjulialauncher -- -D warnings` (matches clippy.yml).
14. `cargo test --features selfupdate,binjuliainstaller,binjulialauncher` AND `cargo test --features selfupdate,binjulialauncher --test command_selfupdate_test` (matches test.yml + selfupdate job).
15. Watch `tests/command_post_update_test.rs` + `tests/command_selfupdate_test.rs` (guard `_post-update` regression).

### Phase 5 — Prepare for PR (DO NOT push or open PR)
16. STOP after Phase 4 is green. Do NOT push the bookmark, do NOT run `gh pr create`, do NOT open any PR.
17. The user (ghyatzo) will write the PR message, push the branch, and open PR1 themselves. Have ready: a summary of changes, the list of fixed rebase regressions (`_post-update` spawn + `path`/`juliup` typo compile errors), D1/D2 decisions, and cross-links to #915, #886, and PR2 — so the user has the raw material for their message.

---

## PR2 — "Unify install layout under JULIAUP_DEPOT_PATH" (breaking, draft/RFC, stacked on PR1)

### jj setup
18. `jj new centralize-paths-v2` — empty change stacked on PR1.
19. `jj describe -m "RFC: unify install layout under JULIAUP_DEPOT_PATH (breaking)"`.
20. `jj bookmark set unify-depot-path`.

### Scope (proposal-level — sketch, refine with maintainer input)
21. Make `JULIAUP_DEPOT_PATH` (or default `~/.julia/juliaup`) the sole authority for BOTH data AND binary:
    - `juliaupselfhome = juliauphome` (= `$JULIAUP_DEPOT_PATH`/juliaup or `~/.julia/juliaup`).
    - `juliaupselfexecfolder = juliaupselfhome/bin`.
    - `juliaupselfexec = juliaupselfexecfolder/juliaup`.
    - Drop `current_exe()` for self-location discovery.
22. Migration: installer writes binary to `~/.julia/juliaup/bin` instead of `~/.juliaup/bin`; update uninstaller, PATH shell-script edits, `julia` symlink creation.
23. Backward-compat / migration path for existing `~/.juliaup` installs (move-on-first-run? symlink? installer detects and migrates?).
24. Drop `get_bin_dir`'s `~/.local/bin` fallback OR redefine system-wide install story (this is where it gets controversial — system-wide installs can't write to `~/.julia/juliaup/bin` for symlinks). This is the crux of the RFC.
25. Mark PR2 as DRAFT; body frames it as a proposal seeking design sign-off before committing to migration mechanics.

### Phase 4' — Verification (same matrix as PR1, on top of PR1's head)
26. Rebuild + retest stacked on `unify-depot-path`.

### Phase 5' — Prepare for PR2 (DO NOT push or open PR)
27. STOP after Phase 4' is green. Do NOT push, do NOT run `gh pr create`, do NOT open any PR.
28. The user (ghyatzo) will write the PR2 (RFC) message, push, and open PR2 as DRAFT themselves. Have ready: the breaking-change list, migration questions, and the system-wide-symlink crux so the user has raw material for their RFC message.

---

## After PR1 merges (user-driven; not automated)
29. When PR1 merges (user pushes & opens it themselves): `jj rebase -b unify-depot-path -d main`, user retargets PR2 base to main.
30. Close #915 and #886 referencing PR1 (user-driven).
