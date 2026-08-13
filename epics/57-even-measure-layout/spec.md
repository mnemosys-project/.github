# Melete — Even-measure exercise layout

**Epic:** [`mnemosys-project/.github#57`](https://github.com/mnemosys-project/.github/issues/57).
**Predecessor:** epic #46 (`epics/46-alphatab-output/`), the alphaTab/`.gp`
renderer port whose first practical output — the Aug 12 2026 practice session,
five exercises — is the concrete input this epic iterates on.
**Date:** 2026-08-13.

## Table of Contents

- [1. Overview](#1-overview)
- [2. Scope and non-goals](#2-scope-and-non-goals)
- [3. The core shift: derive, don't sample](#3-the-core-shift-derive-dont-sample)
- [4. The layout fitter](#4-the-layout-fitter)
- [5. Repeat barlines](#5-repeat-barlines)
- [6. Family rules (6-string bass)](#6-family-rules-6-string-bass)
- [7. Pipeline integration](#7-pipeline-integration)
- [8. Worked examples: the five exercises](#8-worked-examples-the-five-exercises)
- [9. Output-directory cleanup and the build/ convention](#9-output-directory-cleanup-and-the-build-convention)
- [10. Testing strategy](#10-testing-strategy)
- [11. Task breakdown and bookends](#11-task-breakdown-and-bookends)
- [12. Risks, assumptions, open questions](#12-risks-assumptions-open-questions)

## 1. Overview

Epic #46 produced melete's first loadable Guitar Pro output: five exercises for
the Aug 12 2026 practice session. The **musical content is good**; the
**engraving is not**. Every one of the five ends in a **partial measure**, which
Guitar Pro renders with red staff lines — visually jarring and hard to read. The
cause is structural: the time signature and note subdivision are drawn at random
from config pools (`[pool.rhythm]`), and the barring pass (`score.py::bar`) fills
measures until the notes run out, leaving whatever fraction remains as an
un-padded stub.

The five exercises as generated (source `.atex` from the Aug 12 run):

| # | Exercise | Meter / subdivision now | Result |
|---|----------|-------------------------|--------|
| 01 | Chromatic 1-2-4-3, adjacent strings, up and down | 4/4, mixed 8th/16th triplets | ~2⅓ bars, partial |
| 02 | A♭ Ionian, positional, ascending (1 octave) | 3/4, eighths | 1⅓ bars, partial |
| 03 | A Phrygian, positional, descending groups of 3 | 3/4, dotted-16th + 32nd | ~1½ bars, partial |
| 04 | D♭ m7, first inversion, up and down broken | 4/4, triplet-eighths | ~2¼ bars, partial |
| 05 | C chromatic fourths, ascending pairs, descending | 3/4, sixteenths | ~2⅙ bars, partial |

**Goal.** Every generated exercise engraves as **a whole number of complete
measures — never a partial/red bar — wrapped in repeat barlines**, spanning the
correct number of octaves, with a **simple, low-denominator time signature
derived from the pattern** rather than sampled. An even bar count is a strong
preference (symmetric two-line layout), not a hard rule.

**Framing.** For these exercises the time signature carries little musical
meaning — the student follows a tempo and drills a pattern; nobody is counting a
downbeat. The meter's job here is **legibility**: fit the pattern onto the page
symmetrically and readably. That is what lets the fitter treat meter, subdivision,
and (as a last resort) note count as free variables to be *chosen* for a clean
engraving.

## 2. Scope and non-goals

**In scope.**

1. **Repeat barlines** — greenfield support in the `Score` model and the
   alphaTex emitter. Every exercise is wrapped in an open/close repeat.
2. **The layout fitter** — a new pass that derives (subdivision, time signature,
   bar count), guaranteeing complete measures via a documented priority ladder
   and a small set of note-count levers, and recording a **legibility trace** of
   why it chose a given meter.
3. **Full-cycle semantics** — up/down turnaround (default) with the apex-repeat
   lever; chromatic patterns use **all strings**; scales span **two octaves from
   the lowest string**; all targeted at the **6-string bass (bass6)**.
4. **Output-directory cleanup** — stop writing generated artifacts into sibling
   git repositories (the `sample-gp/` mistake); route generated output to a
   gitignored `build/` directory; assert the convention in the repo `MEMORY.md`.
5. **Regenerate the five exercises** as the concrete acceptance check.

**Non-goals (follow-on).**

- **Two-octave strategies for 4- and 5-string basses.** Fewer strings cannot
  always reach two octaves in one position; the strategy is likely family- and
  fingering-specific and is deferred to its own improvement. This epic makes
  bass6 correct first.
- **Musically meaningful meters.** We are not modelling phrasing, accents on
  downbeats, or performance time signatures. Meter here is a layout device.
- **New exercise families or new axes** beyond what the rules above require.

## 3. The core shift: derive, don't sample

Today: `family.generate` emits flat quarter notes at a family-default meter →
`rhythm.apply` **samples** a `time_signature` and a `subdivision` from
`[pool.rhythm]` and restamps durations → `bar()` splits into measures and leaves
the tail partial. The 3/4-with-eighths landing in exercise 02 is not a bug in one
place; it is the system working as designed — a random draw with no feedback from
how the pattern actually tiles.

This epic replaces the **sampling** of meter and subdivision with **derivation**.
A new **layout fitter** computes them from the pattern so the engraving is
legible by construction. Sampling remains for genuinely musical, student-facing
choices (which pattern, which root, tempo); it is removed for the two variables
that only ever served legibility.

**This is a tunable heuristic engine, not a closed formula.** The priority ladder
and the per-family declarations (how to build the cycle, what the natural
grouping is, which note-count levers are legal) are **explicit and extensible**.
We expect to discover corner cases and add rules over time; the design must make
that cheap and keep each decision inspectable (hence the legibility trace).

## 4. The layout fitter

### 4.1 Model

A pattern is a sequence of notes. The fitter works in two quantities:

- **Pulse** — the base written note value shared by the pattern's notes, possibly
  carrying a tuplet ratio (e.g. an eighth-note triplet: written 1/8, ratio 3:2).
- **Group** — the pattern's natural building block: a run of `g` pulses that
  forms **one beat**. `g` comes from the pattern's *cell* (see 4.3).

With one group per beat, the meter denominator is naturally the **quarter note**
(a beat = a quarter), and the group size selects the subdivision inside the beat:

| group size `g` | subdivision (one beat) |
|---|---|
| 2 | two eighths |
| 3 | eighth-note triplet |
| 4 | four sixteenths |
| 6 | sixteenth sextuplet |

This is why the denominator stays low (`/4`) in the common case: the *subdivision*
absorbs the density, not the denominator. A `/8` or `/16` denominator only appears
when the pulse count cannot be grouped into whole beats — which is exactly the
situation the note-count levers exist to repair (the 15/16 → 16/16 → 4/4 move).

### 4.2 Step 1 — Build the full cycle (fixes the note sequence)

Per-family structural rules produce the note sequence and a declared set of legal
note-count adjustments:

- **Turnaround.** `up_down` (the default; see §6) ascends then descends. The
  **apex note/cell may be played once or twice** — this is a lever the fitter is
  allowed to toggle to reach an even, cleanly-tiling count.
- **Octave / string span.** Chromatic uses all strings; scales span two octaves
  from the lowest string (§6).
- The family declares which **note-count levers** are legal for this pattern
  (apex repeat/omit; add or drop one note at an end), so the fitter searches only
  musically acceptable variants.

**The family→fitter contract: layout hints.** The cell size and the legal levers
depend on the *chosen axis values* (a family offering both `straight` and a
`groups_of_4` window has no single static cell), so they cannot live as constants
on the `Family` dataclass and must not be re-guessed by inspecting the voice.
`generate()` therefore emits a structured **`LayoutHints`** value alongside the
`Score`:

- **cell length** `g` (the natural group; §4.3),
- **legal note-count levers** for this pattern (which of apex repeat/omit,
  add/drop-one are musically acceptable here),
- **seam index** — the note offset of the musical turnaround, so the ladder can
  align a bar boundary to it (§4.5).

The fitter is then a **pure function of `(voice, LayoutHints)`**, which is also
its test seam. This is a small change to the four families' return type
(`generate → (Score, LayoutHints)`), carried through the pipeline.

### 4.3 Step 2 — Cell-aware pulse selection

The natural grouping `g` is the pattern's **cell length**, not simply notes per
string:

- **Straight run** (e.g. chromatic 1-2-4-3 across strings): the cell is the
  per-string finger group → `g = 4`.
- **Sequenced/windowed run** (e.g. a 3-notes-per-string scale played in groups of
  four: 1-2-3-4, 2-3-4-5, …): the cell is the **melodic window** → `g = 4`, even
  though there are 3 notes per string. The fitter reads `g` from the family's
  pattern declaration, not the fretboard.
- **Uniform-pulse fallback.** When a traversal yields *unequal* groups (e.g. a
  positional two-octave scale with a mix of 2- and 3-note strings), there is no
  single `g`. The fitter falls back to a **uniform pulse** (typically eighths)
  and tiles by total note count, leaning harder on the note-count levers.

### 4.4 Step 3 — Candidate generation

The fitter enumerates candidates over the cross-product of:

- **note-count variant** (from the legal levers in 4.2),
- **pulse / group size** (from 4.3; usually one natural choice, sometimes two),
- **meter** `(beats b, denominator d)` and **bar count** `M`

subject to the hard constraint that the pattern tiles **exactly**: total time
`= M × (b/d)` with no remainder. In beat-centric terms: with `B = N/g` whole
beats, every `(b, M)` with `b·M = B` is a candidate; `d = 4` unless a
compound/`/8` feel is explicitly chosen.

### 4.5 Step 4 — The priority ladder (objective function)

Candidates are ranked lexicographically:

1. **No partial bar.** Hard filter. (Non-tiling candidates are discarded.)
2. **Lowest sensible denominator.** Prefer `/4`, then `/2`; `/8` only for a
   genuine triplet/compound feel; **never `/16` or finer**.
3. **Sane beats-per-bar** `b`. `b` **must** be in the whitelist **{2, 3, 4, 6}**;
   candidates with an eccentric `b` (5, 7, 13, …) are **rejected**, not merely
   dispreferred. Meter here is a layout device, and an eccentric meter is the
   "counterintuitive time signature" we exist to avoid.
4. **Even bar count** `M`, and among even options the split whose **bar boundary
   lands on the musical seam** (the `LayoutHints` seam index) — for an up/down
   pattern the turnaround, so line 1 is the ascent and line 2 the descent.
5. **Larger beats-per-bar** `b`. Among candidates still tied after seam alignment,
   prefer the larger `b` — fuller bars, fewer lines (e.g. `4/4 × 4` over
   `2/4 × 8` for the same 16 beats).
6. **Fewest levers used.** Among ties, prefer the candidate that changed the note
   count least (ideally not at all).

**Sane-`b` outranks even-`M`, and the lever fires before either is sacrificed.**
The two goals — a clean simple meter and an even bar count — usually agree but can
collide: `B = 14` tiles only as `7/4 × 2` (even but eccentric), `2/4 × 7`, or
`14/4 × 1`. Rather than accept `7/4` (rung 3 rejects it) *or* silently abandon
even, the fitter first engages the **smallest note-count lever** (4.6) to reach a
`B` that yields both a whitelisted `b` **and** an even `M` (14 → 12 → `6/4 × 2`,
or 14 → 16 → `4/4 × 2`). Only if **no small lever** produces such a `B` does it
fall back to an **odd-but-sane** count (e.g. `3/4 × 3`) — never to an eccentric
meter. Given the up/down default, patterns that force this fallback should be
rare.

### 4.6 Step 5 — Note-count levers (last resort)

The levers, smallest-effect first:

- **Toggle apex** — play the top note/cell once vs. twice (changes `N` by one
  cell).
- **Repeat an end note** — repeat the first or last note/cell.
- **Add or drop one** — extend to or trim from a natural boundary.

The fitter applies the **smallest** edit that unlocks a low-denominator, even-bar
fit (the 15/16 → 16/16 → 4/4 case). Which levers are legal is family-declared, so
the fitter never produces a musically nonsensical sequence.

### 4.7 Legibility trace

The fitter records, on the `Score` (or alongside it), **why** it chose a meter:
the winning candidate, the chosen `g`/pulse, any levers applied, and the runners-up
it rejected and at which ladder rung. This makes corner-case tuning a matter of
reading the trace, and gives tests a stable assertion surface beyond the emitted
bytes.

## 5. Repeat barlines

No repeat support exists today (`Score`/`Measure`/`Note` have no repeat field;
`emit.py` emits no repeat tokens). This epic adds it:

- **`Score` model** — represent an exercise-level repeat (open/close spanning the
  whole exercise; count defaulting to the conventional two passes). Keep it in the
  renderer-agnostic core (`score.py`), not the emitter.
- **`emit.py`** — emit the alphaTex repeat tokens on the first and last bars.
  Confirm the exact tokens against the vendored `melete-render` alphaTab version
  (a spike-sized check) before finalizing.

Every exercise is wrapped by default. Repeat is a property of the engraved
exercise, applied after the fitter has settled the bars.

## 6. Family rules (6-string bass)

- **Chromatic** patterns use **all strings** of the bass (bass6: B E A D G C),
  ascending across all six then descending, with the apex handled per the lever.
- **Scales** aim to span **two octaves**, starting on the **lowest string** (B on
  bass6). Where a traversal yields uniform 3-notes-per-string groups, the fitter
  gets a clean `g = 3` (triplets, one string per beat); positional two-octave
  fingerings use the uniform-pulse fallback (4.3).
  - **Two octaves is conditional on feasibility.** `positional` bounds the hand
    to one position (commit `d375ea5`), and for some (root, mode) a two-octave
    positional box cannot hold every degree on bass6. The family attempts two
    octaves in the requested traversal; where that traversal cannot span two
    octaves, it falls back by a **declared rule** — either drop to **one octave
    played up-and-down** (still fills even measures) or switch that exercise to a
    traversal that does span two (e.g. a shifting/`three_nps` fingering). The
    fallback is **never silent**: it is recorded in the layout trace, so a
    "positional" scale never quietly shifts position to fake two octaves.
- **Direction is per-exercise.** `up_down` is the **default** — drilling both
  directions is the norm. `ascending` and `descending` remain explicit options;
  some exercises are deliberately one-directional. For one-directional exercises
  the fitter reaches even measures via meter + note-count levers rather than a
  turnaround.

All of the above targets **bass6**. Fewer-string strategies are a non-goal (§2).

## 7. Pipeline integration

- The fitter is a new pass operating on the `(voice, LayoutHints)` a family emits
  (§4.2), sitting **between** `family.generate` and the barring pass `bar()`. It
  sets the Score's `time_signature`, restamps note durations/tuplets, applies
  note-count levers, and attaches the repeat wrapper and legibility trace. `bar()`
  then tiles a voice that is **guaranteed** to fill whole measures. `generate()`'s
  return type widens from `Score` to `(Score, LayoutHints)` across all four
  families.
- **`rhythm.py` / config change.** The sampled `time_signature` and `subdivision`
  axes in `[pool.rhythm]` are **removed** — the fitter derives them. Orthogonal
  rhythm overlays that preserve the beat grid (e.g. `long_short`, where a
  long+short pair still occupies one group) may remain, but must not reintroduce
  partial bars; any that cannot are dropped. This is a deliberate reduction of the
  config surface and must be reflected in `examples/config.toml` and the docs.
- The renderer boundary is respected: all fitter logic is renderer-agnostic and
  lives outside `alphatab/`. Only the repeat-token emission touches `emit.py`.

## 8. Worked examples: the five exercises

These illustrate the fitter's decision process. Exact final note counts for the
positional/broken cases are pinned by implementation + golden tests (§10) and are
among the corner cases we expect to tune; the **meter decisions** below are the
target.

**01 — Chromatic 1-2-4-3, all six strings, up and down.**
Cell `g = 4` (fingers/string). **Base cycle** (apex once): `there_and_back` over 6
strings = 11 string-groups × 4 = **44 notes** → `B = 11` beats, a prime with no
sane `b` — so the fitter **engages `APEX_REPEAT`** (repeat the apex string's
4-note cell) → **48 notes** → `B = 12` beats of four sixteenths (`d = 4`). Divisor
pairs of 12 with even `M`: `6/4 × 2`, `3/4 × 4`, `2/4 × 6`. Ladder picks
**6/4 × 2** — the bar boundary lands exactly on the turnaround (bar 1 = ascent,
bar 2 = descent); `3/4 × 4` is the acceptable alternative. This is the exemplar of
the apex lever doing its job. Wrap in repeat.

**02 — A♭ Ionian, positional, two octaves, up and down.**
Positional two-octave fingering → unequal per-string groups → **uniform-pulse
fallback** (eighths). A raw up/down count near 29–30 is prime-ish and tiles only
into odd or fine-grained bars — the case the levers exist for. The fitter repeats
an end/apex note to reach **32 eighths → 4/4 × 4** (even, `/4`, seam at bar 2|3).
This is the exemplar of the note-count lever doing real work, and of "two octaves
from the low B."

**03 — A Phrygian, positional, descending, groups of 3.**
Cell `g = 3` (the "groups of 3" window) → triplet pulse, one group per beat.
Direction `descending` (one-directional, no turnaround). `B = N/3` beats tiled
into even bars; if `B` is odd, drop/repeat one group to reach an even `M` at
`3/4` or `4/4`. (The current dotted-16th+32nd `long_short` overlay is orthogonal
and only kept if it preserves the grid.)

**04 — D♭ m7, first inversion, up and down, broken.**
Broken arpeggio in triplet-eighths, `g = 3`. Current cycle ≈ 27 triplet-eighths =
9 beats (partial). The fitter tiles to the nearest even-bar target: **drop to 24
(8 beats) → 4/4 × 2**, or extend to 12 beats → `6/4 × 2`, whichever the legal
levers and the seam favor. Wrap in repeat.

**05 — C chromatic fourths, adjacent strings, ascending pairs, descending.**
Cell = the ascending pair. Current 26 sixteenths tiles to 6.5 beats (partial).
The fitter nudges the count to a friendly multiple — e.g. **24 sixteenths → 6
beats → 3/4 × 2**, or 32 → `4/4 × 2` — choosing the even, `/4`, seam-aligned
option. Wrap in repeat.

Across all five, the same machinery applies: build the cycle, read the cell,
tile whole beats into even simple bars, spend a note only when the count is
stubborn, wrap in repeats.

## 9. Output-directory cleanup and the build/ convention

The Aug 12 artifacts were written to a **sibling directory that is a separate git
repository** (`../sample-gp/`). This is a **development-time discipline problem,
not a code defect** — and the fix is a convention, not a pipeline change.

- **The code's runtime behavior is correct and stays as-is.** `melete generate`
  writes to `./sessions/<date>/` relative to the current working directory. That
  is fine and this epic does **not** change it.
- **The discipline: run melete from within `build/` during development.** When we
  run the generator inside the repo as part of the work, we run it from the
  gitignored `build/` directory so artifacts land in `build/sessions/…`. We stop
  dropping generated results into the repo base directory or — as happened — a
  sibling git repository.
- **Assert the convention in the repo `MEMORY.md`** (repo is public; this is
  melete-specific). Proposed entry:

  > **In development, run melete from `build/`; never write generated artifacts to
  > the repo base or a sibling repo.** melete's runtime writes generated practice
  > artifacts (`.gp`, `.atex`, `session.json`) to `./sessions/<date>/` relative to
  > the working directory — this is intentional and unchanged. When running the
  > generator inside the repo during development, run it from the gitignored
  > `build/` directory so output lands in `build/sessions/…`. Never write
  > generated artifacts into the repo base directory, the VM scratchpad/temp
  > (invisible from the user's macOS host), or a sibling directory one level above
  > the repo (those are separate git repositories — the `sample-gp/` mistake).

This is a documentation + memory task within the epic (no code change to the
output path); the `MEMORY.md` write follows the repo's human-approval memory
policy (already granted for this entry).

**Acknowledged out of scope (future rethink).** The flat-file `sessions/` model,
and the way exercise selection depends on *past* sessions, is good enough to start
but will not scale once months of session directories accumulate — that data
-management redesign is a separate future effort, not part of this epic.

## 10. Testing strategy

- **Fitter unit tests.** For representative patterns assert the derived
  `(subdivision, time_signature, bar_count)`, that the total tiles with **zero
  remainder**, that `M` is even where the ladder promises it, and that the
  **legibility trace** names the winning candidate and any levers used. Include
  the awkward cases (prime-ish counts, positional scales) to lock the lever
  behavior.
- **No-partial invariant.** A property test: for every family × representative
  axis combination, the emitted score has no partial final measure.
- **Repeat emission.** Golden alphaTex asserting the open/close repeat tokens on
  the right bars, validated end-to-end through `melete-render` to a loadable
  `.gp` (extending the existing black-box integration test from epic #46).
- **The five exercises as goldens.** Regenerate exercises 01–05 and freeze their
  corrected alphaTex/`.gp` as regression goldens — the acceptance artifact.
- **Output-path test.** Assert the writer targets `build/` and never a path
  outside the repo.

## 11. Task breakdown and bookends

Bookend tasks (seeded at creation):

- **#58 — Documentation** (`.github`): this spec + the plan. First bookend.
- **melete#110 — Documentation review** (melete): closing sweep across docs;
  spawns per-repo doc tasks as needed. Runs before the retrospective.
- **#59 — Retrospective** (`.github`): terminal bookend; closes the epic.

Anticipated implementation tasks (finalized in the plan, `writing-plans`):

1. **Repeat-barline support** — `Score` model + `emit.py` + tokens verified
   against alphaTab; golden test.
2. **Full-cycle semantics** — turnaround/apex-repeat lever; chromatic all-strings;
   scales two-octaves-from-lowest (bass6); `up_down` default + per-exercise
   direction.
3. **The layout fitter** — cell-aware pulse, candidate generation, priority
   ladder, note-count levers, legibility trace; pipeline insertion.
4. **Config/rhythm reduction** — remove sampled meter/subdivision; reconcile
   `long_short`; update `examples/config.toml`.
5. **Output-dir cleanup + `MEMORY.md`** — route to `build/`, gitignore, remove
   `sample-gp/`, assert the memory.
6. **Regenerate the five exercises** as goldens (acceptance).

A **validation** task (regenerate the day and confirm clean engraving with no red
bars in Guitar Pro) may be seeded at plan time.

## 12. Risks, assumptions, open questions

- **Positional two-octave scales are the hard case.** Unequal per-string groups
  force the uniform-pulse fallback and real note-count edits, and two octaves may
  not fit a `positional` box — resolved in §6 by a declared, trace-recorded
  fallback (drop to one octave up-and-down, or switch traversal). Risk: a chosen
  edit reads as musically odd. Mitigation: family-declared legal levers + the
  legibility trace + goldens; expect iteration.
- **alphaTex repeat tokens.** Exact syntax/behavior must be confirmed against the
  vendored alphaTab version before the model is finalized (spike-sized).
- **`long_short` and other rhythm overlays.** Must preserve the beat grid or be
  dropped. Open question: keep them as an orthogonal overlay, or fold expressive
  rhythm into a later epic. Note `accent_pattern` is a third `[pool.rhythm]` axis
  that is grid-preserving and can stay.
- **Even vs. sane meter.** Resolved in §4.5: a whitelisted `b ∈ {2,3,4,6}` is a
  hard filter that outranks even-`M`, and the note-count lever fires before either
  is sacrificed; the odd-but-sane fallback (`3/4 × 3`) is used only when no small
  lever helps. Residual risk is only that the "small lever" bound needs tuning.
- **Seam definition for non-up/down patterns.** For one-directional exercises the
  `LayoutHints` seam is less obvious; the ladder falls back to the most balanced
  even split.
- **Assumption:** targeting bass6 only keeps the octave-span rules tractable;
  4-/5-string generalization is deferred and may reopen pulse/lever choices.
