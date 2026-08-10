# <Epic name> — Retrospective

<!--
  Template. See ../epic-document-formats.md.
  Copy to epics/<N>-<slug>/retrospective.md and delete these comments.

  Authored with the vergil:epic-retrospective skill, whose preflight refuses to
  run until every other child of the epic is closed. The section structure below
  matches that skill exactly - do not reorder or rename sections.

  Short and scannable up top, detail below. Honest, not a victory lap:
  sections 3 and 4 are load-bearing.
-->

**Epic:** [`mnemosys-project/.github#N`](https://github.com/mnemosys-project/.github/issues/N)
**Spec:** [`spec.md`](./spec.md) · **Plan:** [`plan.md`](./plan.md)
**Date:** YYYY-MM-DD

## §0 At a glance

<!--
  Half a page maximum. This is the landing view: here is the bulk of what was
  done.

  QUERIED, NEVER INVENTED. Assemble from vrg-epic-audit for the task and PR
  graph, and vrg-gh for PR titles, merge dates, and releases. Anything you
  cannot determine is marked "unknown" - never guessed. A retrospective that
  invents its own PR list is exactly the failure this rule prevents.
-->

One paragraph: what we set out to do, and what actually shipped.

### Work delivered

| PR | What it did |
|---|---|
| #N | <one line> |

| | |
|---|---|
| **Repos touched** | <list> |
| **Tasks closed** | <count> |
| **PRs merged** | <count> |
| **Releases cut** | <list, or none> |
| **Opened → closed** | YYYY-MM-DD → YYYY-MM-DD (<N> days) |

## §1 How the plan evolved

A **synthesized narrative** from the plan's *Evolution during execution* log —
the reasoning behind the deviations, not a copy of the entries.

What survived contact with the work, what did not, and what that says about how
the plan was written. If the plan barely changed, say so and say why; that is
itself a finding.

## §2 Lessons learned

Transferable insight. What would be done the same way again, and what
differently.

Prefer lessons that would change a future decision over observations that merely
describe this one. "The generator honours nouns and drops predicates" is a
lesson; "image generation was fiddly" is not.

## §3 Compromises & tradeoffs

**Load-bearing. An empty section here means an unwritten retrospective, not a
clean epic.**

Corners cut, debt knowingly incurred, and the reasoning at the time. Include the
things that were right to do under the circumstances — a compromise is not an
admission of failure, and recording it is how the next person knows it was a
choice rather than an oversight.

## §4 New problems & opportunities

**Also load-bearing.**

What the epic surfaced that was not visible when it started. For each, say where
it went:

| Surfaced | Where it went |
|---|---|
| <finding> | <spun-off epic or task ref, or "logged, not yet acted on"> |

"Logged, not yet acted on" is an acceptable answer. Silently dropping it is not.

## §5 What's next

Pointers to follow-on brainstorms or epics — **referenced, not duplicated**. If
a follow-on epic exists, link it; if it is only an intention, say that plainly.

## Appendix A — Operational notes

<!--
  Optional. Migration, deployment, or validation-heavy epics only.
  Delete this heading entirely if it does not apply.
-->

The mechanical sequence: repo by repo, publish and deploy order, and the gotchas
worth knowing before doing it again.

## Appendix B — Extended metrics

<!--
  Optional. Delete if it does not apply.
-->

Deeper statistics worth keeping.
