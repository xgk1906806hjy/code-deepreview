---
name: four-axis-review
description: >-
  Reviews a diff, branch, pull request, or named module for correctness
  (logic and behavior), security vulnerabilities, performance (latency, slow
  or large SQL, memory), and maintainability (cohesion, coupling, flow).
  Every finding needs a location, a trigger, and an impact. Use when the user
  asks for a code review, risk review, bug review, security review, four-axis
  review, 代码审查, or 四轴审查. In this calendar repository, also enforces
  GoFrame layering, DAO access, transactions, and public API contract rules.
---

# Four-Axis Code Review

Read-only review. Report findings. Do not edit code, tests, docs, or git state.

Read [axes.md](axes.md) before judging findings. Calibrate against [examples.md](examples.md).

## 1. Fix the scope

Resolve the scope once and keep it fixed.

| User asked for | Scope |
| --- | --- |
| Nothing specific | Current branch versus the merge-base with the default base branch, including staged and unstaged changes |
| A commit, range, branch, or PR | That range only |
| A file, directory, or module | That path, plus callers and callees needed to judge it |
| The whole repository or a full module audit | Entry points, state changes, auth, and persistence first. Record every area not read |

Commands:

```bash
git rev-parse --abbrev-ref HEAD
git merge-base HEAD origin/HEAD
git diff --stat <merge-base>...HEAD
git diff <merge-base>...HEAD
git status --short
```

Use `origin/HEAD` when it resolves. Otherwise use the repository's default base branch. A three-dot diff compares against the merge-base.

Include untracked source files from `git status` when the user is reviewing local work. `git diff` omits them.

Stop after one sentence when the diff is empty or the ref does not resolve. Do not invent a review of unchanged code.

A whole-repository request still needs a coverage limit. Name the entry points that were read and the areas that were skipped.

## 2. Read the change

1. Read every changed line. When a diff is truncated, open each changed file until those lines have been seen.
2. For each changed production function, read the callers and the callees that determine whether the new behavior is reachable and correct.
3. Read tests that cover the changed behavior. Missing tests are context for a finding; they are not a finding by themselves.
4. Apply the repository constraints in [axes.md](axes.md). When a constraint is ambiguous for this diff, read the matching section of `.ai/05-模块清单-职责与代码边界.md`. Do not load the whole module catalog or the whole API list.
5. When module ownership is unknown, search `docs/模块/00-模块总览.md`, then read that module's `00` quick-location section. Read `01` only for the section this change touches.

Record the base ref, head SHA, whether the worktree is dirty, and the file list.

## 3. Inventory

List the changed units that can alter behavior: functions, state transitions, public API fields, queries, transactions, jobs, and auth checks. Tag each unit with the axes that apply. Skip generated files except to check that they were not hand-edited and that generation did not pull controller, logic, or test imports into `internal/service`.

## 4. Judge

Review only the tagged units, using the checks in [axes.md](axes.md).

A finding must have all of the following:

- Axis: Correctness, Security, Performance, or Maintainability
- Priority: `P1` or `P2`
- Location: `file:line`
- Trigger: the input, state, or failure that reaches this code
- Impact: wrong behavior, a reachable vulnerability, slower response, a heavy query, unbounded memory, or a module boundary that will make the next change wrong
- Check: the smallest test or observation that would confirm it

Priority:

- `P1`: wrong behavior on a real input, a missing control on a reachable entry, a failure path that still commits, an unbounded query or allocation on a request or job path, or a public contract change with no matching spec in the reviewed scope
- `P2`: a bounded but slow or wide query, extra latency with a named cause, a recovery gap, or cohesion, coupling, or flow that splits one business operation across unrelated owners

Use `P1` and `P2` only. Priority is remediation order. It does not mean a production incident already happened.

Confidence is `high` or `medium`. A `medium` finding still needs a reachable trigger. Drop anything weaker into Excluded suspicions.

## 5. Report

Use this shape and these axis names.

```markdown
# Four-axis review

## Verdict
<One sentence: merge risk and where it concentrates.>

## Coverage
- Base: `<ref>`  Head: `<sha>`  Dirty worktree: yes/no
- Read: <entry points and files actually judged>
- Not verified: <tests not run, schema not inspected, downstream not measured>

## Findings

| ID | Axis | Priority | Location | Trigger | Impact | Check |
| --- | --- | --- | --- | --- | --- | --- |
| R01 | Correctness | P1 | path/file.go:120 | <how it is reached> | <what breaks> | <smallest confirmation> |

## Excluded suspicions
- <What looked wrong, and the evidence that removed it.>
```

When there are no findings, say so in the verdict and keep the coverage and excluded sections. An empty findings table is valid.

Sort findings `P1` first. Number `R01` upward in that order.

Redact secrets, tokens, and personal data. Use placeholder values in examples.

## Review rules

- Report. Do not patch, commit, or update docs as part of the review.
- Do not run `go test ./...`. Run a test only when the user asks to verify, and only for the affected package: `go test ./internal/logic/<module> -run <TestName>`.
- Do not recommend CI, `Jenkinsfile`, or workflow changes as a fix.
- Do not file naming, comments, or a request to make a linear function shorter.
- File slow SQL, large SQL, and excess memory when the query shape or the allocation is visible. Name what grows or why the predicate cannot use an index. A bounded indexed lookup is not a finding.
- File a security issue when a reachable entry is missing a control. Name the entry, the control, and the asset. A finished exploit is not required.
- Do not file a bug that every caller already guards.
- Do not file component outages (database, cache, queue, or broker down) unless the change mishandles a failure mode the code claims to handle.
- When no entry point reaches the code, leave it under Excluded suspicions.
- File each defect once, under the axis with the worse impact.
- Do not add findings for translation coverage, metrics, or a test-coverage percentage. A missing test is the Check column of a real finding, not a finding by itself.
- This repository keeps business rules, state machines, and transactions in `logic`. Do not recommend moving them into `service` to match generic GoFrame layout.
