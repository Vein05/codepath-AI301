# Phase III: Build

**Student:** Sugam Panthi  
**Issue:** [#1612](https://github.com/Shopify/tapioca/issues/1612) — `bin/tapioca dsl --only` should not purge existing directory  
**Status:** Phase III Complete

---

## Implementation Notes

### What Was Built

The fix adds a guard to skip stale RBI file removal when the `--only` flag is active. When `@only` contains any compilers, the purge computation in `generate_dsl_rbi_files` returns an empty set instead of computing `existing_rbi_files - generated_files`, which previously deleted RBI files from every compiler not listed in `--only`.

### Files Modified

| File | Change |
|------|--------|
| `lib/tapioca/commands/abstract_dsl.rb` | Added `@only.any?` guard around purge computation (+5 lines). When `--only` is set, `files_to_purge` is set to an empty `Set` and a log message is printed: "Skipping stale RBI removal because `--only` is set." The `--only --verify` path continues to work normally. |
| `spec/tapioca/cli/dsl_spec.rb` | Added a new test case (+29 lines) that generates RBI files for two compilers, re-runs with `--only SidekiqWorker`, and asserts that both RBI files still exist and no "Removing stale RBI files" message appears. |

### Key Commits

| Commit | Description |
|--------|-------------|
| [`be4220c`](https://github.com/Shopify/tapioca/pull/2642/commits/be4220c) | Skip stale RBI file removal when `--only` is set — core fix and initial test |
| [`a847648`](https://github.com/Shopify/tapioca/pull/2642/commits/a847648) | Use direct file writes instead of `tapioca dsl` for test setup (reviewer feedback) |

---

## Code Changes

**Branch:** [fix/skip-purge-with-only](https://github.com/Vein05/tapioca/tree/fix/skip-purge-with-only)  
**PR:** [#2642](https://github.com/Shopify/tapioca/pull/2642) (merged 2026-06-08)

---

## Challenges Faced

1. **Test setup performance.** The initial test ran a full `tapioca dsl` command to create the baseline RBI files before testing the `--only` behavior. The maintainer (@KaanOzkan) pointed out that this was slow and suggested using `@project.write!` to create the files directly. This was a quick fix but taught me to look at how existing tests in the project set up their fixtures rather than defaulting to end-to-end calls.

2. **Understanding the purge flow.** The purge logic spans two files (`abstract_dsl.rb` and `dsl_generate.rb`) and interacts with the `--verify` flag in addition to `--only`. I had to trace through both the generation and verification paths to make sure the guard did not break `dsl --only --verify`, which should still report missing files without deleting anything.

---

## Testing Strategy

### Tests Added

One new test case in `spec/tapioca/cli/dsl_spec.rb` that validates two properties:
- **Preservation:** RBI files from compilers not listed in `--only` are not deleted after a `--only` run.
- **Log message:** The skip-purge log message appears in stdout, confirming the guard was hit.

The test sets up two RBI files (simulating output from `SmartProperties` and `SidekiqWorker` compilers) using direct file writes, runs `tapioca dsl --only SidekiqWorker`, and asserts both files still exist.

### Validation Performed

| Check | Result |
|-------|--------|
| `bundle exec ruby -Itest -Ispec spec/tapioca/cli/dsl_spec.rb` | All 69 tests pass, 0 failures |
| `bin/typecheck` (Sorbet) | No errors |
| `bin/style` (RuboCop) | No offenses |
| Manual reproduction of original bug | Bug no longer reproduces; RBI files from other compilers are preserved |

---

## Scrum Participation

Participated in weekly standups during class (Wednesdays) and async scrums on Slack (Mondays and Fridays) in `#dts-su26-ai301-build-support`.
