# Function Migration Step-by-Step Runbook (Cursor)

**Purpose**

[61 PLSQL_Retirement_AI_Assisted_Development_Guide.md](61%20PLSQL_Retirement_AI_Assisted_Development_Guide.md) defines the required rules and gates for using Cursor on a PL/SQL retirement story. This runbook turns those rules into an ordered checklist for migrating **one function or procedure**, written for the layout where the PL/SQL source repository and the `.NET` target repository (`plsql-retirement`) are **sibling folders**, not one repository. Follow this document top to bottom for every story; consult guide 61 whenever a step references one of its sections, appendices, or tables.

Do not skip a step to save time. The gate order (Plan before Agent, Agent before Ask, Ask before commit) is the control that keeps Cursor's output reviewable.

## 0 Workspace layout

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

**One-time Cursor setup for this pair of repos**

1. `File > Open Folder` → open `plsql-retirement` (the target repo). This is the folder where Agent mode is allowed to write.
2. `File > Add Folder to Workspace` → add the sibling PL/SQL source repo folder (e.g. `frontdoor`). This lets Plan/Agent/Ask modes inspect the actual legacy source instead of relying only on pasted text.
3. `File > Save Workspace As…` → save a `.code-workspace` file one level above both repos (not inside either repo) so the two-root setup reopens together next session.
4. Treat the source repo root as **read-only** for the rest of this runbook. Every Cursor prompt in this document includes a line telling Cursor not to modify it — do not remove that line.

## 1 Identify the story and the exact source function

1. Confirm the Jira story number, title, and complete acceptance criteria.
2. In the source repo, find the package spec and body for the routine:
   ```
   grep -rn "FUNCTION <name>\|PROCEDURE <name>" frontdoor/src/in_scope/packages frontdoor/src/in_scope/package_bodies
   ```
   Standalone (non-packaged) routines live in `frontdoor/src/additional_code/procedures` instead — see [02 Procedures.md](../Source/Frontdoor/02%20Procedures.md) in the source repo for how that folder is organized.
3. Copy the **complete** function or procedure body, including every exception handler and any constants or helper routines it references. A partial copy is the most common cause of a wrong Plan in step 5.
4. Search `frontdoor/src/all` for callers of this routine and for routines it calls. If a called routine has its own Jira story and is not yet merged into `plsql-retirement`, pause this story — do not stub a temporary replacement (guide 61 §6, "Dependency sequencing").

## 2 Gather Oracle metadata

For every table/column this routine reads, writes, filters, or joins on, run the metadata query from guide 61 §3 against the schema (or the owning schema via `ALL_TAB_COLUMNS` if reached through a synonym):

```sql
SELECT table_name, column_name, data_type, data_length,
       data_precision, data_scale, nullable
  FROM all_tab_columns
 WHERE table_name = 'TABLE_NAME'
   AND column_name IN ('COLUMN_1', 'COLUMN_2')
 ORDER BY column_id;
```

Record the result for every input, output, filter, join, and selected column. Do not infer a .NET type from the PL/SQL parameter name — map from this query's output only (see guide 61 §7 for the type-mapping table).

## 3 Resolve conflicts and get architect sign-off

Compare the acceptance criteria against what the source code actually does. Check it against the "Common conflict" table in guide 61 §6 (negative IDs, async vs. sync, NUMBER precision, aggregate nulls, `SELECT INTO` duplicates, `SQLERRM` exposure, unreachable exception branches). For every mismatch, get an explicit architect decision and write it down in the story before opening Cursor. Do not let Cursor pick silently.

## 4 Preflight in the target repo

Run from the `plsql-retirement` root:

```
git branch --show-current
git status --short
dotnet run --project ContractService/Tests/ContractService.Tests.csproj
```

* Branch must be the intended feature branch, created from an up-to-date `dev`, not `dev`/`main` itself.
* Working tree must be clean.
* Prerequisite stories must already be merged in.
* Record the exact passing test count before touching any code. Stop and diagnose if the baseline fails.

## 5 Cursor Plan mode (read-only)

Switch Cursor to **Plan** mode. Use guide 61 Appendix A as the prompt skeleton, but prepend the sibling-repo boundary line so Cursor knows where the legacy source lives and that it must not touch it:

```
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

Cursor should inspect the actual `plsql-retirement` files (existing patterns — see the domain/controller, RepositoryWrapper, and OracleDAL developer guides, docs [20](20%20Domain_Controller_Architecture_Developer_Guide.md), [22](22%20RepositoryWrapper_Repository_Developer_Guide_v2.md), [23](23%20OracleDAL_Developer_Consumption_Guide_Final.md)) and the sibling source folder, then return a plan naming every file to create or modify plus the protected files that stay unchanged.

## 6 Review the plan (human gate)

Before authorizing implementation, confirm:

* The plan's description of source behavior matches the pasted function line-for-line — branches, statuses, exact messages, cardinality.
* Every architect decision from step 3 appears in the plan.
* The named SQL, bindings, and `OracleDbType` values are explicit, not vague ("parameterize the query" is not acceptable — guide 61 §5).
* A protected-files list is present and correct.

If anything is missing or wrong, refine the prompt and re-run Plan mode. Do not move to Agent mode on a plan you have not fully checked.

## 7 Cursor Agent mode (implementation)

Switch Cursor to **Agent** mode only after the plan is approved. Use guide 61 Appendix B, again with the sibling-repo boundary line prepended:

```
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
```

Approve file-write and command prompts only when they match the approved plan. Decline and escalate to an architect for any package install, framework/SDK upgrade, connection-string edit, commit, or push (guide 61 §11).

## 8 Run verification commands

```
git diff --check
dotnet build PLSQL-Retirement.slnx
dotnet build ContractService/Tests/ContractService.Tests.csproj
dotnet run --project ContractService/Tests/ContractService.Tests.csproj
dotnet test ContractService/Tests/ContractService.Tests.csproj
```

`dotnet test` is expected to fail in the current environment because the .NET 10 SDK invokes an unsupported VSTest target for `Microsoft.Testing.Platform`. Treat `dotnet run` on the xUnit v3 project as the verified test runner and report the `dotnet test` failure separately — do not change the SDK, project, or packages to hide it. Record the passing count before and after implementation.

## 9 Cursor Ask mode final audit

Switch to **Ask** mode. Use guide 61 Appendix C:

```
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

If any section fails, return to Agent mode only for the specific correction called out — do not expand scope beyond that fix, then re-run this audit.

## 10 Commit and GitLab MR (developer-only — never ask Cursor to do this)

```
git status --short
git diff --cached --check
git diff --cached --stat
git commit -m "feat(DAB-###): concise capability description"
git push --set-upstream origin HEAD
```

Stage only the exact approved file list — no broad `git add`. Open a GitLab MR targeting `dev`, using the repository's MR template, and include build, test, warning, and deferred-integration results.

## 11 Definition of done

Confirm every line before calling the story finished (guide 61 §12):

* Implementation matches the approved Plan and source decisions.
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
* You have personally inspected the staged diff before committing.
