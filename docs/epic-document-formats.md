# Epic Document Formats

**Established:** 2026-08-10
**Status:** Standard. Every epic in this organization uses these formats.

An epic carries three documents, and they are not interchangeable. A reader
follows them in order, and each answers a different question:

| Document | Question it answers | Written | Closed by |
| --- | --- | --- | --- |
| [`spec.md`](templates/spec.md) | **What** are we building, and **why**? | Before implementation, from a brainstorm | The documentation bookend task |
| [`plan.md`](templates/plan.md) | **How**, and in **what order**? | After the spec, before code | The same documentation PR |
| [`retrospective.md`](templates/retrospective.md) | What **actually happened**? | After every other task closes | The retrospective task, which closes the epic |

All three live at `epics/<N>-<slug>/` in this repository, where `<N>` is the
epic issue number and `<slug>` is 2–4 kebab-case tokens.

## Why standardize

The value is compounding rather than immediate. A reader who has followed one
epic should be able to follow every subsequent one without relearning the
layout, and an agent picking up an epic should be able to find a section by
name rather than by searching.

Epic [#1](https://github.com/mnemosys-project/.github/issues/1) is the reference
implementation. Where a template and epic #1 disagree, the template is right —
epic #1 was written first and the format was extracted from it afterwards.

## The rules that are not negotiable

### Recorded Decisions are mandatory in every spec

A decision table — what was decided and *why* — is the highest-value section in
a spec and the one most likely to be skipped. It is what makes a spec readable a
year later, and it is what stops a settled question being reopened by someone
who was not in the room.

Record the reasoning, not just the choice. "Hand-rolled IR over abjad" is a
note; "hand-rolled IR over abjad, because the fretboard layer is the tool's core
value and no library provides it" is a decision.

Decisions accumulate. When a review adds new ones — a `paad:pushback` pass, an
alignment check — append them with a subheading naming their source rather than
renumbering the existing table.

### `plan.md` carries an Evolution during execution log

**This section is load-bearing and easy to omit.** The retrospective's §1 is a
synthesized narrative of *why the plan changed*, and its source is this log. A
plan without one leaves the retrospective with nothing to synthesize, and the
reasoning behind every mid-flight deviation is lost at exactly the moment it
becomes interesting.

Append to it as the epic runs, not at the end. One entry per deviation:

```markdown
### Evolution during execution

- **Task B10 split into B10 and B10a.** The weighting function and the sampler
  turned out to be separately reviewable, and holding them in one task meant a
  reviewer had to accept both or neither.
```

An epic that finishes with an empty Evolution log either went perfectly or
nobody was writing it down. The second is far more common.

### The retrospective is honest, not a victory lap

§3 (compromises and tradeoffs) and §4 (new problems) are load-bearing. A
retrospective with an empty §3 is not a clean epic; it is an unwritten one.

### §0 is queried, never invented

The retrospective's "At a glance" section is assembled from `vrg-epic-audit` and
`vrg-gh` — real PR titles, real merge dates, real counts. Anything that cannot
be determined is marked **unknown**. A retrospective that guesses at its own PR
list is precisely the failure this rule exists to prevent.

## Formatting conventions

These apply to all three documents and to org-level documentation generally:

- **Markdown, wrapped at 80 columns.** Long tables and code blocks may exceed it.
- **A table of contents** for any document over roughly 200 lines.
- **Section numbering** in specs (`## 1. Overview`), so decisions and reviews can
  cite `§7` unambiguously and the reference survives edits elsewhere.
- **Relative links between epic documents**; absolute GitHub URLs when linking
  from anything that renders outside the repository, such as `profile/README.md`.
- **State the date in full** — `2026-08-10`, never "last Tuesday".

## Related

- [`NAMING.md`](../NAMING.md) — the naming convention, which governs any name
  these documents introduce.
- The `vergil:epic-create`, `vergil:epic-implement`, and
  `vergil:epic-retrospective` skills, which drive the lifecycle these documents
  record.
