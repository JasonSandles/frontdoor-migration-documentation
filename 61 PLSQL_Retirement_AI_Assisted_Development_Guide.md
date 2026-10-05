# **PL SQL Retirement AI Assisted Development Guide**

## *Required workflow for converting Oracle PL SQL stories with Cursor*

**Purpose**

This guide explains how developers use Cursor to plan, implement, test, review, and submit PL SQL retirement stories. AI accelerates analysis and coding, but it does not make architectural decisions. Every change must preserve the legacy routine’s observable behavior, follow the repository rules, and pass human review before it is committed.

| Document item | Value |
| :---- | :---- |
| Audience | Developers, reviewers, and architects working in the PL SQL Retirement repository |
| Primary tool | Cursor using Plan, Agent, and Ask modes |
| Target platform | .NET 8 with the approved synchronous OracleDAL package |
| Required test framework | xUnit v3 in the corresponding domain test project |
| Version date | September 16, 2026 |

&nbsp;

**Main rule**

Do not ask Cursor to implement a story immediately. First establish the source behavior and metadata, obtain a read-only plan, review that plan, then authorize a tightly bounded implementation.

# **Contents**

1  How AI is used in this project

2  Nonnegotiable architecture rules

3  Information required before planning

4  The required Cursor workflow

5  Writing effective Cursor prompts

6  Source fidelity and architectural decisions

7  Data access and type mapping

8  Testing and verification

9  Git and GitLab workflow

10 Security and exception handling

11 Common problems and responses

12 Definition of done

Appendix A Reusable Plan prompt

Appendix B Reusable Agent prompt

Appendix C Reusable final review prompt

# **1 How AI is used in this project**

Cursor is used as an engineering assistant for repository inspection, source-to-.NET mapping, test design, code generation, and read-only review. The developer remains responsible for the story, the source interpretation, the approvals requested by Cursor, and the final code submitted to GitLab.

| Participant | Responsibility |
| :---- | :---- |
| Architect | Approves framework, architecture, source deviations, status meanings, data types, dependencies, and security decisions. |
| Developer | Collects the complete story and PL SQL, verifies metadata, controls Cursor modes, reviews diffs, runs tests, and owns the merge request. |
| Cursor | Inspects code, proposes a bounded plan, implements only approved scope, runs verification, and reports findings. |
| Reviewer | Confirms source fidelity, architecture, tests, security, and scope before approving the merge request. |

&nbsp;

AI output is evidence to review, not authority to accept. A successful build does not prove that Oracle binding, cardinality, null semantics, status mapping, or error messages are correct.

# **2 Nonnegotiable architecture rules**

* All projects remain on .NET 8 unless architects approve a framework change.  
* Continue using Swashbuckle. Do not replace it with AddOpenApi or MapOpenApi without approval.  
* Use the approved custom ADO.NET-based OracleDAL NuGet package. Do not introduce Entity Framework or Dapper.  
* PL SQL business logic moves into .NET. Oracle SQL belongs in approved Infrastructure Oracle constant classes.  
* Do not create replacement stored procedures, functions, views, or other database objects.  
* Controller constructors continue creating RepositoryWrapper through the approved pattern. Do not inject RepositoryWrapper through DI.  
* RepositoryWrapper owns OracleDAL initialization and connection access. Developers do not read or change connection strings.  
* Connection-string changes require architect approval after consultation with the other architects.  
* Use synchronous service and repository methods when the underlying OracleDAL operation is synchronous. Do not use fake async wrappers.  
* Use xUnit v3 and place tests in the corresponding domain test project.  
* Do not add packages, analyzers, authentication frameworks, or target-framework changes without architect approval.  
* Preserve return codes, nullability, precision, output values, exact messages, and terminal control flow unless a documented architect decision says otherwise.

# **3 Information required before planning**

A story is not ready for Cursor Plan mode until the developer has assembled the following evidence.

| Required input | What to provide |
| :---- | :---- |
| Jira story | Story number, title, and complete acceptance criteria. |
| Legacy source | The complete function or procedure, including exception handlers and referenced constants. |
| Target | Domain service and whether the capability is internal or exposed through an approved endpoint. |
| Oracle metadata | Data type, length, precision, scale, and nullability for every referenced input, output, filter, join, and selected column. |
| Dependencies | Source for called routines or the exact existing .NET replacement and its result behavior. |
| Decisions | Architect-approved resolutions for conflicts, ambiguity, nondeterminism, unsafe messages, and type differences. |
| Branch state | Prerequisite stories merged, dev pulled, feature branch created from updated dev, and clean working tree. |

&nbsp;

## **Oracle metadata query pattern**

SELECT table\_name, column\_name, data\_type, data\_length,  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;data\_precision, data\_scale, nullable  
&nbsp;&nbsp;FROM all\_tab\_columns  
&nbsp;WHERE table\_name \= 'TABLE\_NAME'  
&nbsp;&nbsp;&nbsp;AND column\_name IN ('COLUMN\_1', 'COLUMN\_2')  
&nbsp;ORDER BY column\_id;

Use ALL\_TAB\_COLUMNS when the object belongs to another schema or is reached through a synonym. Do not guess a .NET type from a PL SQL parameter name.

# **4 The required Cursor workflow**

| Gate | Cursor mode | Required outcome |
| :---- | :---- | :---- |
| 1 Source analysis | Ask or manual review | Conflicts and missing metadata are identified before coding. |
| 2 Read-only plan | Plan | Cursor inspects the actual repository and lists exact files, contracts, SQL, tests, risks, and protected scope. |
| 3 Plan approval | Human review | Developer confirms the plan matches the source and every architect decision. |
| 4 Implementation | Agent | Cursor implements only the approved plan using TDD and runs verification. |
| 5 Final audit | Ask | Cursor performs a read-only review of the actual diff and returns PASS or FAIL by section. |
| 6 Commit and MR | Terminal and GitLab | Developer stages exact files, validates the staged diff, commits, pushes, and opens an MR to dev. |

&nbsp;

## **4.1 Preflight before every story**

git branch \--show-current  
git status \--short  
dotnet run \--project ContractService/Tests/ContractService.Tests.csproj

* The branch must be the intended feature branch, not dev or main.  
* The working tree must be clean before implementation begins.  
* All prerequisite stories must be present.  
* The existing test baseline must pass. Record the exact count.  
* Stop if the baseline fails or the branch is missing a dependency.

## **4.2 Plan mode**

Plan mode is read-only. The prompt must instruct Cursor to inspect the actual source, repository contracts, readers, test fakes, OracleDAL overloads, and current rules. The plan must name every file to create or modify and explicitly list protected files that will remain unchanged.

## **4.3 Agent mode**

Agent mode is authorized only after the plan is reviewed. The implementation prompt restates the approved contracts, messages, SQL, bindings, tests, scope restrictions, and verification commands. Cursor must stop before expanding scope.

Approve file-write and command prompts only when they match the authorized task. A prompt to create a required directory or write an approved file is expected; a package install, framework upgrade, connection-string edit, commit, or push is not.

## **4.4 Ask mode final review**

The final audit must review the actual files and git diff, not repeat the implementation summary. It must inventory tests, verify exact messages and bindings, report git status, and end with either Ready to commit or Corrections required before commit.

# **5 Writing effective Cursor prompts**

Every story prompt should make the following boundaries explicit.

| Prompt element | Required content |
| :---- | :---- |
| Contracts | Exact service and repository signatures, including nullability and sync or async behavior. |
| Behavior | Every input branch, dependency branch, status, value, message, and terminal return. |
| SQL | Exact tables, joins, filters, constants, aggregates, cardinality, and named parameters. |
| Oracle bindings | Parameter names, values, OracleDbType values, and call order. |
| Mapping | Zero-row, one-row, multiple-row, DBNull, missing-column, precision, and casing behavior. |
| Tests | Specific service, reader, SQL, exception, regression, and terminal-flow assertions. |
| Protected scope | Files and architecture that must not change. |
| Verification | Build, test, diff, and status commands plus exact reporting requirements. |

&nbsp;

## **Use concrete language**

* Write “preserve the exact message” and include the literal, including any trailing space.  
* Write “negative IDs reach Oracle unchanged” when the source rejects only null and zero.  
* Write “do not add ORDER BY” when the source intentionally has no ordering.  
* Write “multiple rows throw” or “ROWNUM selects one arbitrary row” according to the actual source.  
* Write “OracleDbType.Decimal” or “OracleDbType.Varchar2” rather than saying only “parameterize the query.”  
* Write “do not commit or push” in every implementation prompt.

# **6 Source fidelity and architectural decisions**

The source and acceptance criteria can disagree. Do not allow Cursor to choose silently. Record the conflict, obtain an architect decision, and place that decision directly in the Plan and Agent prompts.

| Common conflict | Required response |
| :---- | :---- |
| AC says greater than zero; source rejects only null and zero | Ask whether negative values must be preserved. Do not strengthen validation without approval. |
| AC says async; OracleDAL method is synchronous | Use a synchronous method with honest naming. Do not wrap the call in Task.Run or Task.FromResult. |
| NUMBER maps to long in AC but precision is NUMBER(20,0) | Use decimal/decimal? to avoid losing values above long.MaxValue. |
| Aggregate returns null | Distinguish an aggregate row containing DBNull from a missing result row. |
| SELECT INTO may return duplicates | Preserve TOO\_MANY\_ROWS behavior unless the source uses ROWNUM to select one. |
| Source message contains SQLERRM | Log details internally and use the approved safe exception response externally. |
| Source has unreachable exception branch | Document and test the distinguishable path without pretending Oracle normally reaches it. |
| Called routine has its own story | Implement and merge dependencies first rather than creating a temporary replacement. |

&nbsp;

## **Dependency sequencing**

Before implementing a story that calls another legacy routine, determine whether the dependency already exists in .NET or has its own Jira story. Pause the dependent story when necessary. Build and merge dependencies in source order so temporary logic is not introduced and later rewritten.

# **7 Data access and type mapping**

| Oracle type or behavior | Project mapping |
| :---- | :---- |
| NUMBER(20,0) identifier | decimal or decimal?; bind with OracleDbType.Decimal |
| DATE | DateTime or DateTime?; bind with OracleDbType.Date |
| VARCHAR2 | string or string? based on metadata; bind with OracleDbType.Varchar2 |
| CHAR | string or string?; preserve source padding unless approved otherwise |
| DBNull | Map according to column nullability and source branch; do not silently substitute defaults |
| SELECT INTO zero rows | Map to the exact source-defined result |
| SELECT INTO multiple rows | Throw deterministically unless ROWNUM or an approved selection rule prevents it |
| COUNT aggregate | Expect one aggregate row; distinguish count zero from an anomalous missing aggregate row |

&nbsp;

Repository methods must call ClearSQLParameters before binding, bind every value by name with an explicit Oracle type, execute the approved SQL constant, and pass the DataTable to a focused reader. Repository code must not catch general exceptions, access connection strings, or dispose an OracleDAL instance it does not own.

# **8 Testing and verification**

## **Required test layers**

| Layer | What it proves | What it does not prove |
| :---- | :---- | :---- |
| Service tests | Validation, dependency order, statuses, values, exact messages, terminal behavior, and exception translation. | Live Oracle behavior or real parameter binding. |
| Reader tests | DataTable cardinality, DBNull, missing columns, aliases, casing, and precision. | Actual provider metadata and live result shapes. |
| SQL tests | Required SQL text, named binds, and absence of forbidden clauses. | That OracleDAL binds or executes correctly. |
| Integration tests | Real binding, filters, Oracle types, result metadata, duplicates, and database failures. | They are deferred unless the story explicitly authorizes a database environment. |

&nbsp;

## **Required verification commands**

git diff \--check

dotnet build PLSQL-Retirement.slnx

dotnet build ContractService/Tests/ContractService.Tests.csproj

dotnet run \--project ContractService/Tests/ContractService.Tests.csproj

dotnet test ContractService/Tests/ContractService.Tests.csproj

In the current environment, dotnet test can fail because the .NET 10 SDK invokes an unsupported VSTest target for Microsoft.Testing.Platform. Do not change the SDK, project, packages, or framework inside a story to hide that toolchain issue. Use dotnet run on the xUnit v3 test project as the verified test runner and report the dotnet test failure separately.

## **Test baseline rule**

Record the passing count before implementation and after implementation. A higher final count does not replace coverage review. The final Ask-mode audit must inventory the new tests and confirm that consolidated tests contain all required assertions.

# **9 Git and GitLab workflow**

1. Merge prerequisite stories into dev and pull the updated dev branch.  
2. Create the feature branch from updated dev using the Jira number and a descriptive name.  
3. Confirm git status is clean before asking Cursor to implement.  
4. After the final review passes, stage the exact approved file list. Do not use a broad add when unrelated changes exist.  
5. Run git diff \--cached \--check and inspect git diff \--cached \--stat.  
6. Commit with a Jira-scoped message such as feat(DAB-250): implement contract existence check.  
7. Push the feature branch and create a GitLab merge request targeting dev.  
8. Complete the repository merge-request template and include build, test, warning, and deferred-integration results.

git status \--short  
git diff \--cached \--check  
git diff \--cached \--stat  
git commit \-m "feat(DAB-\#\#\#): concise capability description"  
git push \--set-upstream origin HEAD

Do not ask Cursor to commit, push, or create the merge request. Those are developer-controlled actions after the read-only review returns Ready to commit.

# **10 Security and exception handling**

* Do not return exception.Message, SQLERRM, SQL, connection details, stack traces, internal paths, or Oracle-specific details to clients.  
* Preserve the complete exception for internal logging through the registered handlers.  
* Use the established OthersErrorException path for unexpected service or repository failures.  
* Do not rewrap an existing OthersErrorException.  
* Do not add redundant general try/catch blocks to repositories.  
* Keep handler registration order and Program.cs unchanged unless the story explicitly authorizes a change.  
* Do not add authentication or authorization behavior as incidental work.

# **11 Common problems and responses**

| Problem | Response |
| :---- | :---- |
| Cursor is in Plan but edited files | Stop, inspect git status, discard nothing automatically, and determine exactly what changed before continuing. |
| Agent asks to write an approved file | Approve only if the path and file are in the approved plan. |
| Agent asks to install a package or upgrade .NET | Decline and stop. Escalate to an architect. |
| Baseline tests fail | Do not implement. Diagnose the branch, dependencies, environment, and existing failures first. |
| NuGet restore returns 401 | Verify the configured GitLab token and source. Never commit credentials. |
| LF will be replaced by CRLF warning | This is normally a line-ending notice on Windows, not a failed add. Verify .gitattributes and the staged diff. |
| Swagger shows only one service | Use the approved compound debug configuration and separate service ports. |
| Cursor proposes fake async | Reject it. Use synchronous naming and calls when OracleDAL is synchronous. |
| Story depends on an unimplemented routine | Pause the story and implement or merge the dependency first. |
| Live Oracle behavior is not tested | Document the exact deferred integration checks. Do not claim source-text tests prove runtime behavior. |

&nbsp;

# **12 Definition of done**

A story is ready to commit only when every item below is true.

* The implementation matches the approved Plan and source decisions.  
* All intended files and only intended files are changed.  
* Protected architecture and configuration files are unchanged.  
* Exact status codes, values, nullability, and messages are covered by tests.  
* Oracle SQL is parameterized and stored in the approved SQL constants class.  
* Reader behavior covers zero, one, multiple, DBNull, missing-column, casing, and precision cases where applicable.  
* The complete solution builds and all warnings are reported.  
* The domain test project builds and the xUnit v3 runner passes with exact counts reported.  
* git diff \--check is clean.  
* The final Ask-mode review marks every section PASS and ends with Ready to commit.  
* Deferred Oracle integration checks are listed honestly.  
* The developer has inspected the staged diff before committing.

# **Appendix A Reusable Plan prompt**

Replace bracketed values with story-specific facts. Include the complete source and decisions after this core prompt.

Create a read-only implementation plan for \[JIRA NUMBER\] \[ROUTINE\].

Do not modify, create, delete, move, format, stage, commit, or push files.

Inspect the actual repository, current branch, git status, rules, existing  
patterns, installed OracleDAL API, and tests before proposing changes.

Preflight:  
\- Confirm the feature branch and clean working tree.  
\- Confirm prerequisite stories are present.  
\- Run the current domain test project and record the exact baseline.  
\- Stop on an incorrect branch, dirty tree, missing dependency, or failed baseline.

Plan the exact service and repository contracts, SQL, named binds,  
OracleDbType values, reader cardinality, status/value/message mapping,  
exception behavior, tests, fake-interface updates, protected files, verification,  
and deferred integration checks.

Show the plan only. Do not implement anything.

# **Appendix B Reusable Agent prompt**

Implement the approved \[JIRA NUMBER\] plan exactly as written.

Repeat preflight and stop if the branch, clean state, dependencies, or baseline  
do not match the approved plan.

Follow TDD. Implement only the approved contracts, SQL, bindings, reader,  
service behavior, and tests. Preserve exact source messages and terminal flow.  
Do not change protected files or expand scope.

Run git diff \--check, the complete solution build, the domain test build,  
dotnet run for the xUnit v3 project, and dotnet test. Report the known test  
toolchain conflict separately without changing configuration.

Show every created and modified file, exact test totals, warnings, git status,  
protected-file confirmation, and deferred integration checks.

Do not commit, push, or create a merge request.

# **Appendix C Reusable final review prompt**

Perform a final read-only review of the uncommitted \[JIRA NUMBER\]  
implementation.

Do not modify, format, stage, commit, or push files. Review the actual source  
and git diff, not only the implementation summary.

Mark each section PASS or FAIL and cite exact files and lines for failures:  
1\. Method contracts  
2\. Input validation and dependency order  
3\. SQL fidelity  
4\. OracleDAL parameter binding  
5\. Reader cardinality and precision  
6\. Service status, value, nullability, and exact messages  
7\. Exception translation and security  
8\. Regression behavior  
9\. Test inventory and coverage  
10\. Architecture and protected scope  
11\. Worktree, build, warnings, and exact test results  
12\. Deferred Oracle integration

End with exactly one verdict:  
\- Ready to commit  
\- Corrections required before commit

Do not implement corrections.