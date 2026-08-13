# Two-handed tapping — an articulation modifier for melete

**Design specification, v1.0**
**Date:** 2026-08-13
**Org:** `mnemosys-project`
**Repository:** `mnemosys-project/melete`
**Epic:** [`mnemosys-project/.github#67`](https://github.com/mnemosys-project/.github/issues/67)

## Table of Contents

- [1. Overview](#1-overview)
- [2. Scope](#2-scope)
- [3. Architecture](#3-architecture)
- [4. Data model](#4-data-model)
- [5. The tapping modifier](#5-the-tapping-modifier)
- [6. Layout: one or two hands](#6-layout-one-or-two-hands)
- [7. Selection and configuration](#7-selection-and-configuration)
- [8. Rendering](#8-rendering)
- [9. Error Handling](#9-error-handling)
- [10. Testing Strategy](#10-testing-strategy)
- [11. Recorded Decisions](#11-recorded-decisions)
- [12. Deferred](#12-deferred)

## 1. Overview

Two-handed tapping is the largest category of the author's course material that
melete cannot generate. This epic adds it as a **cross-cutting articulation
modifier** — one `Score -> Score` function over the families that opt in —
parallel in every structural respect to the rhythm modifier (§8 of the melete
spec). A family decides *which* notes and, normally, *where* on the neck; the
rhythm modifier decides *when*; the tapping modifier decides *which hand frets
each note, where each hand sits, and how each note is attacked*.

**Success criterion.** With a per-family tapping weight configured, `melete
generate` produces a practice sheet whose arpeggio or scale exercise is a
correct two-handed tapped realization — the pitches the family drew, split
across two fretting hands, engraved with tap, left-hand-tap, and hammer/pull
articulations a player can read and play — and the same session log replays to
the same sheet. Held against a sheet, a reader can say yes or no.

**What it inherits, and what it does not.** It inherits the whole generative
model: the families, their axes (`quality`, `inversion`, `direction`, the
`pattern` window-slide, string traversal), the selector, the session log, and
the alphaTab rendering path. It does **not** inherit the assumption those were
built on — that exactly one hand touches the fretboard. That assumption is
generalized here from one hand to one *or two* hands, and no further: a fretting
hand count above two is not a musical case and is explicitly out of scope (§11,
decision 2).

This design is seeded by an R&D survey of 87 historical Guitar Pro files. The
finding that shaped it: the tapping material is overwhelmingly *existing shapes
with a hand-assignment overlay* — two-hand tapped arpeggios and scales put
through the same axes melete already samples — so the honest architecture reuses
the families and adds one orthogonal dimension, articulation, rather than
re-implementing arpeggio and scale geometry inside a new family.

## 2. Scope

### In scope

- A `tapping` module exposing `apply(score, params) -> Score`, structurally
  parallel to `rhythm.apply`: it reads its own axes from the shared parameter
  mapping, rebuilds the `Score` with `dataclasses.replace` so untouched fields
  (notably `key`) survive, and is pure — no I/O, no clock, no randomness.
- Two new orthogonal `Note` fields: `hand` (`LEFT | RIGHT`) and `attack`
  (`TAPPED | PLUCKED | SLURRED`), plus the generalization of `finger` from
  "left hand 1–4" to "finger 1–4 of `hand`". `PLUCKED` and `LEFT` are the
  defaults, so every existing family and test is unchanged.
- A hand-count-aware generalization of `_shared.boxed`, from "these pitches fit
  one hand" to "these pitches partition into one or two hands that each fit".
- Tapping realization for the **`arpeggios`** and **`scales`** families: the
  register-split two-hand shapes (categories A and B of the survey).
- Legato: within a hand's run of consecutive same-string notes, the first note
  is `TAPPED` and the rest are `SLURRED` (hammer ascending, pull descending).
- Selection through a **per-family tapping weight** in `config.toml`, spanning
  never→always, surfaced to the family as a `hands` axis (`1 | 2`) recorded in
  the session log for reproducibility.
- A prerequisite **rendering spike** (§8): confirm the alphaTex path can express
  RH-tap, LH-tap, and hammer/pull, or establish the workaround.

### Non-goals

- **Three or more fretting hands.** Not a musical case; the model is bounded at
  two on purpose (§11, decision 2). "N hands" would be speculative generality.
- **Single-string three-finger + string-jump stretches**, and other bespoke
  tapping shapes — pedal-tone taps and tap-and-pull-to-open cascades (survey
  categories C and D). Deferred (§12), not foreclosed: the data model and the
  partition-as-data seam are chosen so these are additive later.
- **Families supplying their own hand-partition.** In v1 the modifier owns the
  partition. Letting a family propose a partition (a `two_hand` traversal, or a
  `LayoutHints` field) is the deferred next stage; the partition is represented
  as data specifically so that stage is additive (§12).
- **A demand-driven exercise-specification mechanism.** The v1 config weight is
  a stopgap. A richer way to request "arpeggios, tapping" as a first-class
  exercise type is the intended successor and is out of scope here (§12).
- **`intervals` and `chromatic` tapping.** These families emit `hands: 1` only.
  Nothing in the corpus drills them two-handed, and chromatic's exercise *is*
  its left-hand fingering — tapping would contradict its subject.

### Audience

melete is a tool for its author, an advanced player whose instructor sets the
material. That permits assumptions a tool for strangers could not make: the
tapping vocabulary can be drawn directly from this player's course corpus, and
the exercises can assume a player who already taps.

## 3. Architecture

The tapping modifier is a peer of the rhythm modifier in the §4 pipeline. The
selector draws a family's parameters, the family generates a `Score`, and then
the cross-cutting modifiers run:

```text
selection ─▶ family.generate ─▶ tapping.apply ─▶ rhythm.apply ─▶ Score
              (which/where)      (which hand,       (when)
                                  where, attack)
```

**Ordering: tapping before rhythm.** Tapping decides positions and articulation
from pitch and profile; rhythm restamps durations and accents and carries
everything else through untouched. Tapping must run first because its legato
derivation reads note *adjacency on a string*, which rhythm does not disturb,
while tapping changes `string`/`fret`, which rhythm must not predate. Both can
reject a draw, and §9's validity gate resamples — the existing contract.

**The load-bearing boundaries.**

- **The modifier boundary.** `tapping.apply` is the only module that knows a
  note can be tapped or fretted by the right hand. Families remain
  tapping-ignorant; they emit `hands: 1`-shaped Scores with `LEFT`/`PLUCKED`
  notes, and the modifier rewrites them when `hands == 2`. This is what lets the
  four families stay unchanged in substance.
- **The pitch/position seam.** Tapping consumes the family's *pitches* and
  discards its *positions*, exactly as rhythm discards the family's grouping.
  This is the seam that makes tapping a modifier rather than a family: the
  family need not know a tap-idiomatic layout exists, and the modifier need not
  know which arpeggio produced the pitches.
- **`_shared.boxed` as the one hand-aware layout primitive.** The generalization
  from one hand to two lives in exactly one function, shared with the families'
  single-hand path (§6). Nothing else counts hands.

## 4. Data model

`Note` (melete `score.py`) gains two fields, orthogonal because hand and attack
genuinely vary independently — either hand can tap, and a slur can occur under
either hand:

```python
class Hand(Enum):     # which hand frets the note
    LEFT
    RIGHT

class Attack(Enum):   # how the note is sounded
    TAPPED    # attacked by tapping the fret (either hand)
    PLUCKED   # ordinary picked/plucked note — the default
    SLURRED   # sounded by hammer-on/pull-off; no fresh attack
```

- `hand: Hand = Hand.LEFT` — the fretting hand. Single-hand families always
  emit `LEFT`.
- `attack: Attack = Attack.PLUCKED` — the articulation. The default preserves
  every existing family and golden file unchanged.
- `finger: int | None` keeps its type but its meaning generalizes to "finger
  1–4 of `hand`". A left-hand `finger=1` and a right-hand `finger=1` are
  different fingers on different hands; the emitter resolves this per `hand`.

The cross-product covers the whole corpus vocabulary: RH-tap `(RIGHT, TAPPED)`,
LH-tap `(LEFT, TAPPED)`, an ordinary note `(LEFT, PLUCKED)`, and a hammer/pull
target `(either, SLURRED)`. This mirrors the GPIF encoding the survey found,
where `Tapped`, `LeftHandTapped`, and `HopoOrigin`/`HopoDestination` are
independent note-level properties.

The `Note` invariant `pitch == tuning[string] + fret` is unchanged; tapping
moves `string`/`fret` but preserves pitch, so the invariant continues to hold
note-for-note.

## 5. The tapping modifier

`apply(score, params) -> Score` is a no-op when the drawn `hands` axis is `1`:
it returns the score unchanged, so a family that opted out or a draw that came
up single-handed pays nothing. When `hands == 2` it rebuilds the voice in four
pure steps, then returns `replace(score, voice=…)` so `key`, `tempo_range`,
`instrument`, and every other field survive (the rhythm module's `replace`
discipline, and for the same reason — a hand-built `Score` here would silently
drop the key and mis-spell every note).

1. **Partition by register.** Collect the distinct pitches of the exercise and
   assign each to a hand — the lower band to `LEFT`, the upper band to `RIGHT`
   — *once*, before the sequence is walked, so a pitch that recurs never
   switches hands between repetitions (the discipline `arpeggios` already uses
   to assign positions once). The split point is chosen so each band is boxable
   under one hand on the chosen string set (§6); it is derived, not sampled
   (§11, decision 6).
2. **Box each hand.** Lay each band onto a narrow, tap-idiomatic string set —
   the corpus stacks the shape on roughly two strings — with `LEFT` low and
   `RIGHT` high and the fret gap between them that two-hand reach is *for*. Each
   hand's frets must satisfy that hand's `position_span`; the union may exceed
   one hand, which is the entire point.
3. **Articulate.** Stamp each note's `hand`; the attacked notes become `TAPPED`.
4. **Legato.** Walk each hand's notes in playing order. Within a run of
   consecutive notes on the *same string*, the first is `TAPPED` and the rest
   are `SLURRED` — a hammer-on where the fret ascends, a pull-off where it
   descends. Any string change forces a fresh `TAPPED`. This is derived from
   geometry, not a separate axis (§11, decision 7).

The modifier reads `score.instrument` for the profile it needs to place notes,
so it requires nothing the `Score` does not already carry.

## 6. Layout: one or two hands

Today `_shared.boxed(profile, pitches, strings, family, axes)` answers "do these
pitches fit one hand?" — it minimizes total fret travel and **raises when the
result spans more than `profile.position_span`** (issue #57). That refusal is
precisely the constraint two-handed tapping relaxes.

The generalization keeps the single-hand behavior byte-for-byte and adds a
two-hand mode selected by the caller. In two-hand mode the function partitions
the pitches into a low group and a high group, boxes each within one
`position_span`, and returns the placement together with the hand each note
belongs to. The single-hand call is the degenerate case — one group, one hand —
and must produce identical output to today's function for identical input; the
existing layout tests are the guard on that claim (§10).

Placing the partition here, in the one primitive that already owns
`position_span`, is deliberate: it is the seam a future family-supplied
partition (§12) plugs into, and keeping it out of the families is what lets the
four of them stay hand-unaware.

## 7. Selection and configuration

Tapping is not a vocabulary axis the selector samples freely; it is a per-family
weight the user controls, because the requirement is *controllability* — the
author wants to dial a tapping-heavy practice diet, not receive the occasional
surprise tap.

- **Config.** Each tapping-eligible family's pool section
  (`[pool.arpeggios]`, `[pool.scales]`) gains a tapping weight spanning
  never→always. The default is off, so existing configurations are unchanged.
- **The `hands` axis.** The selector resolves the weight into a `hands` value
  (`1 | 2`) drawn per exercise and written into the parameter mapping alongside
  the family and rhythm axes. It travels the same dictionary, is recorded in the
  session log, and replays deterministically. §9's coverage accounting counts it
  as an axis where the family is tapping-eligible.
- **Eligibility.** `arpeggios` and `scales` are eligible. `intervals` and
  `chromatic` always emit `hands: 1`; configuring a tapping weight for them is a
  configuration error, reported, not silently ignored (§9).

This is a deliberate stopgap. The successor — a demand-driven way to request
"arpeggios, tapping" as a first-class exercise type — is deferred (§12); when it
lands it supersedes this weight rather than extending it.

## 8. Rendering

melete engraves by emitting alphaTex and rendering with the vendored
`melete-render` (alphaTab). The emitter must translate the new articulations
into alphaTex note effects — the same `{…}` block that already carries `acc`
and `lf`.

**Prerequisite spike.** alphaTab is a full Guitar Pro renderer and GPIF carries
`Tapped`, `LeftHandTapped`, and hammer/pull natively, so the model can represent
taps; the open question is whether melete's alphaTex *text* path exposes those
effects or needs a workaround. This is settled in-container before the emitter
work, and its outcome may adjust the emitter design (only). The spike is the
first task in the plan; it does not block the pure-Python work (§4–§7), which is
verifiable without a renderer.

## 9. Error Handling

| Failure | Behavior |
|---|---|
| A two-hand spec whose pitches cannot partition into two boxable hands | `tapping.apply` raises, naming the pitches and the profile; §9's validity gate resamples. Never clamped to fit. |
| A tapping weight configured for `intervals`/`chromatic` | Configuration error at load, naming the family and that it is not tapping-eligible. Never silently dropped. |
| `hands: 2` drawn but the family emitted fewer notes than two hands can split | Raise, naming the family and the note count. A one-note "chord" is not a two-hand exercise. |
| The alphaTex path cannot express an articulation (spike outcome) | The emitter fails loudly on the unrepresentable note rather than emitting a plausible wrong effect. The workaround, if any, is chosen at spike time. |
| A family emits a note already carrying `RIGHT`/`TAPPED` | Raise: families are tapping-unaware by contract, and a non-default articulation from one is a bug, not input. |

No swallowed exceptions and no layout clamped to fit: a mislabelled sheet is
worse than a resampled one, because the label is the part a student trusts.

## 10. Testing Strategy

| Component | Approach |
|---|---|
| `Note` defaults | Unit: a default-constructed note is `(LEFT, PLUCKED)`; existing family tests unchanged. |
| `_shared.boxed` single-hand | Regression: identical output to pre-change for every existing input (the central guard). |
| `_shared.boxed` two-hand | Property: each returned hand's span ≤ `position_span`; every note on a string that sounds its pitch; partition assigns each pitch one hand. |
| `tapping.apply` purity | Property: same score + params ⇒ same score; `hands: 1` is identity; `key` survives `replace`. |
| Partition consistency | A recurring pitch is assigned the same hand on every occurrence. |
| Legato derivation | Golden: same-string ascending run ⇒ one `TAPPED` then `SLURRED` hammers; descending ⇒ pull-offs; string change ⇒ fresh `TAPPED`. |
| Emitter | Golden alphaTex for one tapped arpeggio and one tapped scale, byte-compared. |
| End to end | One integration test: a `hands: 2` arpeggio config renders to a `.gp` a human validates once. |

**Central invariant.** `pitch == tuning[string] + fret` holds for every note the
modifier emits — tapping moves position but never changes pitch. The property
test on `tapping.apply` that asserts the multiset of pitches is identical before
and after is the one that proves the hardest claim in the spec: that a re-layout
across two hands is still the exercise the family drew.

## 11. Recorded Decisions

| # | Decision | Rationale |
|---|---|---|
| 1 | Tapping is a cross-cutting modifier, not a fifth family. | The corpus is existing shapes with a hand overlay; a family would re-implement arpeggio/scale geometry, and articulation still needs a `Note` change regardless. Mirrors the rhythm modifier precedent exactly. |
| 2 | The hand count is bounded at two, not modelled as N. | One or two hands is the entire musical space; "N hands" is speculative generality with no case behind it. `Hand` is a two-valued enum on purpose. |
| 3 | The modifier re-lays-out (discards the family's positions), rather than only relabelling. | The signature two-hand shapes are tap-idiomatic layouts the family would never choose; relabelling alone cannot produce them. Discarding positions mirrors rhythm discarding grouping. |
| 4 | `hand` and `attack` are two orthogonal fields, not one flat articulation enum. | Hand and attack vary independently — either hand taps, a slur occurs under either hand — and this matches GPIF's independent note properties. |
| 5 | The partition lives in `_shared.boxed`, not in the families. | It is the one primitive that already owns `position_span`, and keeping it there lets all four families stay hand-unaware and is the seam a future family-supplied partition plugs into. |
| 6 | The split ratio is derived from register and playability, not a sampled axis. | The corpus split (2+2, 2+1, …) falls out of boxability; an axis would multiply the variant space for no observed musical gain. Revisit if variety proves thin. |
| 7 | Legato is derived from same-string adjacency, not a sampled axis. | Where a hand plays consecutive notes on one string, hammer/pull is the only idiomatic attack; deriving it keeps v1 axis-free while remaining faithful. |
| 8 | Tapping is selected by a per-family config weight, not a free vocabulary axis. | The requirement is controllability — a deliberately tapping-heavy diet — which a low-probability free axis cannot guarantee. Recorded as a stopgap for the deferred specification mechanism. |
| 9 | `intervals` and `chromatic` are ineligible; a tapping weight on them errors. | Nothing in the corpus taps them, and chromatic's subject *is* its left-hand fingering. Silent ineligibility would hide a config mistake. |
| 10 | The renderer spike gates the emitter, not the pure-Python work. | §4–§7 are verifiable without a renderer; only the emitter depends on the spike outcome, so the spike need not block the bulk of the epic. |

## 12. Deferred

Named so the boundary is explicit and the design leaves room for each.

- **Next version.** Single-string three-finger + string-jump stretches;
  pedal-tone taps and tap-and-pull-to-open cascades (survey categories C, D). A
  family supplying its own hand-partition (a `two_hand` traversal or a
  `LayoutHints` field), which the partition-as-data seam in §6 is designed to
  accept additively.
- **Next version.** A demand-driven exercise-specification mechanism that lets
  the user request "arpeggios, tapping" as a first-class exercise, superseding
  the §7 config weight. This is the author's intended evolution of how melete is
  told what to generate.
- **Never (as tapping).** Fretting-hand counts above two.

---

**Status:** Draft. Filed as epic
[#67](https://github.com/mnemosys-project/.github/issues/67).
