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

- A `tapping` module exposing a pure `Voice -> Voice` transform, given the
  profile and the drawn `hands` value — structurally parallel to
  `rhythm.restamp` and wired into `pipeline.realize` alongside it. It preserves
  the note count and order, changing only each note's `string`, `fret`, `hand`,
  and `attack`, so the single `replace` that rebuilds the `Score` stays
  `pipeline.realize`'s job. Pure — no I/O, no clock, no randomness.
- Two new orthogonal `Note` fields: `hand` (`LEFT | RIGHT`) and `attack`
  (`TAPPED | PLUCKED | SLURRED`), plus the generalization of `finger` from
  "left hand 1–4" to "finger 1–4 of `hand`". `PLUCKED` and `LEFT` are the
  defaults, so every existing family and test is unchanged.
- A hand-count-aware generalization of `_shared.boxed`, from "these pitches fit
  one hand" to "these pitches partition into one or two hands that each fit".
- A one-line change to `rhythm.restamp`'s accent pass so it never accents a
  `SLURRED` note (§9) — the only edit to the shared rhythm module.
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

`pipeline.realize` is the one place the renderer-agnostic stages are wired
together — it is what both `cli._score` and the selector's validity gate call.
Tapping is a new stage inserted into that composition:

```text
pipeline.realize:
  family.generate ─▶ layout.plan_voice ─▶ tapping ─▶ rhythm.restamp ─▶ replace ─▶ Score
   (which/where)      (fitter: meter,      (which hand,   (when)          (one place
                       subdivision,         where,                         rebuilds
                       tile into bars)      attack)                        the Score)
```

The family and the two existing modifiers are unchanged; tapping is a new
`Voice -> Voice` transform slotted between the fitter and the rhythm restamp,
and `realize` performs the single `replace` at the end for all of them.

**Ordering: after the fitter, before rhythm.** The **fitter**
(`layout.plan_voice`, §4) derives the meter and subdivision and may add or drop
notes (its levers) to tile the cycle into whole bars — so tapping runs *after*
it, articulating the sequence that is actually played rather than the one the
family first proposed. Tapping runs *before* `rhythm.restamp` because rhythm
stamps accents and a `SLURRED` note must not be accented (§9); running tapping
first is what lets the restamp see which notes are slurred. Tapping changes
`string`/`fret`/`hand`/`attack` and preserves note count and order, so it
neither disturbs the fitter's tiling nor invalidates the `LayoutHints` the
fitter consumed. Any stage may raise, and §9's validity gate resamples — the
existing contract.

**The load-bearing boundaries.**

- **The modifier boundary.** The `tapping` module is the only one that knows a
  note can be tapped or fretted by the right hand. Families stay
  tapping-ignorant; they emit `LEFT`/`PLUCKED` notes, and the modifier rewrites
  them when `hands == 2` (and is the identity when `hands == 1`). This is what
  lets the four families stay unchanged in substance.
- **The pitch/position seam.** Tapping consumes the family's (fitter-adjusted)
  *pitches* and discards their *positions*, exactly as rhythm discards the
  family's grouping. This is the seam that makes tapping a modifier rather than
  a family: the family need not know a tap-idiomatic layout exists, and the
  modifier need not know which arpeggio produced the pitches.
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
  different fingers on different hands; the emitter resolves this per `hand`. v1
  assigns no tapping fingerings — `arpeggios`/`scales` leave `finger=None` and
  the modifier adds none — so the per-hand meaning is defined and correctly
  emitted if ever set, but unexercised in v1.

The cross-product covers the whole corpus vocabulary: RH-tap `(RIGHT, TAPPED)`,
LH-tap `(LEFT, TAPPED)`, an ordinary note `(LEFT, PLUCKED)`, and a hammer/pull
target `(either, SLURRED)`. This mirrors the GPIF encoding the survey found,
where `Tapped`, `LeftHandTapped`, and `HopoOrigin`/`HopoDestination` are
independent note-level properties.

The `Note` invariant `pitch == tuning[string] + fret` is unchanged; tapping
moves `string`/`fret` but preserves pitch, so the invariant continues to hold
note-for-note.

## 5. The tapping modifier

The transform — `tapping.reach(voice, profile, hands) -> Voice` — is the
identity when the drawn `hands` value is `1`: it returns the voice unchanged, so a family that opted out or a draw that came up
single-handed pays nothing. When `hands == 2` it rebuilds the voice in four pure
steps and returns the new voice; `pipeline.realize` carries it into the `Score`
with its single `replace`, so `key`, `tempo_range`, `instrument`, and every
other field survive untouched. It reads `score.instrument` for the profile —
here, the `profile` argument `realize` already holds — and needs nothing the
pipeline does not already pass.

1. **Choose the tap-idiomatic layout.** Discard the family's fret positions and
   re-place the pitches under two hand anchors — the left hand lower on the neck,
   the right hand higher, with the reach gap between them that two hands exist to
   span — **on the string set the family already drew** (the distinct strings its
   notes occupy). Keeping the exercise on those strings, rather than choosing a
   new set, is the v1 rule; a narrow, tap-idiomatic shape is kept by configuring
   narrow `string_set` candidates in the tapping-eligible pools, not by a second
   string-selection policy here (§12). **On each string, the lower-fret notes are
   the left hand's and the higher-fret notes are the right's** (the corpus rule,
   §11 decision 6 — not a pitch-register split, which mis-assigns interleaved
   shapes). A pitch's placement, and therefore its hand, is fixed *once* and
   never changes between repetitions (the discipline `arpeggios` already uses to
   assign positions once). This layout is deliberately **not** the family's
   traditional one-hand fingering: a minor arpeggio tapped two-handed sits on the
   neck differently from the same arpeggio boxed under one hand, and that
   difference *is* the technique (§11, decision 11).
2. **Box each hand.** Each hand's assigned frets must satisfy that hand's
   `position_span`; the union may exceed one hand, which is the entire point.
   Where the shape spans multiple octaves it climbs the neck by repeating the
   two-hand pattern at a higher anchor — the hardest layout case, and the one v1
   solves only as far as the corpus data supports (§12). A shape that cannot be
   laid out under two hands **raises** rather than forcing a bad fingering, and
   §9's validity gate resamples.
3. **Articulate.** Stamp each note's `hand` and mark every note `TAPPED` — both
   hands tap, so in a two-hand shape no note is `PLUCKED` (§11, decision 12);
   that value stays the single-hand default. Step 4 then relaxes the notes that
   are actually slurred.
4. **Legato.** Walk each hand's notes in playing order. Within a run of
   consecutive notes on the *same string*, the first is `TAPPED` and the rest
   are `SLURRED` — a hammer-on where the fret ascends, a pull-off where it
   descends. Any string change forces a fresh `TAPPED`. This is derived from
   geometry, not a separate axis (§11, decision 7).

Because tapping runs before `rhythm.restamp`, the voice reaching the restamp
already carries `attack`, and its accent pass is taught to skip `SLURRED` notes
(§9): an accent marks an attack, and a slur has none. This is the only edit
tapping makes to a shared module beyond the `Note` fields (§4) and the
`_shared.boxed` generalization (§6).

## 6. Layout: one or two hands

Today `_shared.boxed(profile, pitches, strings, family, axes)` answers "do these
pitches fit one hand?" — it minimizes total fret travel and **raises when the
result spans more than `profile.position_span`** (issue #57). That refusal is
precisely the constraint two-handed tapping relaxes.

The generalization keeps `boxed` — the single-hand path — byte-for-byte, and
adds a **sibling in `_shared`**, `two_hand_boxed`, for the two-hand case. It
places the pitches on the string set under two anchors and partitions **by neck
region** — on each string the lower-fret notes to the left hand, the higher-fret
to the right — boxes each hand within one `position_span`, and returns each
note's placement with the hand it belongs to. This is a different objective from
`boxed`'s single-hand travel-minimization: it is splitting a reach, not
compacting one, and it deliberately yields a fingering the one-hand path never
would. Keeping it a sibling rather than a mode is what makes "single-hand
behavior unchanged" true by construction — `boxed` is not touched — and the
existing layout tests remain its guard (§10).

The multi-octave climb is where this is genuinely hard. A shape wider than two
boxes has to ascend the neck by repeating the two-hand pattern at successive
anchors, and choosing those anchors well is an open problem v1 does not fully
solve; it lays out what the corpus data covers and raises on the rest (§12),
never forcing a fingering it cannot justify.

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
`melete-render` (alphaTab). This is melete's **only** render path, so whether
the new articulations can travel it is a feasibility question, not a detail. The
emitter must translate `hand`/`attack` into note effects alongside the `acc` and
`lf` it already emits — a right-hand tap, a left-hand tap, and a hammer/pull
slur — and a right-hand-tapped note also needs *right-hand* fingering, where the
emitter uses `lf` (left-hand only) today. The spike resolves that too.

**Prerequisite spike — a go/no-go, run first.** alphaTab is a full Guitar Pro
renderer and the corpus files carry `Tapped`, `LeftHandTapped`, and hammer/pull
natively, so the *model* can represent taps; the open question is whether
melete's alphaTex *text* path exposes those effects. Settled in-container before
the emitter work, with three outcomes:

- **alphaTex expresses them** — proceed as planned; the emitter gains the note
  effects.
- **alphaTex cannot, but a lower-level channel can** — emit the articulations
  through it (the corpus proves alphaTab renders them natively). This is a
  **render-path change**, not an emitter tweak, and is re-scoped as such.
- **No workaround exists** — articulation is deferred and the epic re-scoped.
  The pure-Python layout work (§4–§7) still stands and is independently
  verifiable, but a tapped *sheet* waits.

The spike is the first task in the plan; it gates the emitter and the end-to-end
deliverable, not the pure-Python work, which is verifiable without a renderer.

## 9. Error Handling

| Failure | Behavior |
|---|---|
| A two-hand spec whose pitches cannot partition into two boxable hands | `tapping.reach` raises, naming the pitches and the profile; §9's validity gate resamples. Never clamped to fit. |
| A tapping weight configured for `intervals`/`chromatic` | Configuration error at load, naming the family and that it is not tapping-eligible. Never silently dropped. |
| `hands: 2` drawn but the family emitted fewer notes than two hands can split | Raise, naming the family and the note count. A one-note "chord" is not a two-hand exercise. |
| The alphaTex path cannot express an articulation (spike outcome) | The emitter fails loudly on the unrepresentable note rather than emitting a plausible wrong effect. The workaround, if any, is chosen at spike time. |
| A family emits a note already carrying `RIGHT`/`TAPPED` | Raise: families are tapping-unaware by contract, and a non-default articulation from one is a bug, not input. |
| A rhythm accent pattern would fall on a `SLURRED` note | `restamp`'s accent pass consults `attack` and never accents a slur — a hammer-on/pull-off has no attack to accent. This is the one change tapping makes to the shared `rhythm` module. |

No swallowed exceptions and no layout clamped to fit: a mislabelled sheet is
worse than a resampled one, because the label is the part a student trusts.

## 10. Testing Strategy

| Component | Approach |
|---|---|
| `Note` defaults | Unit: a default-constructed note is `(LEFT, PLUCKED)`; existing family tests unchanged. |
| `_shared.boxed` single-hand | Regression: identical output to pre-change for every existing input (the central guard). |
| `_shared.boxed` two-hand | Property: each returned hand's span ≤ `position_span`; every note on a string that sounds its pitch; partition assigns each pitch one hand. |
| `tapping.reach` purity | Property: same voice + profile + hands ⇒ same voice; `hands: 1` is identity; the pitch multiset is preserved. |
| Partition consistency | A recurring pitch is assigned the same hand on every occurrence. |
| Legato derivation | Golden: same-string ascending run ⇒ one `TAPPED` then `SLURRED` hammers; descending ⇒ pull-offs; string change ⇒ fresh `TAPPED`. |
| Emitter | Golden alphaTex for one tapped arpeggio and one tapped scale, byte-compared. |
| End to end | One integration test: a `hands: 2` arpeggio config renders to a `.gp` a human validates once. |

**Central invariant.** `pitch == tuning[string] + fret` holds for every note the
modifier emits — tapping moves position but never changes pitch. The property
test on `tapping.reach` that asserts the multiset of pitches is identical before
and after is the one that proves the hardest claim in the spec: that a re-layout
across two hands is still the exercise the family drew.

## 11. Recorded Decisions

| # | Decision | Rationale |
|---|---|---|
| 1 | Tapping is a cross-cutting modifier, not a fifth family. | The corpus is existing shapes with a hand overlay; a family would re-implement arpeggio/scale geometry, and articulation still needs a `Note` change regardless. Mirrors the rhythm modifier precedent exactly. |
| 2 | The hand count is bounded at two, not modelled as N. | One or two hands is the entire musical space; "N hands" is speculative generality with no case behind it. `Hand` is a two-valued enum on purpose. |
| 3 | The modifier re-lays-out (discards the family's positions), rather than only relabelling. | The signature two-hand shapes are tap-idiomatic layouts the family would never choose; relabelling alone cannot produce them. Discarding positions mirrors rhythm discarding grouping. |
| 4 | `hand` and `attack` are two orthogonal fields, not one flat articulation enum. | Hand and attack vary independently — either hand taps, a slur occurs under either hand — and this matches GPIF's independent note properties. |
| 5 | The two-hand partition lives in `_shared` beside `boxed` — a sibling `two_hand_boxed` — not in the families. | `_shared` is the one place that already owns `position_span`; keeping the partition there lets all four families stay hand-unaware and is the seam a future family-supplied partition plugs into. A sibling rather than a mode of `boxed` keeps the single-hand path untouched by construction. |
| 6 | Notes split to hands by **per-string fret region** (low frets → LEFT, high frets → RIGHT), derived, not a sampled axis. | Verified against the corpus note data: the exercises interleave pitch across the hands, so a pitch-register split mis-assigns them; the fret-region rule reproduces every surveyed shape and unifies arpeggios and scales under one choreography. The resulting split (2+2, 2+1, …) falls out of boxability. |
| 7 | Legato is derived from same-string adjacency, not a sampled axis. | Where a hand plays consecutive notes on one string, hammer/pull is the only idiomatic attack; deriving it keeps v1 axis-free while remaining faithful. |
| 8 | Tapping is selected by a per-family, config-controlled `hands` axis that is opt-in by default (candidates default to `(1,)`), not a freely/uniformly sampled one. | The requirement is controllability — a deliberately tapping-heavy diet — which a uniformly sampled axis cannot guarantee. It rides the ordinary axis machinery, so it is drawn per eligible family, recorded in the session log, and counted by §9; config fully governs its candidates, and only the eligible families declare it. Recorded as a stopgap for the deferred specification mechanism. |
| 9 | `intervals` and `chromatic` are ineligible; a tapping weight on them errors. | Nothing in the corpus taps them, and chromatic's subject *is* its left-hand fingering. Silent ineligibility would hide a config mistake. |
| 10 | The renderer spike is a go/no-go run first; it gates the emitter and the end-to-end sheet, not the pure-Python work. | §4–§7 are verifiable without a renderer, but the tapped *sheet* depends on the spike: if alphaTex cannot express the articulations and no lower-level channel substitutes, articulation is deferred and the epic re-scoped. Framing it as an emitter detail would hide a possible show-stopper until emit time. |
| 11 | The two-hand layout deliberately departs from the family's one-hand fingering, and the multi-octave climb is only partially solved in v1. | A tapped arpeggio sits on the neck differently from the one-hand shape — that difference is the technique, not a defect — and choosing anchors for a multi-octave ascent is an open layout problem. v1 lays out what the corpus covers and raises (resamples) beyond it rather than forcing a fingering it cannot justify. |
| 12 | In a two-hand shape every note is `TAPPED` or `SLURRED`, never `PLUCKED`. | Both hands are on the fretboard, so nothing plucks; the corpus notates the left hand inconsistently, but a tapping exercise is honest only if the low voice taps. `PLUCKED` stays purely the single-hand default. |

## 12. Deferred

Named so the boundary is explicit and the design leaves room for each.

- **Next version.** Single-string three-finger + string-jump stretches;
  pedal-tone taps and tap-and-pull-to-open cascades (survey categories C, D). A
  family supplying its own hand-partition (a `two_hand` traversal or a
  `LayoutHints` field), which the partition-as-data seam in §6 is designed to
  accept additively.
- **Next version.** A general solution to the multi-octave two-hand climb —
  choosing anchors as the shape ascends the neck across many octaves. v1 handles
  the octave range the corpus covers and raises beyond it (§6, §11 decision 11).
- **Next version.** A demand-driven exercise-specification mechanism that lets
  the user request "arpeggios, tapping" as a first-class exercise, superseding
  the §7 config weight. This is the author's intended evolution of how melete is
  told what to generate.
- **Never (as tapping).** Fretting-hand counts above two.

---

**Status:** Draft. Filed as epic
[#67](https://github.com/mnemosys-project/.github/issues/67).
