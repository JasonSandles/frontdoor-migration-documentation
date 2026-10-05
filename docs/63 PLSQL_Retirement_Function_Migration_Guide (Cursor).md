# PL/SQL Retirement AI-Assisted Function Migration Guide (Cursor)

*Required rules and step-by-step workflow for migrating one PL/SQL function or procedure to .NET with Cursor*

## Purpose

This guide explains how developers use Cursor to plan, implement, test, review, and submit PL/SQL retirement stories. AI accelerates analysis and coding, but it does not make architectural decisions. Every change must preserve the legacy routine's observable behavior, follow the repository rules, and pass human review before it is committed.

The guide has three parts:

* **Part 1** sets the rules that apply to every story.
* **Part 2** turns those rules into an ordered checklist for migrating **one function or procedure**. It is written for the layout where the PL/SQL source repository and the .NET target repository (`plsql-retirement`) are **sibling folders**, not one repository. Follow it top to bottom for every story.
* **The appendices** hold the reference tables that the steps point to.

| Document item | Value |
| :---- | :---- |
| Audience | Developers, reviewers, and architects working in the PL/SQL Retirement repository |
| Primary tool | Cursor using Plan, Agent, and Ask modes |
| Target platform | .NET 8 with the approved synchronous OracleDAL package |
| Required test framework | xUnit v3 in the corresponding domain test project |
| Consolidates | 61 PL/SQL Retirement AI-Assisted Development Guide and 62 Function Migration Step-by-Step Runbook (Cursor) |
| Version date | September 28, 2026 |

> **Main rule**
>
> Do not ask Cursor to implement a story immediately. First establish the source behavior and metadata, obtain a read-only plan, review that plan, then authorize a tightly bounded implementation.
>
> Do not skip a step to save time. The gate order (Plan before Agent, Agent before Ask, Ask before commit) is the control that keeps Cursor's output reviewable.

## Contents

**[Part 1: Ground rules](#part-1-ground-rules)**

1. [How AI is used in this project](#1-how-ai-is-used-in-this-project)
2. [Nonnegotiable architecture rules](#2-nonnegotiable-architecture-rules)
3. [The gated workflow](#3-the-gated-workflow)
4. [Workspace layout and one-time Cursor setup](#4-workspace-layout-and-one-time-cursor-setup)

**[Part 2: Step-by-step migration](#part-2-step-by-step-migration)**

* [Step 1: Identify the story and the exact source routine](#step-1-identify-the-story-and-the-exact-source-routine)
* [Step 2: Gather Oracle metadata](#step-2-gather-oracle-metadata)
* [Step 3: Resolve conflicts and get architect sign-off](#step-3-resolve-conflicts-and-get-architect-sign-off)
* [Step 4: Branch setup and preflight](#step-4-branch-setup-and-preflight)
* [Step 5: Cursor Plan mode (read-only)](#step-5-cursor-plan-mode-read-only)
* [Step 6: Review the plan (human gate)](#step-6-review-the-plan-human-gate)
* [Step 7: Cursor Agent mode (implementation)](#step-7-cursor-agent-mode-implementation)
* [Step 8: Run verification commands](#step-8-run-verification-commands)
* [Step 9: Cursor Ask mode final audit](#step-9-cursor-ask-mode-final-audit)
* [Step 10: Commit and GitLab MR](#step-10-commit-and-gitlab-mr)
* [Step 11: Definition of done](#step-11-definition-of-done)

**[Appendices](#appendices)**

* [Appendix A: Writing effective Cursor prompts](#appendix-a-writing-effective-cursor-prompts)
* [Appendix B: Data access and type mapping](#appendix-b-data-access-and-type-mapping)
* [Appendix C: Test layers](#appendix-c-test-layers)
* [Appendix D: Security and exception handling](#appendix-d-security-and-exception-handling)
* [Appendix E: Common problems and responses](#appendix-e-common-problems-and-responses)

---

## Part 1: Ground rules

### 1 How AI is used in this project

Cursor is used as an engineering assistant for repository inspection, source-to-.NET mapping, test design, code generation, and read-only review. The developer remains responsible for the story, the source interpretation, the approvals requested by Cursor, and the final code submitted to GitLab.

| Participant | Responsibility |
| :---- | :---- |
| Architect | Approves framework, architecture, source deviations, status meanings, data types, dependencies, and security decisions. |
| Developer | Collects the complete story and PL/SQL, verifies metadata, controls Cursor modes, reviews diffs, runs tests, and owns the merge request. |
| Cursor | Inspects code, proposes a bounded plan, implements only approved scope, runs verification, and reports findings. |
| Reviewer | Confirms source fidelity, architecture, tests, security, and scope before approving the merge request. |

AI output is evidence to review, not authority to accept. A successful build does not prove that Oracle binding, cardinality, null semantics, status mapping, or error messages are correct.

### 2 Nonnegotiable architecture rules

* All projects remain on .NET 8 unless architects approve a framework change.
* Continue using Swashbuckle. Do not replace it with `AddOpenApi` or `MapOpenApi` without approval.
* Use the approved custom ADO.NET-based OracleDAL NuGet package. Do not introduce Entity Framework or Dapper.
* PL/SQL business logic moves into .NET. Oracle SQL belongs in approved Infrastructure Oracle constant classes.
* Do not create replacement stored procedures, functions, views, or other database objects.
* Controller constructors continue creating `RepositoryWrapper` through the approved pattern. Do not inject `RepositoryWrapper` through DI.
* `RepositoryWrapper` owns OracleDAL initialization and connection access. Developers do not read or change connection strings.
* Connection-string changes require architect approval after consultation with the other architects.
* Use synchronous service and repository methods when the underlying OracleDAL operation is synchronous. Do not use fake async wrappers.
* Use xUnit v3 and place tests in the corresponding domain test project.
* Do not add packages, analyzers, authentication frameworks, or target-framework changes without architect approval.
* Preserve return codes, nullability, precision, output values, exact messages, and terminal control flow unless a documented architect decision says otherwise.

### 3 The gated workflow

Every story passes through six gates. The Part 2 steps are the concrete actions inside each gate.

| Gate | Cursor mode | Required outcome | Part 2 steps |
| :---- | :---- | :---- | :---- |
| 1 Source analysis | Ask or manual review | Conflicts and missing metadata are identified before coding. | Steps 1–3 |
| 2 Read-only plan | Plan | Cursor inspects the actual repository and lists exact files, contracts, SQL, tests, risks, and protected scope. | Steps 4–5 |
| 3 Plan approval | Human review | Developer confirms the plan matches the source and every architect decision. | Step 6 |
| 4 Implementation | Agent | Cursor implements only the approved plan using TDD and runs verification. | Steps 7–8 |
| 5 Final audit | Ask | Cursor performs a read-only review of the actual diff and returns PASS or FAIL by section. | Step 9 |
| 6 Commit and MR | Terminal and GitLab | Developer stages exact files, validates the staged diff, commits, pushes, and opens an MR to `dev`. | Steps 10–11 |

### 4 Workspace layout and one-time Cursor setup

Confirm your local layout matches this shape before starting. Substitute your actual folder names.

```
<repos-root>/
├── frontdoor/                          ← PL/SQL source repo (READ-ONLY reference)
│   └── src/
│       ├── in_scope/
│       │   ├── packages/               ← package specs (in-scope functions/procedures)
│       │   └── package_bodies/         ← package bodies (actual PL/SQL logic)
│       ├── all/                        ← full package set, used to trace dependents/callers
│       └── additional_code/
│           └── procedures/             ← standalone, non-packaged procedures/functions
└── plsql-retirement/                   ← .NET target repo (WRITE scope; cloned per doc 60)
    ├── ContractService/
    ├── CustomerService/
    ├── PLSQL-Retirement.ExceptionHandler/
    └── PLSQL-Retirement.slnx
```

The target repo is cloned as described in [60 Windows Workspace Cursor Developer Setup](60%20Windows%20Workspace%20Cursor%20Developer%20Setup.md).

**One-time Cursor setup for this pair of repos**

1. `File > Open Folder` → open `plsql-retirement` (the target repo). This is the folder where Agent mode is allowed to write.
2. `File > Add Folder to Workspace` → add the sibling PL/SQL source repo folder (for example, `frontdoor`). This lets Plan, Agent, and Ask modes inspect the actual legacy source instead of relying only on pasted text.
3. `File > Save Workspace As…` → save a `.code-workspace` file one level above both repos (not inside either repo) so the two-root setup reopens together next session.
4. Treat the source repo root as **read-only** for every story. Every Cursor prompt in this guide includes a line telling Cursor not to modify it. Do not remove that line.

---

## Part 2: Step-by-step migration

The commands in this part use `ContractService` as the example domain. Substitute the test project for your story's domain.

### Step 1: Identify the story and the exact source routine

1. Confirm the Jira story number, title, and complete acceptance criteria.
2. Identify the target: the domain service, and whether the capability is internal or exposed through an approved endpoint.
3. In the source repo, find the package spec and body for the routine:

   ```bash
   grep -rn "FUNCTION <name>\|PROCEDURE <name>" frontdoor/src/in_scope/packages frontdoor/src/in_scope/package_bodies
   ```

   Standalone (non-packaged) routines live in `frontdoor/src/additional_code/procedures` instead. See [02 Procedures.md](../Source/Frontdoor/02%20Procedures.md) in the source repo for how that folder is organized.
4. Copy the **complete** function or procedure body, including every exception handler and any constants or helper routines it references. A partial copy is the most common cause of a wrong plan in Step 5.
5. Search `frontdoor/src/all` for callers of this routine and for the routines it calls.

**Dependency sequencing**

For every routine this story calls, determine whether the dependency already exists in .NET or has its own Jira story.

* If it already exists in .NET, record the exact replacement and its result behavior for use in the prompts.
* If it has its own story and is not yet merged into `plsql-retirement`, pause this story. Do not stub a temporary replacement.

Build and merge dependencies in source order so temporary logic is not introduced and later rewritten.

### Step 2: Gather Oracle metadata

For every table and column this routine reads, writes, filters, or joins on, run this query:

```sql
SELECT table_name, column_name, data_type, data_length,
       data_precision, data_scale, nullable
  FROM all_tab_columns
 WHERE table_name = 'TABLE_NAME'
   AND column_name IN ('COLUMN_1', 'COLUMN_2')
 ORDER BY column_id;
```

* Use `ALL_TAB_COLUMNS` when the object belongs to another schema or is reached through a synonym.
* Record the data type, length, precision, scale, and nullability for every input, output, filter, join, and selected column.
* Do not infer a .NET type from the PL/SQL parameter name. Map from this query's output only, using the table in [Appendix B](#appendix-b-data-access-and-type-mapping).

### Step 3: Resolve conflicts and get architect sign-off

The source and acceptance criteria can disagree. Do not allow Cursor to choose silently. Compare the acceptance criteria against what the source code actually does, and check the story against the common conflicts below. For every mismatch, get an explicit architect decision and write it down in the story before opening Cursor. That decision is pasted directly into the Plan and Agent prompts.

| Common conflict | Required response |
| :---- | :---- |
| AC says greater than zero; source rejects only null and zero | Ask whether negative values must be preserved. Do not strengthen validation without approval. |
| AC says async; OracleDAL method is synchronous | Use a synchronous method with honest naming. Do not wrap the call in `Task.Run` or `Task.FromResult`. |
| NUMBER maps to `long` in AC but precision is NUMBER(20,0) | Use `decimal`/`decimal?` to avoid losing values above `long.MaxValue`. |
| Aggregate returns null | Distinguish an aggregate row containing DBNull from a missing result row. |
| `SELECT INTO` may return duplicates | Preserve `TOO_MANY_ROWS` behavior unless the source uses `ROWNUM` to select one. |
| Source message contains `SQLERRM` | Log details internally and use the approved safe exception response externally. |
| Source has unreachable exception branch | Document and test the distinguishable path without pretending Oracle normally reaches it. |
| Called routine has its own story | Implement and merge dependencies first rather than creating a temporary replacement. |

Also record decisions for any ambiguity, nondeterminism, unsafe messages, or type differences not listed above.

### Step 4: Branch setup and preflight

**Branch setup**

1. Merge prerequisite stories into `dev` and pull the updated `dev` branch.
2. Create the feature branch from the updated `dev`, using the Jira number and a descriptive name.

**Preflight**

Run from the `plsql-retirement` root before every story:

```bash
git branch --show-current
git status --short
dotnet run --project ContractService/Tests/ContractService.Tests.csproj
```

* The branch must be the intended feature branch, not `dev` or `main`.
* The working tree must be clean before implementation begins.
* All prerequisite stories must be present.
* The existing test baseline must pass. Record the exact passing count before touching any code.
* Stop and diagnose if the baseline fails or the branch is missing a dependency.

**Plan-mode readiness check**

A story is not ready for Cursor Plan mode until you have assembled all of the following.

| Required input | What to provide | Gathered in |
| :---- | :---- | :---- |
| Jira story | Story number, title, and complete acceptance criteria. | Step 1 |
| Legacy source | The complete function or procedure, including exception handlers and referenced constants. | Step 1 |
| Target | Domain service and whether the capability is internal or exposed through an approved endpoint. | Step 1 |
| Dependencies | Source for called routines or the exact existing .NET replacement and its result behavior. | Step 1 |
| Oracle metadata | Data type, length, precision, scale, and nullability for every referenced input, output, filter, join, and selected column. | Step 2 |
| Decisions | Architect-approved resolutions for conflicts, ambiguity, nondeterminism, unsafe messages, and type differences. | Step 3 |
| Branch state | Prerequisite stories merged, `dev` pulled, feature branch created from updated `dev`, and clean working tree. | Step 4 |

### Step 5: Cursor Plan mode (read-only)

Switch Cursor to **Plan** mode. Plan mode is read-only. The prompt must instruct Cursor to inspect the actual source, repository contracts, readers, test fakes, OracleDAL overloads, and current rules. The plan must name every file to create or modify and explicitly list the protected files that will remain unchanged.

Use this prompt. Replace bracketed values with story-specific facts, and write the story-specific details using the guidance in [Appendix A](#appendix-a-writing-effective-cursor-prompts).

```text
The legacy PL/SQL source for this story is in the sibling folder
frontdoor/src/in_scope/... (path from step 1). Read it for reference only.
Do not modify, create, delete, move, format, stage, commit, or push
anything in the frontdoor folder.

Create a read-only implementation plan for [JIRA NUMBER] [ROUTINE].

Do not modify, create, delete, move, format, stage, commit, or push files
in plsql-retirement either.

Inspect the actual repository, current branch, git status, rules, existing
patterns, installed OracleDAL API, and tests before proposing changes.

Preflight:
- Confirm the feature branch and clean working tree.
- Confirm prerequisite stories are present.
- Run the current domain test project and record the exact baseline.
- Stop on an incorrect branch, dirty tree, missing dependency, or failed baseline.

Plan the exact service and repository contracts, SQL, named binds,
OracleDbType values, reader cardinality, status/value/message mapping,
exception behavior, tests, fake-interface updates, protected files, verification,
and deferred integration checks.

Show the plan only. Do not implement anything.

--- Complete legacy source (step 1) ---
[PASTE FULL FUNCTION/PROCEDURE, INCLUDING EXCEPTION HANDLERS]

--- Oracle metadata (step 2) ---
[PASTE ALL_TAB_COLUMNS RESULTS]

--- Architect decisions (step 3) ---
[PASTE EACH CONFLICT AND ITS RESOLUTION]
```

Cursor should inspect the actual `plsql-retirement` files and the sibling source folder. For the existing patterns it should follow, see [20 Domain/Controller Architecture](20%20Domain_Controller_Architecture_Developer_Guide.md), [22 RepositoryWrapper/Repository](../22%20RepositoryWrapper_Repository_Developer_Guide_v2.md), and [23 OracleDAL Consumption](23%20OracleDAL_Developer_Consumption_Guide_Final.md).

If Cursor edits any file while in Plan mode, stop and follow [Appendix E](#appendix-e-common-problems-and-responses).

### Step 6: Review the plan (human gate)

Before authorizing implementation, confirm:

* The plan's description of source behavior matches the pasted function line for line: branches, statuses, exact messages, and cardinality.
* Every architect decision from Step 3 appears in the plan.
* The named SQL, bindings, and `OracleDbType` values are explicit, not vague. "Parameterize the query" is not acceptable (see [Appendix A](#appendix-a-writing-effective-cursor-prompts)).
* A protected-files list is present and correct.

If anything is missing or wrong, refine the prompt and re-run Plan mode. Do not move to Agent mode on a plan you have not fully checked.

### Step 7: Cursor Agent mode (implementation)

Switch Cursor to **Agent** mode only after the plan is approved. The implementation prompt restates the approved contracts, messages, SQL, bindings, tests, scope restrictions, and verification commands. Cursor must stop before expanding scope.

```text
The legacy PL/SQL source is in the sibling folder frontdoor/ — read-only.
Do not modify anything there.

Implement the approved [JIRA NUMBER] plan exactly as written.

Repeat preflight and stop if the branch, clean state, dependencies, or baseline
do not match the approved plan.

Follow TDD. Implement only the approved contracts, SQL, bindings, reader,
service behavior, and tests. Preserve exact source messages and terminal flow.
Do not change protected files or expand scope.

Run git diff --check, the complete solution build, the domain test build,
dotnet run for the xUnit v3 project, and dotnet test. Report the known test
toolchain conflict separately without changing configuration.

Show every created and modified file, exact test totals, warnings, git status,
protected-file confirmation, and deferred integration checks.

Do not commit, push, or create a merge request.

--- Approved plan (step 6) ---
[RESTATE THE APPROVED CONTRACTS, EXACT MESSAGES, SQL, BINDINGS, TESTS,
AND PROTECTED FILES]

--- Architect decisions (step 3) ---
[PASTE EACH CONFLICT AND ITS RESOLUTION]
```

The tests Cursor writes should cover the layers described in [Appendix C](#appendix-c-test-layers).

**Approving Cursor's requests**

* Approve file-write and command prompts only when they match the approved plan. A prompt to create a required directory or write an approved file is expected.
* Decline and escalate to an architect for any package install, framework or SDK upgrade, or connection-string edit.
* Decline any commit or push. Those are developer actions in Step 10.

### Step 8: Run verification commands

```bash
git diff --check
dotnet build PLSQL-Retirement.slnx
dotnet build ContractService/Tests/ContractService.Tests.csproj
dotnet run --project ContractService/Tests/ContractService.Tests.csproj
dotnet test ContractService/Tests/ContractService.Tests.csproj
```

**Known toolchain issue.** In the current environment, `dotnet test` can fail because the .NET 10 SDK invokes an unsupported VSTest target for `Microsoft.Testing.Platform`. Do not change the SDK, project, packages, or framework inside a story to hide that toolchain issue. Use `dotnet run` on the xUnit v3 test project as the verified test runner and report the `dotnet test` failure separately.

**Test baseline rule.** Record the passing count before implementation (Step 4) and after implementation. A higher final count does not replace coverage review. The Ask-mode audit in Step 9 must inventory the new tests and confirm that consolidated tests contain all required assertions.

### Step 9: Cursor Ask mode final audit

Switch to **Ask** mode. The final audit must review the actual files and git diff, not repeat the implementation summary. It must inventory tests, verify exact messages and bindings, report git status, and end with a single verdict.

```text
The legacy PL/SQL source is in the sibling folder frontdoor/ — read-only.
Do not modify anything there.

Perform a final read-only review of the uncommitted [JIRA NUMBER]
implementation.

Do not modify, format, stage, commit, or push files. Review the actual source
and git diff, not only the implementation summary.

Mark each section PASS or FAIL and cite exact files and lines for failures:
1. Method contracts
2. Input validation and dependency order
3. SQL fidelity
4. OracleDAL parameter binding
5. Reader cardinality and precision
6. Service status, value, nullability, and exact messages
7. Exception translation and security
8. Regression behavior
9. Test inventory and coverage
10. Architecture and protected scope
11. Worktree, build, warnings, and exact test results
12. Deferred Oracle integration

End with exactly one verdict:
- Ready to commit
- Corrections required before commit

Do not implement corrections.
```

If any section fails, return to Agent mode only for the specific correction called out. Do not expand scope beyond that fix. Then re-run Step 8 and this audit.

### Step 10: Commit and GitLab MR

These are developer-controlled actions, performed only after the Ask-mode review returns **Ready to commit**. Never ask Cursor to commit, push, or create the merge request.

1. Stage the exact approved file list. Do not use a broad `git add` when unrelated changes exist.
2. Run `git diff --cached --check` and inspect `git diff --cached --stat`.
3. Commit with a Jira-scoped message, such as `feat(DAB-250): implement contract existence check`.
4. Push the feature branch and create a GitLab merge request targeting `dev`.
5. Complete the repository merge-request template and include build, test, warning, and deferred-integration results.

```bash
git status --short
git add -- <approved file 1> <approved file 2> ...
git diff --cached --check
git diff --cached --stat
git commit -m "feat(DAB-###): concise capability description"
git push --set-upstream origin HEAD
```

### Step 11: Definition of done

A story is ready to commit only when every item below is true.

* The implementation matches the approved plan and source decisions.
* All intended files, and only intended files, are changed.
* Protected architecture and configuration files are unchanged.
* Exact status codes, values, nullability, and messages are covered by tests.
* Oracle SQL is parameterized and stored in the approved SQL constants class.
* Reader behavior covers zero, one, multiple, DBNull, missing-column, casing, and precision cases where applicable.
* The complete solution builds and all warnings are reported.
* The domain test project builds and the xUnit v3 runner passes with exact counts reported.
* `git diff --check` is clean.
* The final Ask-mode review marks every section PASS and ends with **Ready to commit**.
* Deferred Oracle integration checks are listed honestly.
* The developer has personally inspected the staged diff before committing.

---

## Appendices

### Appendix A: Writing effective Cursor prompts

Every story prompt should make the following boundaries explicit.

| Prompt element | Required content |
| :---- | :---- |
| Contracts | Exact service and repository signatures, including nullability and sync or async behavior. |
| Behavior | Every input branch, dependency branch, status, value, message, and terminal return. |
| SQL | Exact tables, joins, filters, constants, aggregates, cardinality, and named parameters. |
| Oracle bindings | Parameter names, values, `OracleDbType` values, and call order. |
| Mapping | Zero-row, one-row, multiple-row, DBNull, missing-column, precision, and casing behavior. |
| Tests | Specific service, reader, SQL, exception, regression, and terminal-flow assertions. |
| Protected scope | Files and architecture that must not change. |
| Verification | Build, test, diff, and status commands plus exact reporting requirements. |

**Use concrete language**

* Write "preserve the exact message" and include the literal, including any trailing space.
* Write "negative IDs reach Oracle unchanged" when the source rejects only null and zero.
* Write "do not add ORDER BY" when the source intentionally has no ordering.
* Write "multiple rows throw" or "ROWNUM selects one arbitrary row" according to the actual source.
* Write "OracleDbType.Decimal" or "OracleDbType.Varchar2" rather than saying only "parameterize the query."
* Write "do not commit or push" in every implementation prompt.

### Appendix B: Data access and type mapping

| Oracle type or behavior | Project mapping |
| :---- | :---- |
| NUMBER(20,0) identifier | `decimal` or `decimal?`; bind with `OracleDbType.Decimal` |
| DATE | `DateTime` or `DateTime?`; bind with `OracleDbType.Date` |
| VARCHAR2 | `string` or `string?` based on metadata; bind with `OracleDbType.Varchar2` |
| CHAR | `string` or `string?`; preserve source padding unless approved otherwise |
| DBNull | Map according to column nullability and source branch; do not silently substitute defaults |
| `SELECT INTO` zero rows | Map to the exact source-defined result |
| `SELECT INTO` multiple rows | Throw deterministically unless `ROWNUM` or an approved selection rule prevents it |
| COUNT aggregate | Expect one aggregate row; distinguish count zero from an anomalous missing aggregate row |

**Repository method rules**

* Call `ClearSQLParameters` before binding.
* Bind every value by name with an explicit Oracle type.
* Execute the approved SQL constant and pass the `DataTable` to a focused reader.
* Do not catch general exceptions, access connection strings, or dispose an OracleDAL instance the repository does not own.

### Appendix C: Test layers

| Layer | What it proves | What it does not prove |
| :---- | :---- | :---- |
| Service tests | Validation, dependency order, statuses, values, exact messages, terminal behavior, and exception translation. | Live Oracle behavior or real parameter binding. |
| Reader tests | `DataTable` cardinality, DBNull, missing columns, aliases, casing, and precision. | Actual provider metadata and live result shapes. |
| SQL tests | Required SQL text, named binds, and absence of forbidden clauses. | That OracleDAL binds or executes correctly. |
| Integration tests | Real binding, filters, Oracle types, result metadata, duplicates, and database failures. | They are deferred unless the story explicitly authorizes a database environment. |

When integration tests are deferred, list the exact deferred checks in the Agent report and the MR. Do not claim that source-text tests prove runtime behavior.

### Appendix D: Security and exception handling

* Do not return `exception.Message`, `SQLERRM`, SQL, connection details, stack traces, internal paths, or Oracle-specific details to clients.
* Preserve the complete exception for internal logging through the registered handlers.
* Use the established `OthersErrorException` path for unexpected service or repository failures.
* Do not rewrap an existing `OthersErrorException`.
* Do not add redundant general try/catch blocks to repositories.
* Keep handler registration order and `Program.cs` unchanged unless the story explicitly authorizes a change.
* Do not add authentication or authorization behavior as incidental work.

### Appendix E: Common problems and responses

| Problem | Response |
| :---- | :---- |
| Cursor is in Plan but edited files | Stop, inspect `git status`, discard nothing automatically, and determine exactly what changed before continuing. |
| Agent asks to write an approved file | Approve only if the path and file are in the approved plan. |
| Agent asks to install a package or upgrade .NET | Decline and stop. Escalate to an architect. |
| Baseline tests fail | Do not implement. Diagnose the branch, dependencies, environment, and existing failures first. |
| NuGet restore returns 401 | Verify the configured GitLab token and source. Never commit credentials. |
| "LF will be replaced by CRLF" warning | This is normally a line-ending notice on Windows, not a failed add. Verify `.gitattributes` and the staged diff. |
| Swagger shows only one service | Use the approved compound debug configuration and separate service ports. |
| Cursor proposes fake async | Reject it. Use synchronous naming and calls when OracleDAL is synchronous. |
| Story depends on an unimplemented routine | Pause the story and implement or merge the dependency first. |
| Live Oracle behavior is not tested | Document the exact deferred integration checks. Do not claim source-text tests prove runtime behavior. |
