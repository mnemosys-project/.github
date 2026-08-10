# <Tool or initiative name> — <one-line descriptor>

<!--
  Template. See ../epic-document-formats.md.
  Copy to epics/<N>-<slug>/spec.md and delete these comments as you fill it in.
  Reference implementation: epics/1-org-bootstrap-melete-v1/spec.md
-->

**Design specification, v1.0**
**Date:** YYYY-MM-DD
**Org:** `mnemosys-project`
**Repository:** `mnemosys-project/<repo>`
**Epic:** [`mnemosys-project/.github#N`](https://github.com/mnemosys-project/.github/issues/N)

## Table of Contents

<!-- Required once the document exceeds roughly 200 lines. -->

- [1. Overview](#1-overview)
- [2. Scope](#2-scope)
- [3. Architecture](#3-architecture)
- [N. Error Handling](#n-error-handling)
- [N. Testing Strategy](#n-testing-strategy)
- [N. Recorded Decisions](#n-recorded-decisions)
- [N. Deferred](#n-deferred)

## 1. Overview

What this is, in a paragraph a reader can absorb before any detail.

**State the success criterion concretely.** Not "improves practice" but "run one
command each morning and get a printable practice sheet good enough to hand to
an instructor." Someone must be able to hold the finished thing up against this
sentence and say yes or no.

If the work descends from something earlier, say what it inherits and — more
usefully — what it explicitly does not.

## 2. Scope

### In scope

The deliverables, as a list.

### Non-goals

Excluded deliberately, not overlooked. This section prevents the most expensive
kind of disagreement, which is the one discovered late. Say *why* each exclusion
is an exclusion; some belong in a later version, others never belong at all.

### Audience

Who uses this, and what that implies about the design. A tool for its author can
make assumptions a tool for strangers cannot.

## 3. Architecture

The shape of the thing: components, how they fit, and the module layout.

**Name the load-bearing boundaries and say what each one buys.** A boundary that
exists for a reason should have that reason written next to it — which module is
the only one that knows about an external dependency, which seam lets two halves
be tested independently. These are the parts a later change is most likely to
violate by accident.

## <N>. <Domain sections>

As many numbered sections as the subject needs — the data model, the algorithms,
the interfaces, the configuration, the output. Number them so reviews and
decisions can cite `§7` and have it stay meaningful.

Where a section embeds a decision, say so inline and carry it into the decision
table. A reader should not have to reconstruct intent from mechanism.

## <N>. Error Handling

A table of failure modes and required behaviour.

| Failure | Behavior |
|---|---|
| <what goes wrong> | <what must happen — fail loudly, name the key, never fall back> |

**No swallowed exceptions and no fallbacks that hide errors.** Failure at the
code layer becomes a wrong answer at the user layer, which is worse than no
answer. If a fallback is genuinely correct, justify it here or do not write it.

## <N>. Testing Strategy

| Component | Approach |
|---|---|
| <module> | <exhaustive / property-based / golden-file / one integration test> |

Name the **central invariant** if there is one — the single property whose
violation catches most bugs in this system. Then name the one test that proves
the hardest claim in the spec, and say which claim it proves.

## <N>. Recorded Decisions

**Mandatory.** See the standard.

| # | Decision | Rationale |
|---|---|---|
| 1 | <what was decided> | <why — the reasoning, not a restatement> |

### Resolutions from <review name>

<!--
  Append later reviews as their own subheading with their own table, continuing
  the numbering. Do not renumber the original table.
-->

| # | Decision | Rationale |
|---|---|---|
| N | <what the review settled> | <what would have gone wrong otherwise> |

## <N>. Deferred

Planned but out of scope, named so the boundary is explicit and so the design
can leave room for each. Distinguish "next version" from "never".

---

**Status:** <Approved YYYY-MM-DD. Filed as epic #N.>
