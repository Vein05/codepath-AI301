# Phase I: Select & Understand

**Student:** Sugam Panthi  
**Issue:** [#1612](https://github.com/Shopify/tapioca/issues/1612) — `bin/tapioca dsl --only` should not purge existing directory  
**Status:** Phase I Complete

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

## Resources Used

- https://github.com/Shopify/tapioca/issues/1612
- https://github.com/Shopify/tapioca/blob/main/lib/tapioca/commands/abstract_dsl.rb
- https://github.com/Shopify/tapioca/blob/main/lib/tapioca/commands/dsl_generate.rb
- https://github.com/Shopify/tapioca/blob/main/spec/tapioca/cli/dsl_spec.rb
