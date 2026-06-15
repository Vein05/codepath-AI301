# Phase II: Reproduce & Plan

**Student:** Sugam Panthi  
**Issue:** [#1612](https://github.com/Shopify/tapioca/issues/1612) — `bin/tapioca dsl --only` should not purge existing directory  
**Status:** Phase II Complete

---

## Reproduction Process

### Environment Setup

Cloned the Tapioca repo to `../shopify/tapioca/`. Ran `bundle install` with the Ruby version specified in `.ruby-version`. No setup issues; the project's dev container config and README instructions worked without modification.

### Steps to Reproduce

1. Clone the Tapioca repository and run `bundle install`.
2. Create a test project (or use an existing one) with at least two DSL compilers registered, such as `SmartProperties` and `SidekiqWorker`.
3. Run `bin/tapioca dsl` to generate RBI files for all compilers. Confirm that RBI files exist for both compilers in the output directory.
4. Run `bin/tapioca dsl --only SidekiqWorker`.
5. Check the output directory. RBI files from compilers other than `SidekiqWorker` (for example, `post.rbi` from `SmartProperties`) are deleted.
6. The terminal output shows "Removing stale RBI files" for the deleted files, confirming the purge ran on the full directory.

The bug reproduces consistently on every run. The `--only` flag restricts generation but does not restrict the purge scope.

### Reproduction Evidence

**Branch:** [fix/skip-purge-with-only](https://github.com/Vein05/tapioca/tree/fix/skip-purge-with-only) (merged via [PR #2642](https://github.com/Shopify/tapioca/pull/2642))

**Root cause:** `lib/tapioca/commands/abstract_dsl.rb:98` computes `existing_rbi_files - generated_files` unconditionally. When `@only` restricts the compiler set, `generated_files` is a strict subset of a full run, so every RBI file from excluded compilers lands in the purge set and gets deleted.

---

## Implementation Plan

**Understand:** The `--only` flag means "partial generation." Partial generation should not trigger full-directory cleanup. The purge set should be empty when `@only` is non-empty.

**Match:** The codebase already guards on `@only` in other places (for example, `abstract_dsl.rb:141` passes it to `constantize_compilers`). Checking `@only.any?` is consistent with how the flag is used elsewhere.

**Plan:**
1. In `lib/tapioca/commands/abstract_dsl.rb`, guard the purge computation inside `generate_dsl_rbi_files` with an `@only.any?` check. When `--only` is present, return an empty `Set` for `files_to_purge` and log: "Skipping stale RBI removal because `--only` is set."
2. Add a test in `spec/tapioca/cli/dsl_spec.rb` that generates RBI files for two compilers via a full `dsl` run, re-runs with `--only SidekiqWorker`, and asserts both RBI files still exist and no "Removing stale RBI files" message appears.
3. Update existing `--only` tests to include the new skip message in their expected stdout.

**Review checklist:**
- `bundle exec ruby -Itest -Ispec spec/tapioca/cli/dsl_spec.rb` passes (all 69 tests, 0 failures)
- `bin/typecheck` passes (Sorbet type checking)
- `bin/style` passes (linting)
- PR follows tapioca's contribution guidelines

**Evaluate:** The test verifies two properties: existing RBI files from other compilers are preserved, and the skip-purge log message appears in stdout.
