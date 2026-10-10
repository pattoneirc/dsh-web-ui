# Agent Note: One materialization of the market clean-checkout fixture

Status: implemented

## Problem

`pnpm test:scripts` spent most of its wall clock in `scripts/market-build-clean.test.mjs` alone. Each of the eight cases called `fixture()`, which materializes the whole committed market tree into a fresh temporary directory, and the copy — not the gate under test — was the cost. Measured on the development machine with an idle machine (`loadavg` 2.5-6.7, 10 cores):

- `.market-inputs/{pet,community,presets}` — 35,421 files, 6.3-7.9s
- `market/dist` — 4,678 files, 519MB, 1.3s
- `.market-inputs/skins` — 789 files, 0.26s
- `node scripts/market-build --check` itself — 2.0-2.2s

So the intuitively expensive part (the 4,678-file dist) was about 15% of the cost; the fetched content cache dominated. Eight cases therefore spent 65-75s of a gate that finished in 67s, and the gate ran on every pull request.

## Decision

`scripts/market-build-clean.test.mjs` materializes the expensive tree once per test file, in a `before` hook, and every case borrows that one tree:

- A case declares **every path it may create, modify or delete** (`borrowFixture(t, scratch)`). The helper snapshots those paths (bytes plus timestamp), and a `t.after` hook — which node:test runs even when the case fails — restores them.
- After restoring, the helper verifies the whole tree against the stamp recorded at build time (path set, kind, size, and a per-path timestamp). A case that writes somewhere it did not declare fails there, by name, instead of silently corrupting a later case.
- The one case that runs a full build rewrites every file under `market/dist`, so it cannot borrow the shared tree directly: `borrowWritableDist` renames the shared dist aside, hands the case a copy of it, and renames the original back afterwards. The original dist inodes are never written, and the 36k-file input cache stays shared, so the case costs one 1.3s dist copy instead of a 9s full materialization.
- `market-build --check` materializes its comparison tree under `.market-check-tmp` and removes it on every path that reaches the comparison, but a rejection raised after that emit (`verifyTryonManifest` on an undeclared tryon file) leaves it behind. It used to disappear with the per-case copy, so the shared tree clears it after each case.
- A final case asserts the materialization counters: the shared tree was built exactly once, and the only other traversal is the single dist copy. The guard is a count, not a time budget, because the repository has no timing calibration and a budget would be machine-sensitive.

Isolation is preserved by construction, not by convention: the restore runs in an after hook on failure, and the stamp check turns any undeclared drift into a failing assertion naming the path.

## Measured

Same working tree and window, pre-change versus post-change:

- `node --test scripts/market-build-clean.test.mjs`: 99.27s pre-change (8 cases, one fixture each) versus 24.36s post-change (8 cases sharing one fixture plus the single dist copy) — 4.1x on the file.
- `pnpm test:scripts`: 67.36s median (three samples, 67.04/67.36/76.81) pre-change versus 27.5s single-sample smoke post-change; 383 of 383 tests pass in both.

The 32s gap the file-level comparison once predicted is smaller in practice because the check runs and the fixture cleanup remain.

## Alternatives considered

- **Reflink or clone copies** (`cp -c` on APFS, `cp --reflink=auto` on Linux): rejected. Reflinks are filesystem-dependent (ext4 without reflink support falls back to a byte copy), and a darwin-only shortcut would not run in the Linux CI lane.
- **A per-case hardlink farm over the shared tree**: rejected. A write through a hardlink rewrites the shared inode, so a case that modifies a file in place would corrupt the base for every later case — the failure mode is silent, which is exactly what the isolation requirement forbids.
- **Symlinking the immutable parts of the tree into a per-case overlay**: rejected. `market-build` derives its own root from `path.resolve(__dirname, '..')`, and Node resolves a symlinked script to its real path, so a symlinked `scripts/market-build` would build into the shared tree instead of the case's overlay.
- **Copying only the files a case mutates and letting it read the rest from the repository**: rejected. The gate resolves every path from its own root, so the fixture has to be a self-contained checkout; partial trees would test a layout the gate never sees.
- **Trimming the fixture's input cache to the files the catalog reads**: rejected for now. It would shrink the fixture further, but it changes what the clean-checkout gate proves, and the sharing above already removes the repetition that the measurement blamed.
- **Running the check cases in parallel**: rejected as the fix for this cost. The measurement puts materialization ahead of the check (6.3-7.9s versus 2.0-2.2s per case), so parallel checks would leave the dominant term in place and add a shared-tree data race on top.

## Consequences

The eight cases now share one tree, so a new case must declare its scratch paths; a case that forgets fails the stamp check by name rather than passing quietly. The stamp check costs one stat walk per case (about 0.2s), and the setup adds one full-tree walk. The full-build case still copies the 519MB dist, which is the largest remaining single cost inside the file. Nothing about `scripts/market-build` changed: the gate's semantics, the manifest fields, and the pin guard behave as before.

## Testing

`node --test scripts/market-build-clean.test.mjs` passes 9 of 9 (eight cases plus the materialization guard). Two negative controls were run against a symlinked copy of the checkout under `/tmp`, so no repository state was touched:

- The pre-change file (from `HEAD`) with the new materialization guard appended fails that guard with `actual: 8`, while its eight original cases still pass — the guard can fail.
- The post-change file with one undeclared write injected into a case passes that case's own assertions and then fails the isolation guard with `scripts/scratch.txt` named — the isolation guard can fail.

`pnpm test:scripts` (383 tests), `pnpm test:standards`, `pnpm typecheck`, `pnpm emoji:check` and `pnpm docs:check` all pass with the change.
