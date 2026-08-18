# Graded exercise ladders

**Design specification, v1.0**
**Date:** 2026-08-18
**Org:** `mnemosys-project`
**Repository:** `mnemosys-project/melete`
**Epic:** [`mnemosys-project/.github#87`](https://github.com/mnemosys-project/.github/issues/87)

## Table of Contents

- [1. Overview](#1-overview)
- [2. Scope](#2-scope)
- [3. The model: identity, deviation, ladder](#3-the-model-identity-deviation-ladder)
- [4. The per-family declaration](#4-the-per-family-declaration)
- [5. The draw](#5-the-draw)
- [6. The validity gate](#6-the-validity-gate)
- [7. Configuration](#7-configuration)
- [8. The session record](#8-the-session-record)
- [9. The sheet](#9-the-sheet)
- [10. Error handling](#10-error-handling)
- [11. Testing strategy and acceptance](#11-testing-strategy-and-acceptance)
- [12. Recorded decisions](#12-recorded-decisions)
- [13. Deferred](#13-deferred)

## 1. Overview

melete samples every axis of an exercise independently. `pattern`,
`accent_pattern`, `note_value_pattern`, `inversion`, `shift` and the tapping
derivation are each drawn on their own recency schedule, and a slot is accepted
as soon as the combination realizes and fits. Nothing in the model expresses
that some of those axes are *harder than others*, and nothing expresses that
their plain values are a legitimate exercise in their own right. The
consequence is visible on every sheet: the draw lands at the top of the
difficulty range, always, because the plain value of an axis is just one
candidate among several and the odds of drawing five plain values at once are
negligible.

The `2026-08-18` session is the worked example. Its second slot is C Ionian,
three-notes-per-string, **groups of three, long-short note values, accent every
three**. That is a good exercise and a correct realization of its parameters. It
is also the only C Ionian on the sheet: bare C Ionian three-notes-per-string —
a demanding stretch drill on its own — is never played, and neither is any
intermediate step between it and the compound exercise that was drawn.

This epic replaces the one-spec slot with a **ladder**: an ordered series of
exercises sharing one identity, beginning at the bare shape and accreting one
technique per rung until it reaches the compound exercise the selector would
have drawn today, plus one rung beyond it. The second goal is equal to the
first: the *route* up the ladder must rotate. If there are ten ways to play a
scale, a week of sessions should walk different subsets of them in different
orders, so the journey into the advanced version is a different journey each
time rather than a fixed curriculum replayed with new roots.

**Success criterion.** With the default configuration, `melete generate`
produces a sheet on which every family's slot reads as a graded page: rung 1 is
a shape the player can already play, each later rung adds exactly one named
technique to the *same* shape, the last ordinary rung is comparable in
difficulty to what melete generates today, and the challenge rung is one the
player is not expected to manage. Held against the sheet, the player can point
at any rung and say what was added to the rung before it.

**Non-goal, stated up front.** This epic does not tune difficulty. Day length,
rung counts and which axis belongs to which tier are all left at deliberately
untuned defaults so the full spectrum can be seen and played before any of it
is calibrated. See [§12](#12-recorded-decisions), decision L8.

## 2. Scope

### In scope

- The identity/deviation split, and a `LadderSpec` alongside `ExerciseSpec`.
- A per-family `Ladder` declaration — `IDENTITY`, `PLAIN`, `TIERS`, `ELIGIBLE` —
  registered beside the existing `Family` record.
- The ladder draw in `selection.py`, including the new `deviation` pseudo-axis
  and its recency weighting.
- A measurement of the ladder validity rate, run **before** the gate is
  finalized.
- `[ladder]` and `[challenge.<family>]` configuration, validated in `config.py`.
- The reshaped `session.json`, and `replay` against it.
- Two-level exercise numbering and rung titling in `emit.py`, `--dry-run`,
  `show` and `replay`.

### Out of scope (deferred, named — §13)

- Technique-weighted history. The `deviation` axis is built as its seam; the
  weight function is unchanged.
- Progressive tempo overload (decision #20 keeps tempo out of the sampled axes).
- Open-string scale variants.
- The one-hand triad defect
  ([`melete#227`](https://github.com/mnemosys-project/melete/issues/227)).
- Per-deviation repair in the validity gate, unless [§6](#6-the-validity-gate)'s
  measurement demands it.

### Renderer boundary

This epic does not cross it. A ladder is a selection-time concept; the emitter's
only change is the *text* of a `\section` marker, which it already owns. No new
module knows the renderer exists, and `melete-render/` is untouched.

### Audience

The reader is a melete contributor who knows the spec's §7 (families), §9
(selection) and §12 (session log), and `docs/design.md`.

## 3. The model: identity, deviation, ladder

An exercise's axes divide in two.

**Identity** is what the page is *about*. It is fixed for every rung of the
ladder, and it is what the player would name if asked what they practised: the
root, the scale type, the traversal, the chord quality, the interval. Identity
axes have no plain value — there is nothing plainer about C than about F♯.

**Deviations** are how the identity is *played*. Every deviation axis has a
distinguished **plain** value, and a rung is defined by exactly which deviations
have left plain. Deviation axes are the difficulty surface.

A **ladder** is the monotone chain from all-plain to the top: rung 1 deviates in
nothing, and each later rung adds one deviation to the set the rung before it
carried. Monotonicity is load-bearing rather than incidental — it is what makes
"what was added" a well-defined question at every rung, and it is what forbids a
ladder from switching a technique off again ([§4](#4-the-per-family-declaration),
`ELIGIBLE` exclusions).

The provisional split, per family:

| Family | Identity | Deviations |
|---|---|---|
| `scales` | `root`, `scale_type`, `traversal` | `pattern`, `hands`, `accent_pattern`, `note_value_pattern` |
| `arpeggios` | `root`, `quality` | `inversion`, `hands`, `pattern`, `accent_pattern`, `note_value_pattern` |
| `intervals` | `root`, `interval`, `context` | `string_skip`, `pattern`, `accent_pattern`, `note_value_pattern` |
| `chromatic` | `start_string`, `start_fret`, `span` | `permutation`, `string_traversal`, `shift`, `accent_pattern`, `note_value_pattern` |

Two of these placements are judgment calls and are recorded as such.
`traversal` is identity because the stretch fingering *is* the exercise — bare
C Ionian three-notes-per-string is a drill, not a plain form of something else,
and the positional version of the same scale is a different page rather than an
easier rung of the same one. `inversion` is a deviation **provisionally**; the
argument for calling it identity (a first-inversion m7b5 is a different shape to
learn, not a technique applied to a shape) is real, and the expectation is that
the question answers itself as more shape families land. See
[§12](#12-recorded-decisions), decision L3.

### `LadderSpec`

```python
@dataclass(frozen=True)
class LadderSpec:
    family: str
    identity: dict[str, AxisValue]
    rungs: tuple[dict[str, AxisValue], ...]      # cumulative; rungs[0] == {}
    challenge: dict[str, AxisValue] | None
```

`rungs` holds the deviations active at each rung, **cumulatively resolved**
rather than as a diff to be recomposed. `challenge` is a separate field, not the
last element of `rungs`, so a reader can identify the level++ exercise without
consulting the configuration that produced it.

### Materialization

Each rung becomes an ordinary `ExerciseSpec`:

```
params = identity | {axis: PLAIN[axis] for axis in deviation_axes} | rungs[k]
params |= REGISTRY[family].derive(params, tapped)
```

This is the containment decision the whole epic rests on. **A ladder is a
selection-time concept only.** `pipeline.realize`, `layout.py`, `rhythm.py`,
`score.py`, every family's `generate`, and the emitter's per-exercise engraving
receive exactly what they receive today — one `ExerciseSpec` per exercise — and
need no knowledge that ladders exist. The blast radius is `ladder.py`,
`families/__init__.py`, `selection.py`, `config.py`, `session.py`, and the
titling paths in `emit.py` and `cli.py`.

## 4. The per-family declaration

Each family declares four things beside its existing `AXES` tuple. They join
`families.Family` as a sibling record, `Ladder`, holding *references* to the
family's own declarations — never copies. This is the pattern `Family`'s
docstring already argues for: adding a fifth family means one line of `REGISTRY`
naming the module, and nothing anywhere else.

```python
@dataclass(frozen=True)
class Ladder:
    identity: tuple[str, ...]
    plain: Mapping[str, AxisValue]
    tiers: tuple[frozenset[str], ...]
    eligible: Eligibility
```

**`identity`** — the axes fixed across the ladder.

**`plain`** — each deviation axis's plain value. `pattern: "straight"`,
`accent_pattern: "none"`, `note_value_pattern: "straight"`, `inversion: "root"`,
`shift: "none"`, `string_traversal: "adjacent"`, `string_skip: "0"`,
`permutation: (1, 2, 3, 4)`, `hands: 1`.

**`tiers`** — an ordered partition of the deviation axes. Order is drawn *within*
a tier and never across, so a ladder can rotate its route while remaining
monotone in difficulty. The provisional tiers:

| Family | Tier 1 | Tier 2 | Tier 3 |
|---|---|---|---|
| `scales` | `pattern` | `hands` | `note_value_pattern`, `accent_pattern` |
| `arpeggios` | `inversion`, `pattern` | `hands` | `note_value_pattern`, `accent_pattern` |
| `intervals` | `string_skip`, `pattern` | — | `note_value_pattern`, `accent_pattern` |
| `chromatic` | `permutation` | `string_traversal`, `shift` | `note_value_pattern`, `accent_pattern` |

**`eligible`** — the preconditions and exclusions that make the family the
authority on what can escalate into what. Two kinds:

- **Preconditions on the identity.** `scales` offers `hands` as a deviation only
  when `traversal == "three_note_per_string"` *and* the drawn `scale_type` is in
  `[pool.scales] tapped_scale_types`; a positional ladder simply never has a
  tapping rung and takes its deviations from pattern and rhythm instead.
  `arpeggios` offers `hands` only for a quality the pool has opted into tapping.
- **Exclusions between deviations.** `arpeggios` declares `hands` and `inversion`
  mutually exclusive. Tapping pins `inversion` to root, and because deviations
  only accrete, a ladder that took first-inversion at rung 2 cannot switch
  tapping on at rung 3 without silently un-deviating an axis. Declaring the pair
  exclusive is how the model refuses that rather than papering over it.

`ELIGIBLE` is the surface on which a family's realizability constraints are
stated *once*. The existing `derive` hooks stay exactly where they are and keep
their present job; `ELIGIBLE` is the selector-facing statement of the same
coupling, consulted before a draw rather than after it.

## 5. The draw

`select()` keeps its shape: one slot per `[session] shape` entry, each drawn
against the within-session pushdown. What changes is what a slot fills.

1. **Identity.** Drawn with the existing per-axis recency weighting, unchanged.
2. **The legal menu.** The family's `ELIGIBLE` is evaluated against the drawn
   identity, yielding the deviation axes available on this ladder.
3. **Which deviations.** A `k`-subset of the menu, `k = rungs - 1`, drawn on a
   new pseudo-axis, `deviation`, **whose values are axis names**. This is what
   makes "I have not been made to play with accents in nine days" expressible in
   the machinery that already exists, and it is the single seam through which
   technique-weighted history arrives later ([§13](#13-deferred)).
4. **The route.** The chosen axes are sorted by tier; within a tier the order is
   drawn. That ordering is the ladder.
5. **Which values.** Each chosen deviation draws a non-plain value from its pool
   with the existing per-value weighting.
6. **The challenge rung.** One further deviation, drawn from
   `[challenge.<family>]` — a disjoint pool — and appended above the top
   ordinary rung. It may also *re-draw* an already-active deviation's value from
   the challenge pool, which is what lets the level++ rung be unfamiliar rather
   than merely longer.

Exclusions are enforced during step 3: once `hands` enters the subset,
`inversion` leaves the menu, and vice versa.

## 6. The validity gate

Today `selection._rejected` realizes one spec and checks it against
`max_notes` and `max_fret_span`; the measured single-spec valid rate is 0.278
for `scales`, which is why `MAX_ATTEMPTS` is 500. A ladder is valid only if
**every** rung is, so the composite rate is materially lower and a configuration
that is legal today could begin failing with an over-constrained error.

The gate is nevertheless kept simple: **resample the whole ladder** through the
existing `MAX_ATTEMPTS` budget. Per-deviation repair — keep the identity,
redraw the offending deviation's value, then its axis, and only then abandon the
ladder — is the obvious remedy and is deliberately **not built up front**.

Instead, a **measurement task runs first**, before the gate is finalized: sample
each family's ladders from a broad pool on `bass6` and realize every rung,
reporting the composite valid rate the way `MAX_ATTEMPTS`' own docstring
reports the single-spec rates it was derived from. If the measured rate supports
whole-ladder resampling at a defensible `MAX_ATTEMPTS`, the repair machinery is
never written. If it does not, repair is built *knowing why*, and the
measurement is the evidence in the docstring.

This ordering is the epic's answer to complexity accretion: the most intricate
component is made contingent on a number rather than on an intuition.

## 7. Configuration

```toml
[ladder]
chromatic = 4          # ordinary rungs, rung 1 (the plain shape) included
scales    = 4
arpeggios = 4
intervals = 4
challenge = true       # append the level++ rung → 5 engraved exercises per slot

[challenge.scales]
patterns            = ["numeric_1235", "fourths"]
accent_patterns     = ["every_5", "displaced"]
note_value_patterns = ["short_long"]
```

`[challenge.<family>]` is validated exactly as `[pool.<family>]` is — same axis
identifiers, same `vocabulary` check, same refusal to default an unwritten key.
It is a **disjoint** pool consulted only for the final rung. That disjointness
is the point: `vocabulary.AXES` already enumerates `every_5`, `displaced`,
`short_long`, `fourths`, `numeric_1235`, `numeric_1353`, `string_skip: 2` and
`inversion: third`, none of which any everyday pool draws. The challenge rung is
where that vocabulary becomes reachable, and keeping it out of the everyday pool
is what guarantees the last rung is unfamiliar.

`config_hash` covers both new sections, so editing a ladder or challenge pool
re-seeds the draw exactly as editing `[pool]` does today.

**Day length is untuned on purpose.** `count = 5` with four ordinary rungs and a
challenge rung is 25 engraved exercises. The player is explicitly not expected
to complete a sheet; the sheet is a spectrum to be surveyed. Reducing
`[session] shape` or varying rung counts per family are both one-line
configuration changes once there is playing experience to tune against.

## 8. The session record

`session.json` gains a rung dimension:

```json
"exercises": [
  { "family": "scales",
    "identity": { "root": 24, "scale_type": "ionian",
                  "traversal": "three_note_per_string" },
    "rungs": [ {},
               { "pattern": "groups_of_3" },
               { "pattern": "groups_of_3", "hands": 2 },
               { "pattern": "groups_of_3", "hands": 2,
                 "accent_pattern": "every_3" } ],
    "challenge": { "pattern": "numeric_1235", "hands": 2,
                   "accent_pattern": "every_3",
                   "note_value_pattern": "short_long" } }
]
```

Rungs are recorded resolved, never as a base plus a rule for recomputing them —
the same reasoning as decision #14. `replay` re-derives nothing; it reads back
what was drawn.

**History accounting.** A slot records its identity values, the deviation *axes*
drawn (on the `deviation` axis), and each deviation's *value*. Plain values are
never drawn, so they are never counted — which is what stops rung 1's mandatory
`straight` and `none` from poisoning the weighting for those values as
deviations.

**Clean reset, no back-compatibility.** There is no `version` key and no v1
reader. A pre-epic session directory fails loudly on read through the existing
`SessionError` path, which is correct: the project is experimental, the logs
live in gitignored `build/`, and `config_hash` changes regardless so tomorrow's
draw differs either way. Ten lines of back-compat to preserve an experimental
recency clock is complexity bought for nothing.

## 9. The sheet

Exercise numbering becomes two-level. `emit_book` names each exercise
`N. <title>` today; a ladder makes that `N.k`, with the rung title stating what
is active:

```
2.1  C Ionian — three-notes-per-string
2.2  C Ionian — three-notes-per-string, groups of 3
2.3  C Ionian — three-notes-per-string, groups of 3, tapped
2.4  C Ionian — three-notes-per-string, groups of 3, tapped, accent every 3
2.5  ⚡ C Ionian — three-notes-per-string, 1-2-3-5, tapped, accent every 5,
     short-long
```

The text is derivable from what exists: `vocabulary.display` owns the display
names and `cli.phrase` already renders them. A rung title is the identity phrase
followed by its active deviations **in the order they switched on**, so the page
is its own account of what is being added. The challenge rung is marked.

`--dry-run`, `show` and `replay` get the same two-level treatment, so a ladder
is inspectable before it is printed.

**Page breaks are not promised.** The emitter's only layout levers are
`\track { systemslayout … }` and `\section` markers; alphaTex as melete emits it
has no page directive, and `melete-render` exposes none. What is guaranteed is
that a ladder is one contiguous, numbered, section-titled group whose first rung
starts on a fresh system. Whether alphaTab can be induced to break a page at a
ladder boundary is scoped as a **spike**; if the answer is no, the two-level
numbering carries the grouping and nothing else changes.

## 10. Error handling

Consistent with §13 of the project spec: no silent degradation, every failure
names what could not be satisfied.

| Condition | Behaviour |
|---|---|
| Drawn identity leaves fewer legal deviations than `[ladder] <family>` requires | Reject and resample the ladder through `MAX_ATTEMPTS`, as an unrealizable spec is rejected today |
| A family's entire deviation menu is smaller than its configured rung count | Loud configuration error naming the family, its menu size and its rung count — no draw could ever satisfy it |
| Any rung fails `max_notes` / `max_fret_span` | Reject and resample the ladder; the reason counter aggregates by rung index so `_over_constrained` can say *which* rung was the obstacle |
| `[challenge.<family>]` names an axis the family does not read, or a value `vocabulary` does not know | Loud configuration error, identical treatment to `[pool.<family>]` |
| `[challenge]` requested but no `[challenge.<family>]` section for a family in the shape | Loud configuration error; never a silent fallback to the everyday pool, which would produce a challenge rung indistinguishable from rung 4 |
| A `session.json` without `identity`/`rungs` | `SessionError` naming the file, via the existing corrupt-record path |

The fifth row is the one that matters most in practice. A challenge rung that
silently degraded to an everyday draw would look exactly like a working feature
while delivering none of it — the §13 failure mode this project exists to avoid.

## 11. Testing strategy and acceptance

- **Ladder construction is unit-tested against a fixed RNG**: monotonicity (rung
  `k` deviations are a superset of rung `k-1`'s), tier ordering (no tier-3
  deviation precedes a tier-1 deviation), exclusion enforcement, and
  precondition pruning.
- **Materialization is round-tripped**: every rung of a drawn ladder produces an
  `ExerciseSpec` that `pipeline.realize` accepts, and rung 1's params equal
  identity-plus-plain exactly.
- **The `deviation` axis is tested like any other axis** against the existing
  recency-weighting tests: a deviation used today is drawn less often tomorrow.
- **Session round-trip**: write, read, `replay` — a replayed ladder is
  byte-identical to the recorded one.
- **Configuration**: each row of [§10](#10-error-handling) has a test asserting
  the message names the family or the key.
- **Goldens**: the existing per-family goldens are unaffected (a rung is an
  ordinary spec); a new book-level golden covers two-level numbering.
- **Acceptance** is a generated sheet, played. Every slot reads as a graded
  page; each rung adds exactly one nameable technique to the rung before it; the
  challenge rung is beyond the player's current reach. This is a human
  judgement, recorded as a validation task.

## 12. Recorded decisions

| # | Decision | Rationale |
|---|---|---|
| L1 | A ladder is a selection-time concept; each rung materializes to an ordinary `ExerciseSpec` | Keeps `pipeline`, `layout`, `rhythm`, `score`, families and per-exercise engraving untouched — the primary defence against complexity accretion |
| L2 | Deviation lattice with per-family constraints, not a difficulty-weighted budget | Assigning a difficulty integer to every axis value is a judgement that drifts and that nobody can validate; "which axes have left plain" is observable |
| L3 | `inversion` is a deviation, **provisionally** | Both readings are defensible; expected to resolve as more shape families land |
| L4 | Tapping is a deviation gated by family-declared preconditions | Keeps identity genuinely fixed while still allowing "the same scale, now tapped"; the family stays the authority on what can escalate into what |
| L5 | `hands` and `inversion` are mutually exclusive on an `arpeggios` ladder | Tapping pins `inversion` to root, and deviations only accrete |
| L6 | The challenge rung stacks one more deviation **and** may draw from a disjoint `[challenge.<family>]` pool | Stacking alone can only recombine the familiar; a widened pool alone may not raise difficulty enough |
| L7 | Deviation set is recency-weighted; order is drawn within declared tiers | Rotation without ever producing a rung harder than the one after it |
| L8 | Day length, rung counts and tier membership ship untuned | The purpose of this iteration is to see the whole spectrum before calibrating it |
| L9 | No back-compatibility for pre-epic `session.json` | Experimental project, gitignored logs, and `config_hash` re-seeds the draw regardless |
| L10 | The simple whole-ladder resample gate ships first; a measurement task establishes whether repair is needed | Makes the most intricate component contingent on a number rather than an intuition |

## 13. Deferred

- **Technique-weighted history.** The `deviation` pseudo-axis is built as its
  seam: weighting techniques by how recently they were *practised* rather than
  by how recently the axis was drawn becomes a different weight function on
  step 3 of [§5](#5-the-draw), not a new mechanism.
- **Per-deviation repair in the validity gate** — contingent on
  [§6](#6-the-validity-gate)'s measurement.
- **Difficulty calibration** — rung counts per family, tier membership, day
  length, and whether `[session] shape` should shrink.
- **Progressive tempo overload** — decision #20 keeps tempo out of the sampled
  axes; a ladder does not change that.
- **Open-string scale variants** — a different page, not a rung.
- **Page breaks at ladder boundaries** — spiked in this epic; built only if the
  spike says it is reachable.
- **One-hand triad arpeggios** —
  [`melete#227`](https://github.com/mnemosys-project/melete/issues/227). Once it
  lands, whether tapping is identity or a deviation on a triad ladder is a
  one-line change to the `arpeggios` `ELIGIBLE` declaration; this spec assumes
  neither behaviour.
