# Coherent fretboard journeys and the layout substrate

**Design specification, v1.0**
**Date:** 2026-08-14
**Org:** `mnemosys-project`
**Repository:** `mnemosys-project/melete`
**Epic:** [`mnemosys-project/.github#72`](https://github.com/mnemosys-project/.github/issues/72)

## Table of Contents

- [1. Overview](#1-overview)
- [2. Scope](#2-scope)
- [3. Architecture](#3-architecture)
- [4. The layout strategy](#4-the-layout-strategy)
- [5. The one-hand journey](#5-the-one-hand-journey)
- [6. Fingering styles and per-family coherence](#6-fingering-styles-and-per-family-coherence)
- [7. Grouping and pattern semantics](#7-grouping-and-pattern-semantics)
- [8. Selection and configuration](#8-selection-and-configuration)
- [9. The two-hand seam (built now, unused now)](#9-the-two-hand-seam-built-now-unused-now)
- [10. Error handling](#10-error-handling)
- [11. Testing strategy and acceptance](#11-testing-strategy-and-acceptance)
- [12. Recorded decisions](#12-recorded-decisions)
- [13. Deferred](#13-deferred)

## 1. Overview

melete generates bass practice exercises by sampling abstract axes, realizing
each draw through a family into a concrete `(string, fret)` voice, and metering
the result into whole bars. The generative model works; the **geometry** does
not. Direction, root anchor, string coverage, octave span, and grouping are each
sampled as *independent* draws (`selection.py`) and merely resampled until the
result passes a note-count and hand-span budget (`selection._rejected`,
`selection.py:443-482`). Nothing computes a coherent journey, so individually
"valid" draws compose into musically incoherent exercises.

This epic replaces sampled geometry with **computed journeys**. Every exercise
becomes a single up-and-down round trip, anchored at its root on the lowest
string, covering the neck under an explicit **layout strategy**, with grouping
patterns unwound correctly across the full span. In the same stroke it extracts
the **placement substrate** — a hand-reach-aware boxing primitive generalized to
one *or two* anchors — that two-handed tapping
([`#67`](https://github.com/mnemosys-project/.github/issues/67)) will be rebased
onto.

**Success criterion.** With the default configuration, `melete generate` produces
a practice sheet whose every scale, arpeggio, chromatic, and interval exercise is
a coherent journey: it starts on an outer string (the root's string for
scales/arpeggios) in the lower neck, travels under a named fingering style to the
turnaround, and returns as the exact retrograde — playable, complete, and
faithful to its title. Held against a sheet, the player and instructor can say
yes or no, and the regenerated five acceptance exercises replace the current
goldens.

**The four defects this closes** (from the 2026-08-13 goldens, epic #57):

| # | Exercise | Defect | Root cause |
|---|---|---|---|
| 1 | Chromatic 3-1-4-2, descending | ~3 of 6 strings; starts mid-neck; repeats a string; one direction | string set and start string sampled independently; `direction` sampled |
| 2 | Bb blues, ascending | root on fret 1 of the A string, not Bb on the low B string | `_shared.boxed` minimizes travel from an arbitrary base; anchor not pinned |
| 3 | C major pentatonic, groups of 4 | grouping turns around before completing an octave | underlying span too short; `windowed` runs out of notes |
| 4 | Em7, across strings | odd open strings; collapses to one string; never reaches the top | `_across` holds a string when the next can't reach; no coherent coverage |

## 2. Scope

### In scope

- A first-class **layout strategy** (§4): given a pitch sequence, an anchor, and a
  string set, it computes each note's `(string, fret)` **and** the exercise's
  coverage. It is built over a **generalized boxing primitive** that boxes a pitch
  run under *N* hand-anchors, N in {1, 2}; this epic implements **N = 1**.
- **Coherent one-hand journeys** (§5) for all four families: anchor the root on
  the lowest string in the lower neck (frets 0-12, semi-randomized, open strings
  allowed), ascend across the strings to the turnaround, descend as the exact
  retrograde. Direction is always up-and-down.
- **Fingering styles** (§6): scales carry **positional/boxed** and
  **three-note-per-string**; arpeggios carry **one canonical seed shape per
  quality** with inversions/positions derived, authored under an instructor-gated
  task. Chromatic keeps its fixed four-finger mechanic, made coherent. Intervals
  get the same geometry treatment.
- **Grouping patterns** (§7) unwound as an **overlapping window sliding by one**
  across the full ascending span, reversed for the descent.
- Geometry moves from sampled to computed (§8): the `direction` and `string_set`
  axes are **removed**, octave count is **emergent** from the outer-to-outer journey
  (not sampled, not a target), the root anchor is **pinned** to the lowest string,
  and the shipping `config.toml` is migrated in the same change.
- **Raise-not-clamp** preserved (§10): an unplayable placement raises and the
  validity gate resamples.
- Acceptance (§11) by **regenerating the five exercises and re-freezing the
  goldens** (#137 harness) with human + instructor sign-off.

### Out of scope (deferred, named — §13)

- Two-handed tapping *implementation* — #67, rebased on this substrate.
- Single-string and two-string scale/arpeggio modes.
- Horizontal / diagonal continuations.
- Upper-neck root anchoring.
- The eventual hand-written bespoke exercise library.

### Renderer boundary

Every change is renderer-agnostic and stays behind the `score.py` seam
(`CLAUDE.md`, *The renderer boundary*). The touched modules — `families/_shared`,
`families/*`, `selection`, `config`, `pipeline`, and the `layout` fitter's
interface — never import the alphaTab emitter, and this epic adds no note field,
so `alphatab/emit.py` and the golden alphaTex are untouched except by the
regenerated acceptance goldens (§11).

### Audience

melete is a tool for its author, an advanced player whose instructor sets the
material. That permits assumptions a tool for strangers could not: the fingering
vocabulary is drawn from this player's practice, and the exercises assume a player
who reads standard bass fingerings.

## 3. Architecture

`pipeline.realize` (`pipeline.py:48-76`) is the one place the renderer-agnostic
stages are wired: `family.generate -> layout.plan_voice (fitter) ->
rhythm.restamp -> replace -> Score`. This epic changes **what the family
computes**, not the pipeline's shape. The fitter still meters the note count; the
restamp still stamps rhythm; the redesign lives inside `family.generate` and its
shared helpers.

Today a family resolves placement, direction, coverage, octaves, pattern, and
root **all at once** inside `generate` from independently sampled axes, then the
selector resamples if the result fails the budget. The redesign keeps `generate`
as the placement owner but routes its geometry through a **layout strategy** built
on a shared boxing primitive.

```text
family.generate(profile, params):
  pitches   = theory.<...>(root, type, octaves)      # WHAT (unchanged)
  strategy  = choose_strategy(family, params)         # HOW  (new: fingering style)
  journey   = strategy.lay_out(pitches, anchor, ...)  # coherent (string,fret), up-and-down
  return Score(journey), LayoutHints(...)
```

**The load-bearing boundaries.**

- **The placement primitive is the one hand-aware layout function.** The
  generalization from one hand to two (§9) lives in exactly one primitive in
  `_shared`; nothing else counts hands or reasons about `position_span`. This is
  the seam #67's two-hand strategy plugs into.
- **Coverage is owned by the strategy, not sampled.** A strategy decides which
  strings the journey uses. "Full-neck vertical" is the one-hand default; "narrow
  register-split" is tapping's, reserved. There is no independent `string_set`
  axis to contradict a strategy's coverage.
- **Musical content stays sampled; geometry is computed.** Root pitch-class, scale
  type, quality, inversion, pattern, tempo, and *which fingering style* remain
  sampled draws (the diversity engine). Direction, string coverage, octave span,
  and the concrete anchor are computed.

## 4. The layout strategy

A **layout strategy** answers "how do these pitches sit on the neck?" It is the
home the code has lacked: today the answer is smeared across `scales._places`
(`scales.py:239-261`), `arpeggios._places` (`arpeggios.py:264-288`),
`_shared.boxed` (`_shared.py:223-276`), and `chromatic._strings`
(`chromatic.py:132-156`), each rolling its own placement with no shared concept.

A strategy is defined by:

- **its placement rule** — how a pitch run maps to `(string, fret)` under
  hand-reach constraints (boxed vs three-note-per-string vs the arpeggio shapes);
- **its coverage** — which strings the journey traverses, computed, not sampled.
  The one-hand vertical strategy of this epic covers the **whole instrument** (its
  `strings` is the full profile string range, outer to outer, §5); string *subsets*
  belong to the deferred single/two-string and horizontal modes (§13);
- **its anchor count** — one hand (this epic) or two (the seam, §9).

The strategies are built over **one generalized boxing primitive** in `_shared`,
which replaces the frozen `boxed`:

```text
box(profile, pitch_run, strings, anchors) -> [(string, fret, anchor_index)]
    # anchors is a tuple of 1 or 2 base frets; each pitch takes the reachable
    # position nearest its owning anchor; each anchor's fretted span must satisfy
    # profile.position_span; raises if no assignment fits.
```

For N = 1 this reproduces today's positional placement but with the **anchor
pinned** rather than chosen to minimize travel from an arbitrary base — which is
the fix for defect 2 (the Bb landing on the A string). `instrument.positions`
(`instrument.py:185-196`), `instrument.hand_span` (`instrument.py:165-182`), and
`profile.position_span` remain the reach primitives; `_reachable`
(`_shared.py:204-220`) remains the per-pitch candidate enumerator.

**Migration note.** The existing single-hand `_shared.boxed` is superseded by the
N = 1 path of `box`; its behavior (nearest reachable position within one
`position_span`, issue #57's span rejection) is preserved as the N = 1 case and
its tests carried forward. #67's earlier plan to freeze `boxed` byte-for-byte and
add a sibling `two_hand_boxed` is superseded by this unification (§9, §12).

## 5. The one-hand journey

Every exercise is a single computed round trip, replacing the sampled `direction`
and the independent string/octave draws:

1. **Anchor.** The root is placed on the **lowest instrument string** (the string
   set is the whole instrument, §4), in the lower neck (frets 0-12), semi-randomized
   so the session spreads across the low region rather than repeating one position.
   Open strings (fret 0) are valid positions and are computed, not special-cased
   away (decision 5). For chromatic and intervals, which have no single "root," the
   anchor is the lowest string of the traversal and its starting fret.
2. **Ascend.** From the anchor the journey travels **outward across the strings**
   under the chosen fingering style until it reaches the **opposite outer string**
   — that is the turnaround. **Outer-string-to-outer-string is the governing
   invariant, and octave count is emergent**, not a target: the journey uses every
   string in order with no gaps and no repeats, and the octaves that yields are
   whatever the instrument gives (≈1.5 on a four-string, ≈2 on a five-string, ≈2.5
   on a six-string). This generalizes across instruments where a fixed "two octaves"
   would not, and it is the direct fix for defects 1 and 4 (partial string coverage,
   never reaching the top string).
3. **Turnaround.** The apex note on the opposite outer string plays once.
4. **Descend.** The descent is the **exact retrograde** of the patterned ascent —
   not a separately computed figure. This reuses the existing retrograde
   discipline (`_shared.directed_by_cell`, `_shared.py:309-341`), now the only
   direction the families produce.

Because direction is no longer sampled, `directed_by_cell`'s `up`/`down` branches
become dead for the families and the cell-aware `up_down` turnaround becomes the
single path. The seam for a future single-direction escape hatch (decision 1) is
preserved by keeping direction a strategy parameter with a fixed value, not by
retaining a sampled axis.

## 6. Fingering styles and per-family coherence

### Scales

The fingering style **is** the placement rule. v1 implements the two styles today
mis-modelled as `traversal` values:

- **positional / boxed** — the pitches sit under one hand position; the box climbs
  the neck by string, staying within `position_span`.
- **three-note-per-string** — three scale degrees per string, climbing outward.

The style is a sampled *content* choice (which mapping to drill); the geometry it
produces is computed. Extent is governed by the outer-to-outer rule (§5), not an
octave count: a boxed position on a six-string already spans every string within
one hand, and a three-note-per-string climb ascends string by string to the top
string. The `positional` single-octave fallback that exists today
(`scales.py:311-323`) is subsumed — the journey's extent is defined by reaching the
opposite outer string, so there is no fixed two-octave target to fall short of and
compromise.

### Arpeggios

Arpeggio fingering is genuinely ambiguous — the third can sit on the root's string
or the string above, and different qualities have different idiomatic shapes — so
v1 does **not** infer it. It encodes **one canonical seed shape per quality**
(`maj7, min7, dom7, m7b5, min6`) as reviewable data, and **derives** inversions and
higher positions from the seed by transposition rather than enumerating every
`(quality, inversion, position)` by hand. Because there is one shape per quality,
arpeggios have a **single layout in v1 and no fingering-style axis** — the
`traversal` axis is removed for this family (§8), unlike scales, which keep two
real styles. The seed set is therefore small (≈5
shapes) and the bulk is computed. Authoring those seed shapes is an explicit,
**instructor-gated task scheduled up front** (§11) — the arpeggio family is blocked
on it, so it is named and scheduled, not discovered mid-implementation. This
replaces `_across` (`arpeggios.py:241-261`), whose "hold a string when the next
can't reach" rule produces defect 4. The seed shapes are the specification of
"correct fingering"; the validation task (§11) confirms the derived results against
a sheet.

### Chromatic

Chromatic's mechanic is fixed and unambiguous — four fingers on four adjacent
frets — so it gains no fingering style. Its journey is made coherent: **start on
an outer string of the string set, traverse the full set to the other outer
string, turn around, return**, with the per-cycle fret shift (`_FRET_STEP`,
`chromatic.py:116`) preserved. This removes the mid-neck starts, the partial
string coverage, and the repeated string of defect 1. `_strings`
(`chromatic.py:132-156`) is rewritten to traverse the whole set outward-and-back
rather than walking a sampled direction from a sampled start string.

### Intervals

The intervals family gets the same geometry treatment: up-and-down, anchored on an
outer string, covering the configured string set, with the interval pattern
unwound across the full span (§7).

## 7. Grouping and pattern semantics

Grouping patterns unwind as an **overlapping window sliding by one** across the
**full** ascending span, then reverse for the descent. For C major, groups of 4:
`C D E F / D E F G / E F G A / ...` — each group starts one degree higher and
shares three notes with the last. `thirds` is the window `(degree, degree+2)`
sliding by one (C-E, D-F, E-G, ...); `groups_of_3` is `(0,1,2)`; and so on.

This is the semantics `_shared.windowed` (`_shared.py:190-201`) already intends
(`range(count - max(window))`). Defect 3 is not a window bug: the window ran out
of notes because the underlying ascent spanned a fraction of the intended range.
The journey redesign (§5) supplies the full outer-to-outer ascent the sliding
window needs, so the pattern reaches across the neck as written. The window length remains the fitter's `cell` (one beat), unchanged
(`scales.py:361`, `arpeggios.py:377`).

## 8. Selection and configuration

The split is **content stays sampled, geometry becomes computed**:

- **Removed axes.** `direction` (`vocabulary.py:152-156`) and `string_set` (the
  sampled per-family string sets, e.g. `config.toml [pool.scales] string_sets`)
  are **removed** from configuration and the vocabulary registry. Direction is
  always up-and-down; string coverage is the whole instrument (§4, §5). Removing
  them outright (rather than leaving them inert) keeps the config honest about what
  is actually variable; the up-and-down seam for a future escape hatch lives in
  code (§5), not in a dormant axis. This follows the recent precedent of retiring
  the sampled meter/subdivision axes (`87fda5f`).
- **Emergent octaves.** `range_octaves` (`_shared.py:72`) is **removed** as a free
  `[1, 2, 3]` draw; octave count is not sampled and not a target. It is emergent
  from the outer-to-outer journey (§5) — whatever reaching the opposite outer string
  yields on the instrument at hand.
- **Arpeggio `traversal` removed.** With arpeggio fingering defined by a single
  canonical seed shape per quality (§6), a `traversal` axis for arpeggios would
  select between identical layouts — the "two values of one axis naming one
  exercise" that §9's coverage accounting cannot tell apart and the codebase
  forbids elsewhere. It is removed from the `arpeggios` pool and axis list.
  **Scales keep `traversal`** — positional and three-note-per-string are genuinely
  different fingering styles. The future b3-on-the-root's-string vs
  b3-on-the-string-above split (§13) is where arpeggios regain two real styles.
- **Config migration.** Removing these axes turns their keys unknown to
  `_reject_unknown` (`config.py:173`), so the shipping `build/config.toml` (and any
  working config) must be migrated in the same change — dropping the `directions`,
  `string_sets`, `string_traversals`, and `octaves` keys from every pool — or
  `melete generate` fails to load. This migration is part of the axis-removal work,
  not a follow-up.
- **Pinned anchor.** `root` stays a sampled pitch class, but its realization
  (`selection._realized`, `selection.py:421-435`) pins it to the lowest string in
  the lower neck (§5) instead of the lowest octave at or above the string set's
  open pitch.
- **Retained content axes.** Root pitch-class, scale type, quality, inversion,
  `pattern`, tempo, and the new **fingering style** remain sampled and
  recency-weighted for diversity. The style joins the per-family axis list
  alongside the existing content axes.

The validity gate (`selection._rejected`, `selection.py:443-482`) and its
`max_notes`/`max_fret_span` budgets are unchanged: computed journeys still pass
through it, and an unplayable one still resamples (§10).

## 9. The two-hand seam (built now, unused now)

This epic builds the substrate #67 needs, implementing only the one-hand half:

- **Generalized boxing.** `box` (§4) takes 1 *or* 2 anchors from day one. The N = 1
  path is this epic; the N = 2 path — two hands, each within its own
  `position_span`, partitioned by per-string fret region (low frets to the lower
  hand, high to the upper) — is the primitive #67's two-hand strategy calls. It is
  designed and specified here but not exercised.
- **Coverage as a strategy property.** Because coverage is owned by the strategy
  (§3), tapping's narrow, register-split coverage is a *different strategy*, not a
  sampled `string_set` fighting the one-hand full-neck default. This removes the
  collision between #67's narrow sets and this epic's computed coverage at the
  source.
- **Left to #67.** The `Hand`/`Attack` fields on `Note`, the `tapping.reach`
  modifier, its `pipeline.realize` slot, and the emitter/renderer work remain
  #67's. This epic adds no note field and touches no emitter. The placement seam
  is shaped so those land additively.

**#67 is superseded by design.** Its current spec/plan assume a frozen `boxed`, a
sampled narrow `string_set`, and an inherited `direction` axis — all changed here.
Once this epic lands, #67 is re-brainstormed and rebased on this substrate as its
own epic-create run (§13).

## 10. Error handling

| Failure | Behavior |
|---|---|
| A computed journey cannot be placed under one hand within `position_span` | `box` raises, naming the pitches, string set, and profile; `selection._rejected` resamples. Never clamped to fit. |
| An arpeggio quality has no seed shape defined | Raise at generation, naming the family and quality — a missing seed is a specification gap (the instructor-gated authoring task, §11), not a silent default. |
| A style cannot reach the opposite outer string on the instrument | Raise, naming the family, style, and profile; the journey's extent is undefined if the outer string is unreachable. This surfaces a genuine style/instrument mismatch rather than truncating to a partial journey. |
| An open-string position is required | Allowed — fret 0 is a valid computed position, not an error (decision 5). |

No swallowed exceptions and no layout clamped to fit: a mislabelled exercise is
worse than a resampled one, because the label is the part the student trusts
(project policy, `~/.claude/CLAUDE.md`, *No silent failures*).

## 11. Testing strategy and acceptance

| Component | Approach |
|---|---|
| `box` N = 1 | Property: every returned position sounds its pitch (`tuning[string] + fret == pitch`); the hand span is within `position_span`; the anchor is pinned to the intended string/region. Regression: the superseded `boxed` inputs produce equivalent placements. |
| One-hand journey | Golden per family: the ascent covers the strategy's strings in order with no gaps or repeats; the descent is the exact retrograde; the apex plays once. |
| Coverage | The journey starts on one outer string and reaches the opposite outer string, using every string between with no gaps or repeats; the octave count is whatever the instrument yields (asserted across bass4/bass5/bass6). |
| Grouping | The overlapping sliding window spans the full ascent (defect 3): an outer-to-outer scale in groups of 4 yields the expected overlapping groups end to end, not a run that turns around before reaching the opposite outer string. |
| Anchor | A scale rooted at a given pitch class anchors on the lowest string in the lower neck (defect 2): Bb anchors on the low B string at fret 11, not the A string at fret 1. |
| Chromatic coherence | Starts on an outer string, traverses the full set to the other outer string, returns; no repeated string (defect 1). |
| Removed axes | Config with a `direction`, `string_set`, `string_traversals`, or `octaves` key is rejected by `_reject_unknown`; no draw carries them. |
| Config migration | The migrated `build/config.toml` loads cleanly and `melete generate` runs against it (guards the removed-axis surprise). |
| Arpeggio derivation | A seed shape plus its derived inversions/positions place every arpeggio tone on a string sounding its pitch; the derivation is a transposition of the seed, not a re-inference. |

**Acceptance is by regeneration, not only unit tests.** The five acceptance
exercises are regenerated under the new model and the goldens re-frozen (#137
harness), gated by the **validation task** (`melete#152`): the instructor confirms
each exercise is actually playable and correct — up-and-down, anchored low, fully
covering, correctly grouped, with a playable fingering — before the goldens are
accepted. This is the property no unit test can assert and the epic's true
success criterion (§1).

## 12. Recorded decisions

| # | Decision | Rationale |
|---|---|---|
| 1 | Direction is always up-and-down; the single-direction escape hatch is a code seam, not a retained axis. | Almost every exercise is worth playing both ways; single-direction is a rare future exception. A dormant sampled axis would invite the incoherence this epic removes. |
| 2 | Geometry (direction, coverage, octaves, anchor) is computed; musical content (root, type, quality, inversion, pattern, tempo, fingering style) stays sampled. | The incoherence came from sampling geometry independently and resampling to fit. Computing geometry makes the four defects impossible by construction while preserving the diversity the content axes provide. |
| 3 | Placement is owned by a first-class **layout strategy** over **one generalized boxing primitive** (N in {1,2}), not the frozen `boxed` + bolted `two_hand_boxed` of #67. | Both epics answer "how do pitches sit on the neck." One primitive, parameterized by anchor count, is the honest shared substrate; a frozen primitive plus a sibling duplicates reach math and splits the concept. |
| 4 | Coverage is a property of the strategy, not a sampled `string_set` axis. | A strategy that computes full-neck coverage cannot also honor an independent narrow `string_set` draw. Making coverage strategy-owned removes the collision with #67's narrow tap sets at the source and matches the author's "extent is a property of the exercise." |
| 5 | Open strings are allowed and computed, not special-cased away. | Picking positions correctly makes fret 0 a legitimate choice; premature avoidance is optimization without evidence. Revisited only if the regenerated exercises show it is a problem. |
| 6 | Arpeggio fingering is authored data (seed shapes, decision 12), not inferred. | Arpeggio shapes are genuinely ambiguous and idiomatic; instructor-validated seed data is more honest than an algorithm guessing "traditional fingering," and it is the seam future styles (including tapping) extend. |
| 7 | Grouping is an overlapping window sliding by one across the full span. | Non-overlapping blocks reduce to the plain scale with barlines; the shared-note overlap is the exercise. Defect 3 was a short span, not a wrong window. |
| 8 | Chromatic gains no fingering style but is made coherent (outer-string start, full traversal, up-and-down). | Its mechanic — four fingers, four frets — is unambiguous; only its journey was incoherent. |
| 9 | The `Hand`/`Attack` model, the tapping modifier, and the emitter work stay #67's; this epic builds only the placement seam. | Keeps the renderer boundary clean and the epic sized to v1 while making #67 additive rather than colliding. |
| 10 | #67 is superseded and will be re-brainstormed on this substrate. | It was planned before this feedback; its frozen-`boxed`/sampled-`string_set`/`direction`-axis assumptions are all changed here. Rebasing it is a known enabling chain (§13). The author owns re-linking #67; this epic asserts the supersession but takes no tracker action. |
| 11 | Extent is governed by **outer-string-to-outer-string**, with octave count emergent, not a two-octave target. | Bouncing off both outer strings is the real invariant; it uses the whole neck and generalizes across instruments (≈1.5 octaves on a four-string, ≈2.5 on a six-string) where a fixed two octaves would either overshoot or leave the top string unused — the latter being defect 4 itself. |
| 12 | Arpeggio fingering is one canonical **seed shape per quality** with inversions/positions **derived**, not an enumerated table. | Keeps the hand-authored, instructor-gated data set small (≈5 shapes) and the bulk computed, so the arpeggio family is unblocked by a scheduled up-front task rather than a large enumeration discovered mid-build. |

## 13. Deferred

Named so the boundary is explicit and the design leaves room for each.

- **Next (its own epic).** Two-handed tapping (#67), re-brainstormed and rebased on
  this substrate: the N = 2 boxing path, the register-split coverage strategy, the
  `Hand`/`Attack` model, and the articulation renderer work.
- **Next version.** Single-string and two-string scale/arpeggio modes — an advanced
  case the author does not need yet. The traversals dropped as unbuilt in v1 return
  with the modes that give them meaning: scales' `octave_per_string` and
  `single_string`, and arpeggios' `across_strings` and `single_string`.
- **Next version.** A real arpeggio fingering-style axis — the b3 on the root's
  string vs the string above — reinstating `traversal` for arpeggios once there is
  more than one seed shape per quality to choose between (§6, §8).
- **Next version.** Horizontal / diagonal continuations — staying in a vertical
  neck slice and shifting the root, and diagonal three-note-per-string climbs.
- **Next version.** Upper-neck root anchoring — roots in the upper half of the
  neck; low anchoring covers the common, practical case for v1.
- **Later.** A hand-written **bespoke exercise library** for shapes too complex to
  derive from general rules, pulled from a curated source rather than generated.

---

**Status:** Draft. Filed as epic
[#72](https://github.com/mnemosys-project/.github/issues/72).
