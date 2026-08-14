# Coherent fretboard journeys Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> `superpowers:subagent-driven-development` (recommended) or
> `superpowers:executing-plans` to implement this plan task-by-task. Steps use
> checkbox (`- [ ]`) syntax for tracking.

**Epic:** [`mnemosys-project/.github#72`](https://github.com/mnemosys-project/.github/issues/72)
**Spec:** [`spec.md`](./spec.md)

**Goal:** Replace melete's independently-sampled exercise geometry with computed,
coherent up-and-down journeys — anchored at the root on the lowest string,
traversing outer string to opposite outer string under an explicit fingering
style — and extract the one-hand half of the placement substrate that two-handed
tapping (#67) will rebase onto.

**Architecture:** A new generalized boxing primitive (`_shared.box`, 1 *or* 2
anchors) supersedes the free-search `_shared.boxed`. Each family's `generate`
stops reading the `direction`, `string_set`, and `range_octaves` axes and instead
computes an outer-to-outer journey whose extent, coverage, and octave count fall
out of reaching the opposite outer string. Scales carry two fingering styles
(positional, three-note-per-string); arpeggios carry one canonical seed shape per
quality with inversions/positions derived; chromatic and intervals are made
coherent. The three geometry axes are then removed from config, vocabulary, and
the family axis lists, and the shipping `config.toml` is migrated. Acceptance is
the regenerated five goldens, signed off by the instructor.

**Tech Stack:** Python 3.13+, `uv`, `pytest`; the vendored `melete-render`
(Node + `@coderline/alphatab`) for rendering; validation via
`vrg-container-run -- vrg-validate`. All git via `vrg-git`/`vrg-commit`.

## Global Constraints

Every task's requirements implicitly include these, copied from the spec:

- **Extent is governed by outer-string-to-outer-string; octave count is emergent,
  never a sampled target** (spec §5, decision 11). A four-string yields ≈1.5
  octaves, a six-string ≈2.5.
- **Direction is always up-and-down** (spec §5, decision 1). No family samples or
  reads a `direction` axis after this epic; the single-direction escape hatch is a
  code seam, not a config axis.
- **The one-hand vertical strategy covers the whole instrument** — its string set
  is the full profile string range, outer to outer (spec §4, §5, decision 4).
- **Geometry is computed; musical content stays sampled** (spec §3, decision 2):
  root pitch-class, scale type, quality, inversion, pattern, tempo, and fingering
  style remain sampled.
- **The central invariant `pitch == tuning[string] + fret` holds for every note**
  (spec §11; `instrument.py:9`).
- **Raise, never clamp**: an unplayable placement raises `ValueError` and the
  validity gate (`selection._rejected`) resamples (spec §10).
- **Open strings (fret 0) are valid computed positions, not special-cased**
  (spec decision 5).
- **The two-hand path is a designed seam only**: `box` accepts up to two anchors,
  but N ≥ 2 raises `NotImplementedError` naming #67 — no untested two-hand code
  ships here (spec §9, decision 3).
- **The renderer boundary holds**: no touched module imports `alphatab/emit.py`,
  and no `Note` field is added (spec §2, `CLAUDE.md`).

## Placement Law

Every task below lands its PR in `mnemosys-project/melete`, so every task issue is
filed there. The epic issue and these `spec.md`/`plan.md` documents live in
`mnemosys-project/.github` (published by the documentation task, `.github#73`).

## Human-Gated Preconditions

| Gate | Why |
|---|---|
| PR submission and merge | Standing policy: agents report ready, humans submit. |
| Arpeggio seed-shape validation (Task C1) | The canonical shapes are the instructor's musical knowledge; provisional seeds ship, but the instructor confirms them at Task F1. |
| Golden re-freeze sign-off (Task F1) | Acceptance is the instructor confirming the five regenerated exercises are playable and correct (validation task `melete#152`). |

## The REFACTOR Step

- [ ] **REFACTOR (standing step for every implementation task)**
  - Extract duplicated logic — on the second occurrence, not in anticipation.
  - Move hard-coded values to a shared registry.
  - Consolidate with existing patterns rather than inventing a parallel one.
  - Improve names, then re-run the task's tests to confirm they still pass.

A task is not complete until this step has run and its tests are green afterwards.

## File Structure

| File | Responsibility | Tasks |
|---|---|---|
| `src/melete/families/_shared.py` | Add `box` (generalized 1–2 anchor primitive); later remove `boxed`, `string_set`, `octaves`, `RANGE_OCTAVES` | A1, E2 |
| `src/melete/families/journey.py` *(new)* | The shared one-hand journey: outer-to-outer ascending placement per fingering style, then the up-and-down retrograde | B1 |
| `src/melete/families/arpeggio_shapes.py` *(new)* | Canonical seed shape per chord quality + inversion/position derivation | C1 |
| `src/melete/families/scales.py` | Compute the journey; drop `direction`/`string_set`/`range_octaves` | B2, E2 |
| `src/melete/families/arpeggios.py` | Compute the journey from seed shapes; drop the three axes | C2, E2 |
| `src/melete/families/chromatic.py` | Coherent outer-string-to-outer-string up-and-down traversal; drop `direction` | D1, E2 |
| `src/melete/families/intervals.py` | Same geometry treatment; drop `direction`/`string_set` | D2, E2 |
| `src/melete/selection.py` | Anchor `root` on the lowest instrument string (`_realized`) | E1 |
| `src/melete/config.py`, `src/melete/vocabulary.py` | Remove the three geometry axes and their registry/display entries | E2 |
| `build/config.toml` | Drop the removed keys | E2 |
| `tests/alphatab/golden/…` | Re-frozen five-exercise goldens | F1 |

---

## Phase A — The placement substrate

### Task A1: The generalized boxing primitive `_shared.box`

**Repo:** `mnemosys-project/melete`
**Blocked-by:** —

Adds the one hand-aware placement primitive, pinned to an explicit anchor rather
than free-searching for the lowest-travel base (`boxed`'s search is what let the
Bb land on the A string, defect 2). N = 1 is implemented; N ≥ 2 is the #67 seam.

**Files:**
- Modify: `src/melete/families/_shared.py`
- Test: `tests/families/test_shared_box.py`

**Interfaces:**
- Consumes: `_shared._reachable`, `melete.instrument.hand_span`,
  `melete.instrument.positions`.
- Produces:
  `box(profile, pitches, strings, anchors, family, axes) -> list[tuple[int, int]]`
  — one `(string, fret)` per input pitch, in input order, preserving pitch. Each
  pitch takes the reachable position within `strings` nearest to the single anchor
  base fret; the result must satisfy `profile.position_span` or it raises
  `ValueError`. `anchors` is a tuple; `len(anchors) == 1` is implemented and
  `len(anchors) >= 2` raises `NotImplementedError` naming #67.

- [ ] **Step 1: Write the failing tests**

```python
# tests/families/test_shared_box.py
import pytest
from melete.families._shared import box
from melete.instrument import PROFILES

BASS6 = PROFILES["bass6"]  # tuning B0 E1 A1 D2 G2 C3


def test_box_pins_to_the_anchor_not_the_lowest_travel_base():
    # Bb2=46 sits at fret 11 on the low B string (index 0). Anchored there, the
    # box must place it at (0, 11), not drift to a lower-fret string.
    places = box(BASS6, [46], strings=(0, 1, 2, 3, 4, 5), anchors=(11,),
                 family="scales", axes="root")
    assert places == [(0, 11)]
    assert BASS6.tuning[0] + 11 == 46  # invariant


def test_box_raises_when_wider_than_one_position():
    # Two octaves of a pentatonic across three strings cannot fit one hand.
    with pytest.raises(ValueError, match="scales"):
        box(BASS6, [23, 35, 47], strings=(0, 1, 2), anchors=(0,),
            family="scales", axes="root, scale_type")


def test_two_anchor_box_is_the_67_seam():
    with pytest.raises(NotImplementedError, match="#67"):
        box(BASS6, [23, 35], strings=(0, 1), anchors=(0, 7),
            family="tapping", axes="hands")
```

- [ ] **Step 2: Run and confirm failure**
  Run: `vrg-container-run -- uv run pytest tests/families/test_shared_box.py -v`
  Expected: FAIL — `ImportError: cannot import name 'box'`.

- [ ] **Step 3: Implement the minimum that passes**
  Add to `_shared.py`, modelled on `boxed` (`_shared.py:223-276`) but pinned:

```python
def box(
    profile: InstrumentProfile,
    pitches: Sequence[int],
    strings: tuple[int, ...],
    anchors: tuple[int, ...],
    family: str,
    axes: str,
) -> list[tuple[int, int]]:
    """Place `pitches` under one hand anchored at `anchors[0]` (spec §4).

    Each pitch takes the position within `strings` nearest the anchor base fret,
    ties to the lower fret then the lower string. The result must fit one
    `position_span`, or this raises — the anchor is pinned, so unlike the
    superseded `boxed` it never drifts off the root to minimise travel. Two
    anchors are the #67 two-hand seam and are not realized here.
    """
    if len(anchors) != 1:
        msg = f"box: {len(anchors)} anchors is the two-hand seam owned by #67; this epic lays out one hand"
        raise NotImplementedError(msg)
    base = anchors[0]
    choices = [_reachable(profile, pitch, strings, family, axes) for pitch in pitches]
    places = [min(c, key=lambda p: (abs(p[1] - base), p[0])) for c in choices]
    span = hand_span(fret for _string, fret in places)
    if span > profile.position_span:
        msg = (
            f"{family}: the layout anchored at fret {base} spans {span} frets against a "
            f"position of {profile.position_span} on profile {profile.name!r}: {axes} cannot "
            f"all be satisfied under one hand. §9 resamples this rather than engraving a shift"
        )
        raise ValueError(msg)
    return places
```

- [ ] **Step 4: Run and confirm pass**, plus the existing `_shared` suite (nothing
  else changed): `vrg-container-run -- uv run pytest tests/families/ -q`

- [ ] **Step 5: REFACTOR** (standing step), then commit
  `vrg-commit --type feat --scope families --message "add the generalized box placement primitive (#72)"`

---

## Phase B — The journey builder and the scale styles

### Task B1: The one-hand journey builder

**Repo:** `mnemosys-project/melete`
**Blocked-by:** A1

The shared outer-to-outer journey: given an ascending pitch generator and a
fingering style, place notes from the low anchor to the opposite outer string,
then return the up-and-down order. Extent and octave count are emergent.

**Files:**
- Create: `src/melete/families/journey.py`
- Test: `tests/families/test_journey.py`

**Interfaces:**
- Consumes: `_shared.box`, `_shared.directed_by_cell`,
  `melete.instrument.InstrumentProfile`, `melete.instrument.positions`.
- Produces:
  - `boxed_span(profile, pitches, family, axes) -> tuple[list[int], list[tuple[int,int]]]`
    — take as many leading `pitches` as fit one hand position anchored at the
    lowest string's root fret across the whole instrument, returning the used
    pitches and their `(string, fret)` places, ending on the highest string in the
    box.
  - `per_string(profile, pitches, notes_per_string, family, axes) -> tuple[list[int], list[tuple[int,int]]]`
    — place `notes_per_string` consecutive pitches on each instrument string from
    the lowest, climbing to the highest string; returns the used pitches and
    places. Reaches the top string by construction (the fix for defect 4).
  - `updown(order, cell) -> list[int]` — the up-and-down index order, always
    `directed_by_cell(order, "up_down", cell)` (direction is never sampled).

- [ ] **Step 1: Write the failing tests**

```python
# tests/families/test_journey.py
from melete.families import journey
from melete.instrument import PROFILES
from melete import theory

BASS6 = PROFILES["bass6"]   # 6 strings, tuning B0 E1 A1 D2 G2 C3


def test_per_string_reaches_the_top_string():
    # Enough ascending C major pitches to cover 3 notes on all 6 strings.
    pitches = theory.scale_pitches(root=BASS6.tuning[0] + 3, scale_type="ionian", octaves=4)
    used, places = journey.per_string(BASS6, pitches, notes_per_string=3,
                                      family="scales", axes="root")
    strings = [s for s, _f in places]
    assert strings[0] == 0 and strings[-1] == len(BASS6.tuning) - 1   # outer to outer
    assert sorted(set(strings)) == list(range(len(BASS6.tuning)))     # every string, no gap
    for pitch, (s, f) in zip(used, places):
        assert BASS6.tuning[s] + f == pitch                          # invariant


def test_boxed_span_stays_in_one_position_and_uses_the_outer_strings():
    pitches = theory.scale_pitches(root=BASS6.tuning[0] + 3, scale_type="ionian", octaves=4)
    used, places = journey.boxed_span(BASS6, pitches, family="scales", axes="root")
    from melete.instrument import hand_span
    assert hand_span(f for _s, f in places) <= BASS6.position_span
    assert places[0][0] == 0 and places[-1][0] == len(BASS6.tuning) - 1


def test_updown_is_always_up_and_down():
    # cell 1: ascend 0..3 then back down without replaying the apex.
    assert journey.updown([0, 1, 2, 3], cell=1) == [0, 1, 2, 3, 2, 1, 0]
```

- [ ] **Step 2: Run and confirm failure**
  Run: `vrg-container-run -- uv run pytest tests/families/test_journey.py -v`
  Expected: FAIL — `ModuleNotFoundError: No module named 'melete.families.journey'`.

- [ ] **Step 3: Implement the minimum that passes**

```python
"""The one-hand journey: outer string to opposite outer string, up and down.

Extent is governed by reaching the opposite outer string (spec §5); octave count
is whatever that yields on the instrument. Placement per fingering style lives
here; the families choose a style and their pitch content.
"""
from __future__ import annotations

from typing import TYPE_CHECKING

from melete.families._shared import box, directed_by_cell
from melete.instrument import positions

if TYPE_CHECKING:
    from collections.abc import Sequence
    from melete.instrument import InstrumentProfile


def per_string(profile, pitches, notes_per_string, family, axes):
    """`notes_per_string` consecutive pitches on each string, low string up."""
    used: list[int] = []
    places: list[tuple[int, int]] = []
    for string in range(len(profile.tuning)):
        for pitch in pitches[len(used): len(used) + notes_per_string]:
            fret = pitch - profile.tuning[string]
            if not 0 <= fret <= profile.fret_count:
                msg = (
                    f"{family}: pitch {pitch} needs fret {fret} on string {string} of "
                    f"{profile.name!r} (frets 0..{profile.fret_count}); {axes} cannot all be "
                    f"satisfied. The journey is never truncated to fit (spec §5)"
                )
                raise ValueError(msg)
            used.append(pitch)
            places.append((string, fret))
    return used, places


def boxed_span(profile, pitches, family, axes):
    """The longest leading run of `pitches` that fits one hand across all strings.

    Anchored at the lowest string's first available fret for the opening pitch,
    the box grows pitch by pitch while it still fits one position and still climbs
    toward the top string; extent ends at the highest string in the box.
    """
    strings = tuple(range(len(profile.tuning)))
    base = min(f for s, f in positions(profile, pitches[0]) if s == 0)
    used: list[int] = []
    places: list[tuple[int, int]] = []
    for pitch in pitches:
        try:
            candidate = box(profile, [*used, pitch], strings, (base,), family, axes)
        except ValueError:
            break
        used.append(pitch)
        places = candidate
    return used, places


def updown(order: Sequence[int], cell: int) -> list[int]:
    """The single direction this epic produces: up, then the retrograde (spec §5)."""
    return directed_by_cell(list(order), "up_down", cell)
```

- [ ] **Step 4: Run and confirm pass**
  Run: `vrg-container-run -- uv run pytest tests/families/test_journey.py -v`

- [ ] **Step 5: REFACTOR** (standing step), then commit
  `vrg-commit --type feat --scope families --message "add the one-hand journey builder (#72)"`

### Task B2: Scales compute the outer-to-outer journey

**Repo:** `mnemosys-project/melete`
**Blocked-by:** B1

Rewrite `scales.generate` to compute the journey: full-instrument coverage,
outer-to-outer extent, always up-and-down, the overlapping `pattern` window across
the full ascent. It stops reading `direction`, `string_set`, and `range_octaves`
(the axes are still sampled and arrive in `params`; the family simply no longer
reads them — extra keys are carried, per the existing contract).

**Files:**
- Modify: `src/melete/families/scales.py`
- Test: `tests/families/test_scales.py`

**Interfaces:**
- Consumes: `journey.boxed_span`, `journey.per_string`, `journey.updown`,
  `_shared.windowed`, `theory.scale_pitches`.
- Produces: `scales.generate(profile, params)` unchanged in signature; the voice is
  the up-and-down patterned journey; `AXES` drops `string_set`, `range_octaves`,
  `direction` and keeps `root`, `scale_type`, `traversal` (the fingering style),
  `pattern`. `traversal` realizes `positional` (→ `boxed_span`) and
  `three_note_per_string` (→ `per_string` with 3). `octave_per_string` and
  `single_string` are removed (deferred to the single/two-string modes, spec §13).

- [ ] **Step 1: Write the failing tests**

```python
# tests/families/test_scales.py (add; adjust existing direction/string_set cases)
from melete.families import scales
from melete.instrument import PROFILES

BASS6 = PROFILES["bass6"]


def _params(**over):
    p = {"root": BASS6.tuning[0] + 3, "scale_type": "ionian",
         "traversal": "three_note_per_string", "pattern": "straight"}
    p.update(over)
    return p


def test_scale_journey_is_up_and_down_and_reaches_both_outer_strings():
    score, _hints = scales.generate(BASS6, _params())
    strings = [n.string for n in score.voice]
    assert min(strings) == 0 and max(strings) == len(BASS6.tuning) - 1
    # up-and-down: the string sequence is a palindrome-shaped there-and-back
    assert strings == strings[: len(strings) // 2 + 1] + strings[: len(strings) // 2][::-1] \
        or strings[0] == strings[-1]   # starts and ends on the low string
    for n in score.voice:
        assert BASS6.tuning[n.string] + n.fret == n.pitch


def test_groups_of_4_span_the_full_ascent(monkeypatch=None):
    score, _hints = scales.generate(BASS6, _params(pattern="groups_of_4"))
    # the overlapping window reaches the top string before turning (defect 3)
    ascent = score.voice[: len(score.voice) // 2 + 1]
    assert max(n.string for n in ascent) == len(BASS6.tuning) - 1
```

- [ ] **Step 2: Run and confirm failure**
  Run: `vrg-container-run -- uv run pytest tests/families/test_scales.py -v`
  Expected: FAIL — journey not computed / strings do not reach the top.

- [ ] **Step 3: Implement**
  In `generate`: drop the `string_set`, `octaves`, and `direction` reads; generate
  a long ascending pitch run (`theory.scale_pitches` with enough octaves to cover
  the instrument — e.g. `len(profile.tuning)` octaves is always more than a hand or
  the string count needs), dispatch on the fingering style to `journey.per_string`
  (3) or `journey.boxed_span`, apply `windowed` over the *used* pitches, and order
  with `journey.updown`. Set `AXES = ("root", "scale_type", "traversal", "pattern")`
  and `_TRAVERSALS = (_POSITIONAL, _THREE_NOTE_PER_STRING)`. Remove the
  `_LAYOUT_FALLBACK` one-octave compromise (subsumed by emergent extent, spec §6).
  The `up_down` seam/lever hints stay (they already assume up-and-down).

```python
    read = Parameters(_FAMILY, AXES, params)
    root = read.integer("root")
    scale_type = read.identifier("scale_type")
    traversal = realizable(read, "traversal", _TRAVERSALS)
    pattern = realizable(read, "pattern", tuple(_PATTERN_WINDOWS))

    supply = theory.scale_pitches(root, scale_type, len(profile.tuning) + 1)
    if traversal == _THREE_NOTE_PER_STRING:
        pitches, places = journey.per_string(profile, supply, _NOTES_PER_STRING, _FAMILY, _POSITIONAL_AXES)
    else:  # positional
        pitches, places = journey.boxed_span(profile, supply, _FAMILY, _POSITIONAL_AXES)

    window = _PATTERN_WINDOWS[pattern]
    ascending = windowed(window, len(pitches))
    order = journey.updown(ascending, len(window))
    # ... build voice from pitches[degree]/places[degree] over `order`, title
    #     drops the direction word (always up-and-down), hints as today.
```

- [ ] **Step 4: Run and confirm pass**, plus the full suite. Existing scale tests
  that pin `direction`/`string_set`/`octaves` are updated to the new axis set (they
  are testing removed behaviour). `vrg-container-run -- uv run pytest tests/ -q`

- [ ] **Step 5: REFACTOR** (standing step), then commit
  `vrg-commit --type feat --scope scales --message "compute the outer-to-outer scale journey (#72)"`

---

## Phase C — Arpeggios

### Task C1: Arpeggio seed shapes and derivation

**Repo:** `mnemosys-project/melete`
**Blocked-by:** —
**Human gate:** the seed shapes are provisional until the instructor confirms them
at Task F1 (spec §6, decision 12).

One canonical seed shape per chord quality, plus derivation of inversions and
higher octave positions, replacing `_across`'s "hold a string" rule (defect 4).

**Files:**
- Create: `src/melete/families/arpeggio_shapes.py`
- Test: `tests/families/test_arpeggio_shapes.py`

**Interfaces:**
- Consumes: `theory.chord_pitches`, `melete.instrument.InstrumentProfile`.
- Produces:
  - `SEED_SHAPES: dict[str, tuple[tuple[int, int], ...]]` — per quality, the
    `(string_offset, fret_offset)` of each chord tone relative to the root's
    placement, for one octave in the standard plucking shape. **Provisional —
    instructor-validated at F1.**
  - `shape_places(profile, root_place, quality, tones) -> list[tuple[int,int]]` —
    tile the seed shape from `root_place` up the strings across `tones`
    (derivation), returning `(string, fret)` per tone, preserving pitch, reaching
    the top string when the tone count allows. Raises `ValueError` when a tone
    falls off the neck (never clamped).

- [ ] **Step 1: Write the failing tests**

```python
# tests/families/test_arpeggio_shapes.py
import pytest
from melete.families import arpeggio_shapes as shapes
from melete.instrument import PROFILES
from melete import theory

BASS6 = PROFILES["bass6"]


def test_every_quality_the_pool_drills_has_a_seed():
    for quality in ("maj7", "min7", "dom7", "m7b5", "min6"):
        assert quality in shapes.SEED_SHAPES


def test_min7_shape_preserves_pitch_and_climbs_to_the_top_string():
    root = BASS6.tuning[0] + 10           # a low-string root
    tones = theory.chord_pitches(root, "min7")
    tones = tones + [t + 12 for t in tones] + [tones[0] + 24]  # two octaves + close
    places = shapes.shape_places(BASS6, (0, 10), "min7", tones)
    assert places[0][0] == 0                                   # starts low string
    assert max(s for s, _f in places) == len(BASS6.tuning) - 1  # uses the top string
    for pitch, (s, f) in zip(tones, places):
        assert BASS6.tuning[s] + f == pitch


def test_off_neck_tone_raises():
    with pytest.raises(ValueError, match="arpeggios"):
        shapes.shape_places(BASS6, (5, 23), "maj7", [200])
```

- [ ] **Step 2: Run and confirm failure**
  Run: `vrg-container-run -- uv run pytest tests/families/test_arpeggio_shapes.py -v`

- [ ] **Step 3: Implement**
  Define `SEED_SHAPES` as the standard one-octave plucking shape per quality (root
  and third on adjacent strings, fifth and seventh continuing up — the shapes a
  bassist reads; provisional pending F1), and `shape_places` that walks each tone
  to the next string per the seed's string-offset pattern, deriving higher octaves
  by repeating the shape one octave up the neck, and choosing the fret that sounds
  the tone on that string (raising if none). Keep the derivation a pure function of
  the seed so a corrected seed at F1 needs no code change.

- [ ] **Step 4: Run and confirm pass**, plus `tests/families/`.

- [ ] **Step 5: REFACTOR** (standing step), then commit
  `vrg-commit --type feat --scope arpeggios --message "add arpeggio seed shapes and derivation (#72)"`

### Task C2: Arpeggios compute the journey from seed shapes

**Repo:** `mnemosys-project/melete`
**Blocked-by:** B1, C1

**Files:**
- Modify: `src/melete/families/arpeggios.py`
- Test: `tests/families/test_arpeggios.py`

**Interfaces:**
- Consumes: `arpeggio_shapes.shape_places`, `journey.updown`, `_shared.windowed`,
  `theory.chord_pitches`.
- Produces: `arpeggios.generate` unchanged in signature; the voice is the
  up-and-down seed-shape journey to the top string; `AXES` drops `string_set`,
  `range_octaves`, `direction` and keeps `root`, `quality`, `inversion`,
  `traversal`, `pattern`. `positional` and `across_strings` collapse to the single
  seed-shape layout (positional stays a name if the pool still drills it, but both
  route through `shape_places`); `single_string` is removed (deferred).

- [ ] **Step 1: Write the failing test**

```python
# tests/families/test_arpeggios.py (add; adjust existing direction/string_set cases)
from melete.families import arpeggios
from melete.instrument import PROFILES

BASS6 = PROFILES["bass6"]


def test_arpeggio_journey_uses_all_needed_strings_and_is_up_and_down():
    params = {"root": BASS6.tuning[0] + 10, "quality": "min7", "inversion": "root",
              "traversal": "across_strings", "pattern": "straight"}
    score, _hints = arpeggios.generate(BASS6, params)
    strings = [n.string for n in score.voice]
    assert strings[0] == 0 and max(strings) == len(BASS6.tuning) - 1   # not one-string collapse
    assert strings[0] == strings[-1]                                    # returns to the low string
    for n in score.voice:
        assert BASS6.tuning[n.string] + n.fret == n.pitch
```

- [ ] **Step 2: Run and confirm failure**
  Expected: FAIL — old `_across` collapses onto a single string / reads removed axes.

- [ ] **Step 3: Implement**
  In `generate`: drop the three axis reads; build ascending tones long enough to
  reach the top string, place via `arpeggio_shapes.shape_places` from the root's
  low-string placement, apply `windowed`, order with `journey.updown`. Remove
  `_across`, `_fret`/`_demanded` single-string paths superseded by `shape_places`.
  `AXES = ("root", "quality", "inversion", "traversal", "pattern")`; drop the
  direction word from the title.

- [ ] **Step 4: Run and confirm pass**, plus the full suite (update tests pinning
  removed axes). `vrg-container-run -- uv run pytest tests/ -q`

- [ ] **Step 5: REFACTOR** (standing step), then commit
  `vrg-commit --type feat --scope arpeggios --message "compute the seed-shape arpeggio journey (#72)"`

---

## Phase D — Chromatic and intervals

### Task D1: Chromatic outer-to-outer coherence

**Repo:** `mnemosys-project/melete`
**Blocked-by:** —

Make the chromatic journey coherent: start on an outer string of the string set,
traverse the full set to the opposite outer string, turn around and return; no
mid-neck start, no repeated string, always up-and-down (fixes defect 1). The
four-finger mechanic and the per-cycle `shift` are preserved.

**Files:**
- Modify: `src/melete/families/chromatic.py`
- Test: `tests/families/test_chromatic.py`

**Interfaces:**
- Consumes: `_shared.there_and_back`.
- Produces: `chromatic._strings` traverses `start_string .. start_string ± (span-1)`
  outward and back (always up-and-down), no `direction` argument. `generate` drops
  the `direction` read and `AXES` drops `direction`; `single_string` (`span == 1`)
  stays a single up-and-down string.

- [ ] **Step 1: Write the failing test**

```python
# tests/families/test_chromatic.py (add; adjust existing direction cases)
from melete.families import chromatic
from melete.instrument import PROFILES

BASS6 = PROFILES["bass6"]


def test_chromatic_covers_the_full_span_and_returns():
    params = {"permutation": (3, 1, 4, 2), "start_string": 0, "start_fret": 5,
              "string_traversal": "adjacent", "shift": "none", "span": 6}
    score, _hints = chromatic.generate(BASS6, params)
    strings = sorted({n.string for n in score.voice})
    assert strings == list(range(6))                 # every string, no gap or repeat-only
    first, last = score.voice[0].string, score.voice[-1].string
    assert first == 0 and last == 0                   # outer string out and back
```

- [ ] **Step 2: Run and confirm failure**
  Expected: FAIL — old `_strings` walks a sampled `direction` from a sampled start.

- [ ] **Step 3: Implement**
  Rewrite `_strings(traversal, start_string, span)` to build the ascending walk
  `start_string + cycle * step` and always return `there_and_back(walk)`; delete
  the `direction`-based branch. In `generate`, drop `direction = read.identifier(...)`,
  set the seam/levers to the up-and-down case unconditionally, and remove
  `"direction"` from `AXES`.

- [ ] **Step 4: Run and confirm pass**, plus the full suite.

- [ ] **Step 5: REFACTOR** (standing step), then commit
  `vrg-commit --type feat --scope chromatic --message "outer-to-outer up-and-down chromatic traversal (#72)"`

### Task D2: Intervals geometry treatment

**Repo:** `mnemosys-project/melete`
**Blocked-by:** B1

Apply the same geometry to `intervals`: anchor low, cover the full instrument,
up-and-down, the interval pattern unwound across the full span. Drop `direction`
and `string_set`.

**Files:**
- Modify: `src/melete/families/intervals.py`
- Test: `tests/families/test_intervals.py`

**Interfaces:**
- Consumes: `journey.per_string` / `journey.boxed_span` (whichever the interval
  traversal maps to), `journey.updown`, `_shared.windowed`.
- Produces: `intervals.generate` unchanged in signature; the voice is the
  up-and-down interval journey over the full instrument; `AXES` drops `string_set`
  and `direction`, keeping `interval`, `context`, `root`, `scale_type`,
  `string_skip`, `pattern`.

- [ ] **Step 1: Write the failing test** — assert the interval journey starts on
  the low string, reaches the top string, returns, and preserves the invariant
  (mirror `test_scales.py`'s journey test against `intervals.generate` with a
  minimal valid interval spec). Read `intervals.py` for the exact required params.

- [ ] **Step 2: Run and confirm failure.**

- [ ] **Step 3: Implement** — route interval placement through `journey`, always
  `updown`, drop the `direction`/`string_set` reads and the corresponding `AXES`
  entries; drop the direction word from the title.

- [ ] **Step 4: Run and confirm pass**, plus the full suite.

- [ ] **Step 5: REFACTOR** (standing step), then commit
  `vrg-commit --type feat --scope intervals --message "compute the interval journey (#72)"`

---

## Phase E — Anchor realization, axis removal, config migration

### Task E1: Anchor `root` on the lowest instrument string

**Repo:** `mnemosys-project/melete`
**Blocked-by:** —

`selection._realized` currently places the root relative to the sampled string
set's lowest string. Pin it to the lowest *instrument* string (index 0), in the
lower neck (frets 0–11), which is the anchor the journeys assume (spec §5, §8).

**Files:**
- Modify: `src/melete/selection.py`
- Test: `tests/test_selection.py`

**Interfaces:**
- Produces: `_realized` maps `root` (a pitch class) to `profile.tuning[0] +
  (pitch_class - profile.tuning[0]) % 12` — the lowest fret on the lowest string
  sounding it — independent of any string-set param.

- [ ] **Step 1: Write the failing test**

```python
# tests/test_selection.py (add)
from melete.selection import _realized
from melete.instrument import PROFILES

BASS6 = PROFILES["bass6"]  # lowest string open = B0 = 23


def test_root_anchors_on_the_lowest_instrument_string():
    # pitch class of Bb (10) -> Bb2 = 46 = fret 11 on the low B string.
    out = _realized({"root": 10}, BASS6)
    assert out["root"] == 46
    assert (out["root"] - BASS6.tuning[0]) % 12 == (10 - BASS6.tuning[0]) % 12
```

- [ ] **Step 2: Run and confirm failure**
  Expected: FAIL — `_realized` reads `params[STRING_SET]` (KeyError) or anchors on
  a different string.

- [ ] **Step 3: Implement** — replace the `strings = params[STRING_SET]; open_pitch
  = profile.tuning[strings[0]]` lines with `open_pitch = profile.tuning[0]`, and
  drop the `STRING_SET` reference.

- [ ] **Step 4: Run and confirm pass**, plus the selection suite. (Golden drift is
  expected and re-frozen at F1.)

- [ ] **Step 5: REFACTOR** (standing step), then commit
  `vrg-commit --type feat --scope selection --message "anchor the root on the lowest instrument string (#72)"`

### Task E2: Remove the three geometry axes and migrate the config

**Repo:** `mnemosys-project/melete`
**Blocked-by:** B2, C2, D1, D2, E1 (no family reads the axes; anchor no longer
needs the string set)

Remove `direction`, `string_set`, and `range_octaves` from the sampled surface now
that nothing reads them, and migrate the shipping config so it still loads.

**Files:**
- Modify: `src/melete/config.py` (drop `_DIRECTION`, `_STRING_SET`, `_OCTAVES` from
  `_AXES_BY_FAMILY`; remove the now-unused `_string_set` validator)
- Modify: `src/melete/vocabulary.py` (remove the `"direction"` axis entry — it is no
  longer sampled or displayed; titles no longer state direction)
- Modify: `src/melete/families/_shared.py` (remove `boxed`, `string_set`,
  `octaves`, `RANGE_OCTAVES` — all superseded)
- Modify: `build/config.toml` (drop `directions`, `string_sets`, `string_traversals`
  where it is a scale/arpeggio set, and `octaves` from every pool)
- Test: `tests/test_config.py`, `tests/test_vocabulary.py`

**Interfaces:**
- Produces: a config carrying `directions`, `string_sets`, or `octaves` under any
  pool fails `_reject_unknown` (spec §11); `build/config.toml` loads and
  `melete generate` runs against it.

- [ ] **Step 1: Write the failing tests**

```python
# tests/test_config.py (add)
import pytest
from melete.config import load   # the module's existing loader entry point


def test_direction_key_is_now_unknown(tmp_path):
    with pytest.raises(Exception, match="direction"):
        load(_write_config(tmp_path, "scales", extra='directions = "all"'))


def test_string_set_key_is_now_unknown(tmp_path):
    with pytest.raises(Exception, match="string_set"):
        load(_write_config(tmp_path, "scales", extra="string_sets = [[0, 1, 2]]"))


def test_shipping_config_loads(tmp_path):
    load("build/config.toml")   # the migrated file loads cleanly
```

(`_write_config` follows the existing `tests/test_config.py` fixtures.)

- [ ] **Step 2: Run and confirm failure**
  Run: `vrg-container-run -- uv run pytest tests/test_config.py -k "unknown or shipping" -v`

- [ ] **Step 3: Implement** — remove the three `_Axis` entries from each
  `_AXES_BY_FAMILY` tuple, delete `_DIRECTION`/`_STRING_SET`/`_OCTAVES` axis
  constants and the `_string_set` helper, remove the `"direction"` entry from
  `vocabulary.AXES`, delete `_shared.boxed`/`string_set`/`octaves`/`RANGE_OCTAVES`,
  and edit `build/config.toml` to drop the removed keys from every pool.

- [ ] **Step 4: Run and confirm pass**, plus `vrg-container-run -- vrg-validate`
  (the full suite, with the migrated config).

- [ ] **Step 5: REFACTOR** (standing step), then commit
  `vrg-commit --type refactor --scope config --message "remove the direction, string_set and range_octaves axes (#72)"`

---

## Phase F — Acceptance

### Task F1: Regenerate the five exercises and re-freeze the goldens

**Repo:** `mnemosys-project/melete`
**Blocked-by:** E2
**Operational — validation task `melete#152`.** Not a code PR; run via
`vergil:issue-validate` and record the instructor's sign-off as the SUCCESS comment.

- [ ] **Step 1: Regenerate** the five acceptance exercises from the migrated config
  and render to `.gp` (`vrg-container-run -- vrg-validate` runs generation; render
  via the vendored tool as the existing golden harness, #137, does).
- [ ] **Step 2: Review with the instructor** — confirm each exercise is playable and
  correct: up-and-down, root anchored on the low string, outer-to-outer coverage
  using the top string, correct overlapping grouping, playable fingering (and the
  provisional arpeggio seed shapes, Task C1). Correct any seed shape the instructor
  rejects (a `SEED_SHAPES` data change only) and regenerate.
- [ ] **Step 3: Re-freeze** the goldens under `tests/alphatab/golden/` on sign-off
  (the #137 harness), and record `Outcome: SUCCESS` with the instructor's
  confirmation on `melete#152`.

---

## Task Summary

| # | Task | Repo | Blocked-by |
|---|---|---|---|
| A1 | `_shared.box` generalized primitive | `melete` | — |
| B1 | One-hand journey builder (`journey.py`) | `melete` | A1 |
| B2 | Scales compute the journey | `melete` | B1 |
| C1 | Arpeggio seed shapes + derivation | `melete` | — |
| C2 | Arpeggios compute the journey | `melete` | B1, C1 |
| D1 | Chromatic outer-to-outer coherence | `melete` | — |
| D2 | Intervals geometry | `melete` | B1 |
| E1 | Anchor root on the lowest instrument string | `melete` | — |
| E2 | Remove the three axes + migrate config | `melete` | B2, C2, D1, D2, E1 |
| F1 | Regenerate + re-freeze goldens (validation) | `melete` | E2 |

**Genuinely parallel:** A1, C1, D1, E1 have no blockers. B1→B2 and (B1,C1)→C2 and
B1→D2 are chains that join at E2. **Only looks parallel:** E2 must follow every
family rewrite and E1, because it removes the axes they stopped reading.

## Spec Coverage

| Spec section | Task |
|---|---|
| §4 layout strategy / generalized boxing | A1, B1 |
| §5 the one-hand journey (anchor, outer-to-outer, up-and-down) | B1, B2, E1 |
| §6 fingering styles (scales, arpeggios, chromatic, intervals) | B2, C1, C2, D1, D2 |
| §7 grouping (overlapping window across full ascent) | B2 (via `windowed` + full journey) |
| §8 sampled vs computed; axis removal; config migration | E1, E2 |
| §9 two-hand seam (N≥2 raises) | A1 |
| §10 error handling (raise, never clamp; open strings) | A1, B1, C1 |
| §11 testing & acceptance (goldens, instructor sign-off) | every task's tests; F1 |
| §13 deferred (single/two-string, tapping impl) | not implemented by design |

## Evolution during execution

**Required. Append as the epic runs, not at the end.** One entry per deviation —
what changed and, above all, *why*.

<!-- (no entries yet — implementation has not started) -->
