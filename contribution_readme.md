# Contribution 1: `bin/tapioca dsl --only` should not purge existing directory

**Contribution Number:** 1  
**Student:** Sugam Panthi  
**Issue:** https://github.com/Shopify/tapioca/issues/1612  
**Status:** Phase II In Progress

---

## Why I Chose This Issue

Tapioca is a widely used Ruby tool in the Shopify ecosystem, and its codebase is well-structured with strong typing via Sorbet. This issue is a good first contribution because it does not require understanding Ruby type theory or writing a new DSL compiler. It is a command-behavior bug with a clear root cause and a well-scoped fix.

I want to learn how production Ruby CLIs handle flag interactions and how mature open-source projects structure their test suites. The maintainers are responsive; @KaanOzkan confirmed the issue is still open and ready for a PR on 2026-06-05.

---

## Understanding the Issue

### Problem Description

When running `bin/tapioca dsl --only SomeCompiler`, tapioca restricts DSL RBI generation to the specified compiler(s) but still runs full stale-file cleanup afterward. Since only a subset of compilers ran, the cleanup treats every RBI file from other compilers as "stale" and deletes them. This produces a large, misleading diff that has nothing to do with the compiler being tested.

### Expected Behavior

`--only` is a debugging/focused-run flag. When it is present, tapioca should generate RBI files for the specified compiler(s) only and leave all other existing RBI files untouched. A log message should inform the user that purging was skipped because of the `--only` flag.

### Current Behavior

Tapioca computes `existing_rbi_files - generated_files` and deletes the result. When `--only` restricts generation to one compiler, almost every existing RBI file ends up in the purge set. Running `bin/tapioca dsl --only SidekiqWorker` in a project with 50 DSL compilers would delete RBI files from the other 49 compilers.

### Affected Components

- `lib/tapioca/commands/abstract_dsl.rb:76-100`: the `generate_dsl_rbi_files` method computes the purge set at line 98
- `lib/tapioca/commands/dsl_generate.rb:32-35`: `DslGenerate#execute` calls `generate_dsl_rbi_files` and then `purge_stale_dsl_rbi_files`
- `lib/tapioca/commands/abstract_dsl.rb:251`: the `purge_stale_dsl_rbi_files` method that performs the deletion

---

## Reproduction Process

### Environment Setup

Cloned the repo to `../shopify/tapioca/`. Ruby version and bundle install tracked separately.

### Steps to Reproduce

1. Set up a project with multiple DSL compilers (e.g., SmartProperties and SidekiqWorker)
2. Run `bin/tapioca dsl` to generate all RBI files
3. Run `bin/tapioca dsl --only SidekiqWorker`
4. Observe that RBI files from other compilers (e.g., `post.rbi` from SmartProperties) are deleted

### Reproduction Evidence

- **Commit showing reproduction:** TBD (will link after forking and reproducing)
- **Screenshots/logs:** TBD
- **My findings:** The purge logic at `abstract_dsl.rb:98` does not account for whether `@only` restricts the compiler set. The subtraction `existing_rbi_files - generated_files` is correct for full runs but wrong for partial runs.

---

## Solution Approach

### Analysis

The root cause is in `generate_dsl_rbi_files` at `abstract_dsl.rb:98`. The method computes `files_to_purge = existing_rbi_files - generated_files` unconditionally. When `@only` is non-empty, `generated_files` is a strict subset of what a full run would produce, so the purge set incorrectly includes files from compilers that were intentionally excluded from this run.

### Proposed Solution

Skip the purge computation entirely when `@only` is present. Return an empty set so `DslGenerate#execute` has nothing to delete. Add a log message informing the user that purging was skipped.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** `--only` means "partial generation." Partial generation should not trigger full-directory cleanup. The purge set should be empty when `@only` is non-empty.

**Match:** The codebase already checks `@only` in other places (e.g., `abstract_dsl.rb:141` passes it to `constantize_compilers`). The pattern of guarding behavior on `@only.empty?` is consistent with how the flag is used elsewhere.

**Plan:**
1. Modify `lib/tapioca/commands/abstract_dsl.rb`: in `generate_dsl_rbi_files`, return an empty `Set` for `files_to_purge` when `@only` is non-empty. Add a log message: "Skipping purging because of `--only` flag."
2. Add a test in `spec/tapioca/cli/dsl_spec.rb` near the existing `--only` tests that verifies: (a) existing RBI files from other compilers are not deleted, and (b) the skip-purge log message appears.

**Implement:** TBD (will link branch and commits)

**Review:**
- Does the fix pass `bundle exec ruby -Itest -Ispec spec/tapioca/cli/dsl_spec.rb`?
- Does `bin/typecheck` pass (Sorbet type checking)?
- Does `bin/style` pass (linting)?
- Does `bin/test` pass (full test suite)?
- Does the PR follow tapioca's contribution guidelines?

**Evaluate:** The test creates a pre-existing RBI file (`post.rbi`), runs `dsl --only SidekiqWorker`, and asserts that `post.rbi` still exists and no "Removing stale RBI files" message appears.

---

## Testing Strategy

### Unit Tests

- [ ] Existing RBI files from other compilers are preserved when `--only` is used
- [ ] The generated RBI file for the specified compiler is still created
- [ ] A log message about skipping purge appears in stdout
- [ ] Normal purge behavior is unchanged when `--only` is not used (no regression)

### Integration Tests

- [ ] Full `bin/tapioca dsl` still purges stale files correctly
- [ ] `bin/tapioca dsl --only X` with a valid compiler generates only that compiler's output and keeps everything else

### Manual Testing

TBD after implementation.

---

## Implementation Notes

### Code Changes

- **Files modified:**
  - `lib/tapioca/commands/abstract_dsl.rb` (skip purge when `@only` is present)
  - `spec/tapioca/cli/dsl_spec.rb` (add test for `--only` preserving existing files)
- **Key commits:** TBD
- **Approach decisions:** Chose to guard the purge computation in `generate_dsl_rbi_files` rather than in `DslGenerate#execute` because the method is responsible for returning the purge set. Keeping the guard close to the computation is cleaner.

---

## Pull Request

**PR Link:** TBD

**PR Description:**

> Do not purge stale DSL RBIs when using --only
>
> Fixes #1612.
>
> When `tapioca dsl --only` is used, only a subset of DSL compilers runs. Previously, Tapioca still compared the generated files against the full existing DSL RBI directory, causing unrelated RBI files to be removed as stale.
>
> This skips stale RBI purging when `--only` is present, preserving existing generated files during focused compiler runs.

**Maintainer Feedback:**
- TBD

**Status:** Not yet submitted

---

## Learnings & Reflections

### Technical Skills Gained

TBD

### Challenges Overcome

TBD

### What I'd Do Differently Next Time

TBD

---

## Resources Used

- https://github.com/Shopify/tapioca/issues/1612
- https://github.com/Shopify/tapioca/blob/main/lib/tapioca/commands/abstract_dsl.rb
- https://github.com/Shopify/tapioca/blob/main/lib/tapioca/commands/dsl_generate.rb
- https://github.com/Shopify/tapioca/blob/main/spec/tapioca/cli/dsl_spec.rb
