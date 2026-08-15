# Two-handed tapping — a tabulated tap-shape vocabulary for melete

**Design specification, v2.0 (rebased onto the coherent-journey layout model)**
**Date:** 2026-08-15
**Org:** `mnemosys-project`
**Repository:** `mnemosys-project/melete`
**Epic:** [`mnemosys-project/.github#67`](https://github.com/mnemosys-project/.github/issues/67)
**Supersedes:** v1.0 (2026-08-13), which predated epic #72 (the coherent
up-and-down journey). See [§13 What changed in the rebase](#13-what-changed-in-the-rebase).

## Table of Contents

- [1. Overview](#1-overview)
- [2. Scope](#2-scope)
- [3. Architecture](#3-architecture)
- [4. Data model](#4-data-model)
- [5. The tap-shape vocabulary](#5-the-tap-shape-vocabulary)
- [6. Placement: the two-hand box and the tapped journey](#6-placement-the-two-hand-box-and-the-tapped-journey)
- [7. Selection and configuration](#7-selection-and-configuration)
- [8. Rendering](#8-rendering)
- [9. Error Handling](#9-error-handling)
- [10. Testing Strategy](#10-testing-strategy)
- [11. Recorded Decisions](#11-recorded-decisions)
- [12. Deferred](#12-deferred)
- [13. What changed in the rebase](#13-what-changed-in-the-rebase)

## 1. Overview

Two-handed tapping is the largest category of the author's course material that
melete cannot generate. This epic adds it as a **tabulated tap-shape
vocabulary** for the `arpeggios` family: a curated set of two-hand fingering
choreographies, one per (chord quality × inversion), that the family walks up
and down the neck the same way it already walks a one-hand arpeggio.

The v1.0 spec modelled tapping as a derived, cross-cutting modifier that split
notes to hands by a fixed geometric rule. Direct testimony from the player the
tool is built for retired that model: the two-hand choreography **is not
derivable from a simple rule** — the hand-and-finger assignment *leapfrogs*,
changing with every inversion as the shape climbs, and this per-inversion
variation is the technique, not an incidental detail. So the honest v1
architecture **tabulates** the choreography — captures it as instructor-validated
data — rather than deriving it, and leaves algorithmic derivation as the explicit
next stage the data seam is built to accept (§12).

**Success criterion.** With tapped triads configured in the `arpeggios` pool,
`melete generate` produces a practice sheet whose arpeggio exercise is a correct
two-handed tapped realization of a triad — the triad walked up and back across
the neck, each inversion voiced across two fretting hands with the tap, and
hammer/pull articulations a player can read and play — and the same session log
replays to the same sheet. Held against a sheet, the player can say yes or no.

**What it inherits, and what it does not.** It inherits the whole post-#72
generative model: the families, the **coherent up-and-down journey** each one now
computes across the whole instrument, the `box` placement primitive, the
selector, the session log, and the alphaTab rendering path. It does **not**
inherit the assumption those were built on — that exactly one hand touches the
fretboard. That assumption is generalized here from one hand to one *or two*
hands, and no further: a fretting-hand count above two is not a musical case and
is explicitly out of scope (§11, decision 2).

This is, as far as the author can determine, the first attempt to codify
two-handed tapping choreography in software. The tabulated vocabulary is both the
v1 deliverable and the raw material for a later, more ambitious contribution:
once enough shapes are captured, the mathematical structure underneath them may
support deriving the choreography algorithmically (§12).

## 2. Scope

### In scope (v1)

- Two new orthogonal `Note` fields: `hand` (`LEFT | RIGHT`) and `attack`
  (`TAPPED | PLUCKED | SLURRED`), plus the generalization of `finger` from
  "left hand 1–4" to "finger 1–4 of `hand`". `PLUCKED` and `LEFT` are the
  defaults, so every existing family, golden file, and test is unchanged.
- A **curated triad tap-shape vocabulary** — one two-hand choreography per
  (quality × inversion) for the four triads `maj`, `min`, `dim`, `aug` — living
  beside the existing one-hand `arpeggio_shapes.SEED_SHAPES`. **PROVISIONAL**
  musical data, authored and confirmed by the instructor and gated on its own
  validation task, exactly as the one-hand seed shapes are gated on `melete#152`.
- Realization of `box`'s reserved **two-anchor path** in `families/_shared.py`:
  given a shape's hand partition, place each hand's tones within its own
  `position_span` on the string set, preserving pitch. Raises when a two-hand box
  cannot be laid out; §9's validity gate resamples.
- A **tapped-journey driver** in the `arpeggios` family that walks a triad's
  inversions up and back across the neck, realizing each inversion's tap shape
  and deriving legato (hammer-on/pull-off) within each hand's same-string runs.
- A one-line change to `rhythm.restamp`'s accent pass so it never accents a
  `SLURRED` note (§9) — the only edit to the shared rhythm module.
- Selection by **treating tapped triads as quality candidates** in the
  `arpeggios` pool: a triad quality routes to the tapped journey, a seventh to
  the one-hand journey, and `hands` (`1 | 2`) is **derived from the drawn
  quality** and recorded in the session log for reproducibility. No unshaped
  tapped pair is representable, because `hands` is not sampled independently of
  quality (§7).
- A prerequisite **rendering spike** (§8): confirm the alphaTex path can express
  right-hand tap, left-hand tap, hammer/pull, **and per-hand (right-hand)
  fingering**, or establish the workaround.

### Non-goals (v1) — deferred, named, not foreclosed (§12)

- **Scales tapping.** Scale tapping is a distinct choreography with its own
  complexity and is deferred whole. Only `arpeggios` is tapping-eligible in v1.
- **Seventh-chord tapping.** v1 tabulates the four triads only. Sevenths add a
  chord tone and a fourth inversion, and lean toward the stretched shapes below.
- **Stretched / re-voiced shapes.** Shapes that separate the hands by an octave
  (e.g. 1–5 in the left hand, 3–7 an octave up in the right) **change the pitches
  that sound**. v1 stays in one octave range and preserves pitch (§4, §10); the
  pitch-changing tier is the explicit advanced follow-up.
- **Deriving the choreography algorithmically.** v1 tabulates. Deriving the
  shapes from theory is the intended successor the data seam is built for (§12).
- **Three or more fretting hands.** Not a musical case; the model is bounded at
  two on purpose (§11, decision 2).
- **`intervals` and `chromatic` tapping.** Nothing in the corpus taps them, and
  chromatic's exercise *is* its left-hand fingering — tapping would contradict
  its subject.

### Audience

melete is a tool for its author, an advanced player whose instructor sets the
material. That permits assumptions a tool for strangers could not make: the
tapping vocabulary is drawn directly from this player's course corpus and
validated by his instructor, and the exercises assume a player who already taps.

## 3. Architecture

Post-#72, every family computes **one coherent up-and-down journey** across the
whole instrument — outer string to opposite outer string and back — and owns its
own placement, calling the shared `box` primitive (`families/_shared.py`) for the
hand-reach math. Two-handed tapping follows that same seam rather than fighting
it. It is realized in three layers.

**The three layers.**

- **Cross-cutting (the whole tool).** `Note` gains `hand` and `attack` (§4);
  `rhythm.restamp` learns to skip accents on slurred notes (§9); the emitter
  renders the new articulations and per-hand fingering (§8). These are the only
  edits outside the `arpeggios`/`_shared` neighborhood.
- **Shared mechanics (`families/_shared.py`).** `box`'s reserved two-anchor path
  (§6) places a supplied partition across two hands; the legato derivation and
  articulation stamping are written so a future scales driver reuses them. This
  is the two-hand sibling of the one-hand `box`/`journey` machinery #72 built.
- **Family-specific (`families/arpeggios.py`, a new tap-shape module).** The
  curated triad tap-shape vocabulary (§5) and the tapped-journey driver that
  walks a triad's inversions across the neck applying those shapes (§6).

**Placement happens at family generation, not after a fitter.** This is the
load-bearing reversal from v1.0. Because the choreography is *inversion-indexed*,
it is produced where the inversion is still known — inside `arpeggios.generate` —
not downstream of the layout fitter, by which point the voice is a flat note
sequence stripped of inversion structure. The pipeline gains one small stage —
legato derivation, after the fitter:

```text
pipeline.realize:
  arpeggios.generate ─▶ layout.plan_voice ─▶ legato ─▶ rhythm.restamp ─▶ replace ─▶ (Score, LayoutPlan)
   (which pitches +      (fitter: meter,       (derive     (when; never      (assembly;
    two-hand placement    subdivision,          TAPPED/     accents a slur)    realize
    + hand + finger)      tile into bars)        SLURRED                        returns the
                                                 on tiled                       plan too)
                                                 voice)
```

The fitter tiles the tapped journey into whole bars exactly as it tiles a
one-hand journey; each note's placement, `hand`, and `finger` survive tiling
unchanged. **Legato is derived after the fitter**, on the final tiled voice (a
small shared pass in `pipeline.realize`), so a note the fitter repeated or
dropped still yields a correct first-`TAPPED`-per-run rather than a stranded
hammer-on — this is the one place the tapped journey and the fitter interact, and
running legato last makes that interaction correct by construction.
`rhythm.restamp` then stamps accents, skipping slurred notes (§9). Any stage may
raise, and §9's validity gate resamples — the existing contract. `realize`
returns `(Score, LayoutPlan)` as it does today; both call sites (`cli._score`,
the selector's validity gate) already discard the plan.

**The load-bearing boundaries.**

- **The tap-shape data boundary.** The choreography lives in one curated module
  as data, not as code branching on quality and inversion. The driver is
  quality-agnostic: it looks a shape up and applies it. This is what makes adding
  a quality (or, later, a scale, or a derived generator) additive.
- **`box` as the one hand-reach-aware primitive.** The generalization from one
  hand to two lives in `box`'s anchor count — 1 or 2. Nothing else counts hands;
  the `position_span` reach bound (issue #57) is applied once per hand there.
- **The journey backbone is shared.** A tapped arpeggio is the *same up-and-down
  journey* the family already produces, re-placed two-handed. Tapping does not
  introduce a second traversal model.

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

- `hand: Hand = Hand.LEFT` — the fretting hand. Single-hand families always emit
  `LEFT`.
- `attack: Attack = Attack.PLUCKED` — the articulation. The default preserves
  every existing family and golden file unchanged.
- `finger: int | None` keeps its type (1–4 or `None`) but its meaning generalizes
  to "finger 1–4 of `hand`". Today only `chromatic` assigns fingers; the other
  three families emit `finger=None`. **Tapping reintroduces fingering for
  `arpeggios` in the tapped case**: a triad tap shape prescribes a finger for
  every note (that prescription is a core part of the choreography being
  captured), so the emitter must resolve `finger` per `hand` — a right-hand
  `finger=1` is a different finger from a left-hand `finger=1` (§8).

**Pitch is preserved (v1).** A v1 tap shape re-places the triad's own notes in
the same octave range it would otherwise occupy; it never adds octave
displacement (that is the deferred stretched tier, §2). The `Note` invariant
`pitch == tuning[string] + fret` therefore holds note-for-note, and the multiset
of pitches before and after placement is identical (§10).

The `hand`×`attack` cross-product covers the whole v1 vocabulary: RH-tap
`(RIGHT, TAPPED)`, LH-tap `(LEFT, TAPPED)`, an ordinary note `(LEFT, PLUCKED)`,
and a hammer/pull target `(either, SLURRED)`. This mirrors the GPIF encoding the
R&D survey found, where `Tapped`, `LeftHandTapped`, and
`HopoOrigin`/`HopoDestination` are independent note-level properties.

## 5. The tap-shape vocabulary

This is the heart of the epic and the genuinely new artifact. A **tap shape** is
the two-hand fingering choreography for one (quality, inversion): for each chord
tone, which hand frets it, with which finger, and where it sits relative to an
anchor. The shapes are the two-hand sibling of the one-hand
`arpeggio_shapes.SEED_SHAPES` and live beside them.

**Structure.** For each of the four triads (`maj`, `min`, `dim`, `aug`), and each
of its inversions (root, first, second — a triad has no third inversion), a shape
is an ordered sequence of chord-tone placements, each carrying:

- **hand** — `LEFT` or `RIGHT` (the partition; see below);
- **finger** — 1–4 of that hand;
- **string offset** and **fret offset** — the tone's position relative to the
  shape's anchor, from which `box` derives the absolute `(string, fret)` while
  honouring each hand's `position_span`.

The pitch of each placement is the triad's own chord tone at the journey's
current register (pitch preserved, §4); the shape supplies only *where and how*,
not *which pitch*.

**The partition is data, not a derived rule.** The v1.0 spec split notes to hands
by a fixed "low frets → left, high frets → right" geometric rule and claimed it
reproduced every surveyed shape. Player testimony contradicts that: the hands
**fold across each other and leapfrog**, and *which* scale degree each hand voices
**changes with every inversion**. In a root-position minor shape the left hand may
voice the root and third while the right voices the fifth and the next root; at
the next inversion the left hand takes both thirds and the right keeps the fifth
and root — a reassignment no single fret-region rule produces. So each shape
**names its own partition explicitly**, and the leapfrog is simply the sequence of
those per-inversion partitions as the journey climbs. (No global fret-ordering
invariant such as "every left fret below every right fret" holds — the hands
cross; see §11 decision 6.)

**Provisional and instructor-gated.** These shapes are musical judgement, not
code. They are marked PROVISIONAL and confirmed by the instructor before the
generated sheets are trusted, exactly as `arpeggio_shapes.py`'s one-hand seed
shapes are gated on `melete#152`. The plan gives the tap-shape data its **own
validation task** so the code can land and be tested against the shapes as
authored while the musical confirmation proceeds in parallel.

**Coverage note — dim and aug are required.** The four triads are not
interchangeable in priority: diminished and augmented are load-bearing because so
much of the course material derives from the harmonic-minor scale and its modes,
where those triads dominate. v1 must ship working `dim` and `aug` tap shapes, not
only `maj`/`min`.

## 6. Placement: the two-hand box and the tapped journey

**`box`'s two-anchor path.** Today `box(profile, pitches, strings, anchors,
family, axes)` places pitches under a *single* pinned anchor and raises when the
fretted span exceeds `profile.position_span` (issue #57); it already reserves the
two-anchor case for this epic, raising `NotImplementedError` naming #67 for any
`len(anchors) != 1`. This epic realizes that path: given two anchors (left lower,
right higher) and the shape's per-tone hand assignment, it places each hand's
tones near that hand's anchor, requires each hand's fretted span to satisfy
`position_span`, requires both hands non-empty, preserves pitch, and returns each
note's `(string, fret, hand)`. The single-hand path is untouched — the two-hand
case is the `anchors` count, not a mode of the one-hand logic — so the existing
placement tests remain its guard (§10). Unlike the v1.0 design, the partition is
**supplied by the shape**, not derived inside `box`; `box` owns only the reach
math, keeping it the one place that counts hands and honours `position_span`.

**The tapped journey.** The `arpeggios` tapped-journey driver produces the same
coherent up-and-down traversal the one-hand family produces — the triad walked
across the neck outer-string-to-opposite-outer and back — but at each step it
realizes the current inversion's tap shape via `box`'s two-anchor path, stamping
`hand`, `finger`, and (via legato, below) `attack`. The driver is
quality-agnostic: it reads the shape for `(quality, inversion)` and applies it,
so `dim` and `aug` are data entries, not code paths.

**The v1 chaining rule (deterministic).** The driver walks the inversions in
ascending order — root, first, second, then repeating — one tap shape per octave
register, anchoring each shape at the lowest string that sounds its lowest tone
at that register. It climbs until the next shape's tones would leave the neck or
fail `box`'s two-anchor `position_span`, then reverses back down the same shapes.
This is a deliberately minimal rule, chosen so the tapped journey is
**deterministic and testable** in v1; the exact anchor progression is confirmed
against the tabulated shapes during implementation. Choosing anchors *optimally*
across many octaves — the general leapfrog-climb problem — is the deferred hard
part (§12), and the raise condition above is the mechanical boundary between what
v1 lays out and what it refuses.

**The multi-position climb is the honest hard edge.** Choosing anchors as the
two-hand shape ascends the neck across the journey is where this is genuinely
difficult, and v1 solves only as far as the tabulated shapes and the corpus
support. A journey position that cannot be laid out under two hands **raises**
rather than forcing a bad fingering, and §9's validity gate resamples. This
mirrors #72's one-hand journey, which likewise raises (rather than truncating)
when a run cannot be reached.

**Legato.** Within a hand's run of consecutive notes on the *same string*, the
first is `TAPPED` and the rest are `SLURRED` — a hammer-on where the fret
ascends, a pull-off where it descends. Any string change (or hand change) forces
a fresh `TAPPED`. This is derived from geometry, not tabulated per shape (§11
decision 7), and it runs **after the fitter** on the final tiled voice, so a note
the fitter repeated or dropped still yields a correct first-`TAPPED`-per-run
rather than a stranded slur (§3).

## 7. Selection and configuration

Tapping is controlled, not freely sampled: the author wants to dial a
tapping-heavy practice diet, not receive the occasional surprise tap. The
mechanism is deliberately minimal — it rides the existing recency-weighted
candidate sampling rather than adding cross-axis machinery, because the sampler
draws every axis independently and cannot couple `hands` to `quality` (§13).

- **Tapped triads are quality candidates.** The `arpeggios` pool
  (`[pool.arpeggios] qualities`) may list the four triads (`maj`, `min`, `dim`,
  `aug`) alongside the one-hand sevenths. Because no one-hand triad seed shape
  exists (and none is added — decision 10), a **triad candidate is inherently a
  tapped candidate**: the family routes any triad through the tapped journey and
  any seventh through the one-hand journey.
- **`hands` is derived, not sampled.** The selector does not draw `hands`
  independently; it is a function of the drawn quality (triad ⇒ `2`, seventh ⇒
  `1`), written into `params` and recorded in the session log so coverage
  accounting and replay see it like any other axis. This is what makes an
  unshaped tapped pair **unrepresentable** — `hands` cannot disagree with the
  quality, because it is computed from it.
- **Controllability is the pool composition.** Whether tapping appears, and how
  often, is governed by which triads the pool lists and the ordinary recency
  weighting every axis already uses — the same dial that controls key and
  quality spread. The default pool lists no triads, so existing configurations
  are unchanged and untapped.
- **Only `arpeggios` taps.** No other family routes a triad to a tapped journey;
  `scales`, `intervals`, and `chromatic` have no tapped path in v1 (§9,
  decision 9).

This is a deliberate stopgap. The successor — a demand-driven way to request
"arpeggios, tapping" as a first-class exercise type — is deferred (§12); when it
lands it supersedes this weight rather than extending it.

## 8. Rendering

melete engraves by emitting alphaTex and rendering with the vendored
`melete-render` (alphaTab). This is melete's **only** render path, so whether the
new articulations can travel it is a feasibility question, not a detail. The
emitter's `_note_token` builds an `effects` list (today emitting `acc`/`lf`/`ac`)
and must additionally translate `hand`/`attack` into note effects: a right-hand
tap, a left-hand tap, and a hammer/pull slur — and a right-hand-tapped note needs
*right-hand* fingering, where the emitter uses `lf` (left-hand only) today. Since
tap shapes assign a finger to every note (§4, §5), correct per-hand fingering is
now a first-class requirement, not an afterthought.

**Prerequisite spike — a go/no-go, run first.** alphaTab is a full Guitar Pro
renderer and the corpus files carry `Tapped`, `LeftHandTapped`, and hammer/pull
natively, so the *model* can represent taps; the open question is whether
melete's alphaTex *text* path exposes those effects **and per-hand fingering**.
Settled in-container before the emitter work, with three outcomes:

- **alphaTex expresses them** — proceed as planned; the emitter gains the note
  effects and a right-hand-fingering token.
- **alphaTex cannot, but a lower-level channel can** — emit through it (the corpus
  proves alphaTab renders them natively). This is a **render-path change**, not an
  emitter tweak, and is re-scoped as such.
- **No workaround exists** — articulation is deferred and the epic re-scoped. The
  pure-Python vocabulary and placement work (§4–§7) still stands and is
  independently verifiable, but a tapped *sheet* waits.

The spike is the first task in the plan; it gates the emitter and the end-to-end
deliverable, not the pure-Python work, which is verifiable without a renderer.

## 9. Error Handling

| Failure | Behavior |
|---|---|
| A journey position whose tones cannot partition into two boxable hands | `box`'s two-anchor path raises, naming the pitches and profile; §9's validity gate resamples. Never clamped to fit. |
| A tapped (triad) quality paired with `hands: 1`, or a seventh with `hands: 2` | Impossible by construction — `hands` is derived from the drawn quality, not sampled, so it can never disagree with it (§7). |
| A triad quality listed in a pool whose family has no tapped path (`scales`/`intervals`/`chromatic`) | Rejected by the existing per-family config axis validation: those families read no `quality` axis, so the key is a loud configuration error, never a silent tapped exercise. |
| A requested `(quality, inversion)` has no tap shape | Raise at generation, naming the missing shape. A missing shape is unfinished data, not a case to improvise. |
| The alphaTex path cannot express an articulation or per-hand fingering (spike outcome) | The emitter fails loudly on the unrepresentable note rather than emitting a plausible wrong effect. The workaround, if any, is chosen at spike time. |
| A rhythm accent pattern would fall on a `SLURRED` note | `restamp`'s accent pass consults `attack` and never accents a slur — a hammer-on/pull-off has no attack to accent. This is the one change tapping makes to the shared `rhythm` module. |

No swallowed exceptions and no layout clamped to fit: a mislabelled sheet is worse
than a resampled one, because the label is the part a student trusts.

## 10. Testing Strategy

| Component | Approach |
|---|---|
| `Note` defaults | Unit: a default-constructed note is `(LEFT, PLUCKED)`; existing family tests unchanged. |
| `box` single-hand | Regression: identical output to pre-change for every existing input (the central guard that the one-hand path is untouched). |
| `box` two-anchor | Property: each hand's fretted span ≤ `position_span`; both hands non-empty; every note on a string that sounds its pitch; the supplied partition is honoured. |
| Tap-shape vocabulary | Unit: every (triad × inversion) has a shape; each shape's pitches are exactly the triad's chord tones; each note carries a hand and a 1–4 finger. |
| Tapped-journey placement | Property: the emitted pitch multiset equals the triad's `theory.chord_pitches` tiled across the journey's register; both hands used; no note `PLUCKED`. |
| Legato derivation | Golden: same-string ascending run ⇒ one `TAPPED` then `SLURRED` hammers; descending ⇒ pull-offs; string/hand change ⇒ fresh `TAPPED`. |
| Selection constraint | A `hands: 2` draw never pairs with an unshaped quality; `hands` is recorded in the session log and replays. |
| `restamp` slur | A slurred note is never accented, for any accent pattern. |
| Emitter | Golden alphaTex for one tapped triad arpeggio, byte-compared; default `PLUCKED`/`LEFT` notes emit exactly as before. |
| End to end | One integration test: a `hands: 2` arpeggio config renders to a `.gp` whose `Content/score.gpif` carries the `Tapped` property — the success criterion, checked the way the R&D survey detected tapping. |

**Central invariant.** `pitch == tuning[string] + fret` holds for every note the
driver emits, and the multiset of emitted pitches equals the triad's own chord
tones (`theory.chord_pitches`) tiled across the journey's register — the property
that proves the two-hand layout still sounds exactly the triad it names, in one
octave range. The oracle is the chord tones directly, not a one-hand journey: the
one-hand `arpeggios` family cannot place a triad (`shape_places` has no triad
seed shape — decision 10), so there is no one-hand triad journey to compare
against.

## 11. Recorded Decisions

| # | Decision | Rationale |
|---|---|---|
| 1 | Tapping is a **tabulated tap-shape vocabulary** for `arpeggios`, not a derived cross-cutting modifier. | Player testimony: the two-hand choreography leapfrogs per inversion and is not derivable from a simple rule. Tabulation is the only honest v1, and it is how the instances needed for a future derivation are accumulated (§12). Supersedes v1.0 decisions 1 and 3. |
| 2 | The hand count is bounded at two, not modelled as N. | One or two hands is the entire musical space; "N hands" is speculative generality. `Hand` is a two-valued enum on purpose. |
| 3 | Placement happens at **family generation**, where the inversion is known — not after the fitter. | The choreography is inversion-indexed; a post-fitter modifier would have to reconstruct inversion structure from a flattened, lever-adjusted voice. Reverses v1.0's "run tapping after the fitter." |
| 4 | `hand` and `attack` are two orthogonal `Note` fields. | Hand and attack vary independently — either hand taps, a slur occurs under either hand — matching GPIF's independent note properties. Carried unchanged from v1.0. |
| 5 | The two-hand placement is `box`'s reserved **two-anchor path**, not a new sibling function. | #72 already built `box` with an `anchors` count and reserved `len(anchors) == 2` for this epic (`NotImplementedError` naming #67). Realizing the reserved seam keeps the one-hand path byte-for-byte and the reach math in one place. Supersedes v1.0 decision 5 (`two_hand_boxed` sibling). |
| 6 | The hand partition is **data in the shape**, not a geometric rule; no global fret-ordering invariant holds. | The hands fold across each other and the scale-degree→hand assignment changes per inversion, so "low frets left, high frets right" is false. **Reverses v1.0 decision 6**, which asserted the fret-region rule reproduced every shape. |
| 7 | Legato is derived from same-string (same-hand) adjacency, not tabulated. | Where a hand plays consecutive notes on one string, hammer/pull is the only idiomatic attack; deriving it keeps the shapes to *placement* and off *articulation*. Carried from v1.0 decision 7. |
| 8 | Tapping is selected by **listing tapped triads as `arpeggios` quality candidates**; `hands` is **derived** from the drawn quality, not sampled. | The independent, recency-weighted sampler cannot express a hands↔quality coupling; a shared quality pool plus a derived `hands` makes an unshaped tapped pair unrepresentable without new selector machinery, and controllability comes from pool composition. Supersedes v1.0 decision 8; the freely-sampled `hands`/`range_octaves` axis it named was retired by #72. |
| 9 | Only `arpeggios` is tapping-eligible in v1; `scales`, `intervals`, `chromatic` are not. | Scale tapping is a separate choreography deferred whole; intervals/chromatic have no tapped corpus, and chromatic's subject *is* its left-hand fingering. |
| 10 | v1 tabulates the **four triads only** (`maj`, `min`, `dim`, `aug`); sevenths deferred. | Triads are the fundamental two-hand vocabulary and stay in one octave range. `dim`/`aug` are required for the harmonic-minor-derived material. Sevenths add an inversion and lean stretched. |
| 11 | v1 **preserves pitch** (same octave range); stretched/re-voiced shapes are deferred. | Octave-separating the hands changes which pitches sound — a higher-order complexity. Keeping pitch fixed keeps the central invariant (§10) and the "same exercise, re-placed" framing true. |
| 12 | The renderer spike is a go/no-go run first; it now also settles **per-hand fingering**. | §4–§7 are verifiable without a renderer, but the tapped *sheet* depends on the spike; tap shapes assign fingers, so right-hand fingering is a first-class spike question, not a detail. Carried from v1.0 decision 10, widened. |

## 12. Deferred

Named so the boundary is explicit and the design leaves room for each.

- **Next version.** Scales tapping — its own choreography and shape vocabulary,
  reusing the shared two-hand `box`/legato mechanics this epic builds.
- **Next version.** Seventh-chord tap shapes (`maj7`/`min7`/`dom7`/`m7b5`/`min6`),
  adding a chord tone and a fourth inversion to the tabulated vocabulary.
- **Next version.** Stretched / re-voiced shapes that separate the hands across
  octaves (e.g. 1–5 left, 3–7 an octave up). These **change the sounding
  pitches**, so they relax the central invariant deliberately and need their own
  design.
- **Next version.** A general solution to the multi-position two-hand climb —
  choosing anchors as the shape ascends across many octaves. v1 lays out what the
  tabulated shapes and corpus cover and raises beyond it (§6).
- **Later.** **Deriving** the choreography algorithmically from the accumulated
  tabulated shapes. The tap-shape module is represented as data specifically so a
  derivation generator can replace hand-authored entries additively. The author's
  intent is that, once the corpus is rich enough, the mathematical structure the
  shapes reveal is itself a publishable contribution.
- **Later.** Integrating complex external tapping corpora (e.g. Charles
  Berthoud's exercises) as a stress test for the vocabulary and the eventual
  derivation.
- **Later.** A demand-driven exercise-specification mechanism that lets the user
  request "arpeggios, tapping" as a first-class exercise, superseding the §7
  config weight.
- **Never (as tapping).** Fretting-hand counts above two.

## 13. What changed in the rebase

The v1.0 spec (2026-08-13) was written before epic #72 (the coherent up-and-down
journey) landed. #72 rewrote exactly the seams tapping plugs into. This section
records the deltas so reviewers who read v1.0 can see what moved and why.

| v1.0 assumption | Post-#72 reality | Consequence |
|---|---|---|
| Tapping is a derived `Voice→Voice` modifier wired after the fitter. | The choreography is inversion-indexed and not derivable; families own placement at generation time. | Tapping becomes a tabulated vocabulary + a family driver; placement moves before the fitter (§3, decisions 1, 3). |
| Notes split to hands by a fixed per-string fret region (low→left, high→right), which "reproduces every shape." | The hands leapfrog and fold across each other; the assignment changes per inversion. | The partition is data in each shape; the fret-region rule is deleted (decision 6). |
| A new sibling `_shared.two_hand_boxed`. | `_shared.boxed` was replaced by `box(..., anchors, ...)`, which already reserves the two-anchor case for #67. | Realize `box`'s reserved two-anchor path; no new sibling (decision 5). |
| The `hands` axis is modelled on the `range_octaves` axis, freely sampled. | `range_octaves`/`direction`/`string_set` were retired by #72, and the sampler draws axes independently — it cannot couple `hands` to `quality`. | `hands` is **derived** from the drawn quality (tapped triads are quality candidates), not sampled; there is no drawn `string_set` to "keep the exercise on" (§7, decision 8). |
| The exercise re-places the family's pitches "on the string set the family already drew." | There is no sampled string set; the journey computes whole-instrument coverage. | The tapped journey is the same up-and-down traversal, re-placed two-handed (§3, §6). |
| `pipeline.realize` returns a `Score`. | It returns `(Score, LayoutPlan)`. | Cosmetic for tapping; both call sites already discard the plan (§3). |
| Multi-octave two-hand climb is a rare deferred edge. | Journeys span the whole neck, so multi-position placement is the common case. | The climb is confronted in v1 as far as the shapes/corpus support, raising beyond it (§6). |
| Eligible families are `arpeggios` and `scales`; qualities follow the pool. | — | v1 narrows to `arpeggios` + the four triads, in one octave range (decisions 9, 10, 11). |

---

**Status:** Draft (v2.0 rebase). Filed as epic
[#67](https://github.com/mnemosys-project/.github/issues/67).
