# <Initiative name> Implementation Plan

<!--
  Template. See ../epic-document-formats.md.
  Copy to epics/<N>-<slug>/plan.md and delete these comments as you fill it in.
  Reference implementation: epics/1-org-bootstrap-melete-v1/plan.md
-->

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> `superpowers:subagent-driven-development` (recommended) or
> `superpowers:executing-plans` to implement this plan task-by-task. Steps use
> checkbox (`- [ ]`) syntax for tracking.

**Epic:** [`mnemosys-project/.github#N`](https://github.com/mnemosys-project/.github/issues/N)
**Spec:** [`spec.md`](./spec.md)

**Goal:** One sentence describing what this builds.

**Architecture:** Two or three sentences on the approach and the phase order —
in particular, why the phases are in the order they are.

**Tech Stack:** Key technologies, versions, and dependency limits.

## Global Constraints

The spec's project-wide requirements, one line each, with **exact values copied
verbatim from the spec**. Every task's requirements implicitly include this
section, which is what stops them being restated — or contradicted — twenty
times.

- **<Constraint>** (spec §N, decision #M).

## Placement Law

A task lives in the repo where its closing PR lands. A PR only `Closes` an issue
in its own repo; cross-repo relationships are `Ref` or comments. Every task below
names its repo, and that is where its issue is filed.

## Human-Gated Preconditions

Steps that are **not agent-performable**. Each is a precondition another task is
`Blocked-by`, attested by a human and never performed by the agent.

| Gate | Why |
|---|---|
| Repository creation | Creating a repository is a human act. |
| PR submission and merge | Standing policy: agents report ready, humans submit. |
| Releases (bump, tag, publish) | Deployment may depend on a release; the agent never cuts one. |

An agent reaching one of these stops, comments "blocked: preconditions not met",
and does not fabricate the step.

## The REFACTOR Step

State it once here rather than repeating it in every task. Red and green are
explicit in each task's steps; REFACTOR is the third beat and the one that gets
skipped unless it is written down.

- [ ] **REFACTOR (standing step for every implementation task)**
  - Extract duplicated logic — on the second occurrence, not in anticipation.
  - Move hard-coded values to configuration or a shared registry.
  - Consolidate with existing patterns rather than inventing a parallel one.
  - Improve names, then re-run the task's tests to confirm they still pass.

A task is not complete until this step has been performed and its tests are
green afterwards.

---

# Phase <A> — <Name>

What this phase delivers, and why it stands on its own.

## Task <A1>: <Component name>

**Repo:** `mnemosys-project/<repo>`
**Blocked-by:** <task or human gate, or ->

**Files:**
- Create: `exact/path/to/file.py`
- Modify: `exact/path/to/existing.py`
- Test: `tests/exact/path/to/test.py`

**Interfaces:**
- Consumes: what this uses from earlier tasks — exact signatures.
- Produces: exact names, parameter and return types that later tasks rely on.
  A task's implementer sees only their own task; this block is how they learn
  the names their neighbours use.

- [ ] **Step 1: Write the failing test**

```python
def test_specific_behavior():
    assert function(input) == expected
```

- [ ] **Step 2: Run it and confirm it fails**

Run: `vrg-container-run -- uv run pytest tests/path/test.py -v`
Expected: FAIL, with the specific error.

- [ ] **Step 3: Implement the minimum that passes**

```python
def function(value):
    ...
```

- [ ] **Step 4: Run it and confirm it passes**

- [ ] **Step 5: REFACTOR** (see the standing step), then commit

```bash
vrg-commit --type feat --scope <scope> --message "<what changed>"
```

<!--
  No placeholders. Every step contains the actual content an implementer needs.
  These are plan failures, never write them:
    - "TBD", "implement later", "add appropriate error handling"
    - "Write tests for the above" without the test code
    - "Same shape as Task N" — repeat it; tasks are read out of order
    - References to types or functions no task defines
-->

---

# Task Summary

| # | Task | Repo | Blocked-by |
|---|---|---|---|
| A1 | <name> | `<repo>` | — |

Note which tasks are genuinely parallel and which only look it.

## Spec Coverage

Every spec section maps to at least one task. A section with no task is either a
gap or a deliberate deferral, and either way it should be visible here.

| Spec section | Task |
|---|---|
| §N <name> | <task> |

## Evolution during execution

**Required. Append as the epic runs, not at the end.**

This is the source for the retrospective's §1. A plan without this log leaves
the retrospective with nothing to synthesize, and the reasoning behind every
mid-flight deviation is lost exactly when it becomes interesting.

One entry per deviation — what changed and, above all, *why*:

```markdown
- **<Task> <what changed>.** <Why the original shape did not survive contact
  with the work.>
```

An epic finishing with an empty log either went perfectly or nobody was writing
it down.
