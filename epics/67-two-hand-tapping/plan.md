# Two-handed tapping Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> `superpowers:subagent-driven-development` (recommended) or
> `superpowers:executing-plans` to implement this plan task-by-task. Steps use
> checkbox (`- [ ]`) syntax for tracking.

**Epic:** [`mnemosys-project/.github#67`](https://github.com/mnemosys-project/.github/issues/67)
**Spec:** [`spec.md`](./spec.md)

**Goal:** Add two-handed tapping to melete as a voice-level articulation
modifier (`tapping.reach`) wired into `pipeline.realize`, so `arpeggios` and
`scales` exercises can be realized as register-split two-hand tapped shapes.

**Architecture:** A new `Voice -> Voice` modifier re-lays-out a family's pitches
across one or two fretting hands (per-string fret region: low frets left, high
frets right) and stamps articulation (`hand`, `attack`), with legato derived
from same-string adjacency. It slots into `pipeline.realize` after the layout
fitter and before `rhythm.restamp`. The phases go data-model first (nothing
compiles against absent `Note` fields), then the pure layout/modifier core
(verifiable without a renderer), then selection/config, then rendering — which
is gated by a go/no-go renderer spike that runs first because a hard failure
there re-scopes the epic.

**Tech Stack:** Python 3.13+, `uv`, `pytest`; the vendored `melete-render`
(Node + `@coderline/alphatab`) for rendering; validation via
`vrg-container-run -- vrg-validate`.

## Global Constraints

Every task's requirements implicitly include these, copied verbatim from the
spec:

- **Hand count is bounded at two, never more** (spec §1, §11 decision 2).
  `Hand` is a two-valued enum.
- **`PLUCKED` and `LEFT` are the defaults; every existing family and golden file
  is unchanged** (spec §2, §4).
- **The central invariant `pitch == tuning[string] + fret` holds for every note
  the modifier emits — tapping moves position but never changes pitch** (spec
  §10).
- **In a two-hand shape every note is `TAPPED` or `SLURRED`, never `PLUCKED`**
  (spec §11 decision 12).
- **Notes split to hands by per-string fret region — low frets → `LEFT`, high
  frets → `RIGHT`** (spec §11 decision 6). Never a pitch-register split.
- **No layout is clamped to fit; an unrealizable two-hand spec raises and §9
  resamples** (spec §9, §11 decision 11).
- **Eligible families are `arpeggios` and `scales` only; a tapping weight on
  `intervals`/`chromatic` is a configuration error** (spec §7, §11 decision 9).
- **The single-hand `_shared.boxed` path stays byte-for-byte identical** (spec
  §6, §10).

## Placement Law

Every task below lands its PR in `mnemosys-project/melete`, so every task issue
is filed there. The epic issue and these `spec.md`/`plan.md` documents live in
`mnemosys-project/.github` (published by the documentation task, `.github#68`).

## Human-Gated Preconditions

| Gate | Why |
|---|---|
| PR submission and merge | Standing policy: agents report ready, humans submit. |

No repository creation or release is required by this epic.

## The REFACTOR Step

- [ ] **REFACTOR (standing step for every implementation task)**
  - Extract duplicated logic — on the second occurrence, not in anticipation.
  - Move hard-coded values to configuration or a shared registry.
  - Consolidate with existing patterns rather than inventing a parallel one.
  - Improve names, then re-run the task's tests to confirm they still pass.

A task is not complete until this step has been performed and its tests are
green afterwards.

---

# Phase A — Data model and renderer feasibility

Delivers the `Note` fields everything else reads and writes, and settles the one
external unknown (whether alphaTex can express taps) before any rendering work
is scheduled.

## Task A1: Renderer feasibility spike (go/no-go)

**Repo:** `mnemosys-project/melete`
**Blocked-by:** —

A spike, not a TDD task: it produces a short findings note, committed to the
epic, that the emitter task (D1) consumes. It is first because a hard failure
re-scopes the epic (spec §8).

**Files:**
- Create: `docs/reports/alphatex-tapping-effects.md`

**Interfaces:**
- Produces: a table of the exact alphaTex note-effect tokens for **right-hand
  tap**, **left-hand tap**, **hammer-on**, **pull-off**, and **right-hand
  fingering** (or the finding that a token does not exist), plus a verdict:
  `expressible in alphaTex` / `needs lower-level channel` / `no workaround`.

- [ ] **Step 1: Emit probe alphaTex and render it in-container**
  Write a throwaway alphaTex string exercising candidate effects on single notes
  (e.g. a tapped note, a left-hand-tapped note, a hammer/pull pair) and render
  it with the vendored tool:
  `vrg-container-run -- node melete-render/render.js < probe.alphatex`
  Inspect whether the produced `.gp` carries the `Tapped` / `LeftHandTapped` /
  `HopoOrigin`/`HopoDestination` note properties (unzip `Content/score.gpif` and
  grep, exactly as the R&D survey did).

- [ ] **Step 2: Cross-check against the vendored alphaTab source**
  Grep the in-container `@coderline/alphatab` dist for the alphaTex effect
  keywords to confirm the tokens the parser accepts (the host has no alphaTab
  install; the container does).

- [ ] **Step 3: Record findings and verdict**
  Write `docs/reports/alphatex-tapping-effects.md` with the token table and the
  verdict. If the verdict is not "expressible", stop and raise it with the human
  — D1 and the end-to-end deliverable are re-scoped per spec §8 before more work
  is scheduled.

- [ ] **Step 4: Commit**
  `vrg-commit --type docs --scope reports --message "record alphaTex tapping-effect feasibility (#67)"`

## Task A2: `Hand` and `Attack` on `Note`

**Repo:** `mnemosys-project/melete`
**Blocked-by:** —

**Files:**
- Modify: `src/melete/score.py`
- Test: `tests/test_score.py`

**Interfaces:**
- Produces: `class Hand(Enum)` with members `LEFT`, `RIGHT`; `class Attack(Enum)`
  with members `TAPPED`, `PLUCKED`, `SLURRED`; two new `Note` fields
  `hand: Hand = Hand.LEFT` and `attack: Attack = Attack.PLUCKED`, appended after
  `tied`. Both exported from `melete.score`.

- [ ] **Step 1: Write the failing test**

```python
# tests/test_score.py
from melete.score import Attack, Hand, Note
from fractions import Fraction


def test_note_defaults_are_left_and_plucked():
    note = Note(pitch=60, string=0, fret=5, duration=Fraction(1, 4),
                finger=None, accent=False)
    assert note.hand is Hand.LEFT
    assert note.attack is Attack.PLUCKED


def test_note_carries_hand_and_attack():
    note = Note(pitch=60, string=0, fret=5, duration=Fraction(1, 4),
                finger=1, accent=False, hand=Hand.RIGHT, attack=Attack.TAPPED)
    assert note.hand is Hand.RIGHT
    assert note.attack is Attack.TAPPED
```

- [ ] **Step 2: Run it and confirm it fails**
  Run: `vrg-container-run -- uv run pytest tests/test_score.py -k "hand_and_attack or defaults_are_left" -v`
  Expected: FAIL — `ImportError: cannot import name 'Hand'`.

- [ ] **Step 3: Implement the minimum that passes**
  Add `from enum import Enum` to the imports, define the enums above `Note`, and
  append the two fields:

```python
class Hand(Enum):
    """Which hand frets a note (spec §4). One or two hands, never more."""

    LEFT = "left"
    RIGHT = "right"


class Attack(Enum):
    """How a note is sounded (spec §4)."""

    TAPPED = "tapped"    # attacked by tapping the fret, either hand
    PLUCKED = "plucked"  # ordinary picked/plucked note — the default
    SLURRED = "slurred"  # hammer-on/pull-off; no fresh attack
```

```python
    # appended to Note, after `tied`:
    hand: Hand = Hand.LEFT
    attack: Attack = Attack.PLUCKED
```

- [ ] **Step 4: Run it and confirm it passes**, plus the full existing suite to
  prove the defaults leave every family unchanged:
  `vrg-container-run -- uv run pytest tests/ -q`
  Expected: PASS, no regressions.

- [ ] **Step 5: REFACTOR** (see the standing step), then commit
  `vrg-commit --type feat --scope score --message "add Hand and Attack to Note (#67)"`

---

# Phase B — Layout and the tapping modifier

The pure core. Verifiable entirely without a renderer.

## Task B1: Two-hand boxing in `_shared`

**Repo:** `mnemosys-project/melete`
**Blocked-by:** A2

**Files:**
- Modify: `src/melete/families/_shared.py`
- Test: `tests/families/test_shared_two_hand.py`

**Interfaces:**
- Consumes: `melete.instrument.positions`, `melete.instrument.hand_span`,
  `_shared._reachable`, `melete.score.Hand`.
- Produces:
  `two_hand_boxed(profile, pitches, strings, family, axes) -> list[tuple[int, int, Hand]]`
  — one `(string, fret, hand)` per input pitch, in input order, preserving
  pitch. Left hand takes the lower-fret region, right hand the higher; each
  hand's fretted span ≤ `profile.position_span`; both hands non-empty. Raises
  `ValueError` (naming pitches, profile, axes) when no two-hand box fits — the
  single-hand `boxed` is untouched.

- [ ] **Step 1: Write the failing test**

```python
# tests/families/test_shared_two_hand.py
from melete.families._shared import two_hand_boxed
from melete.instrument import InstrumentProfile
from melete.score import Hand
import pytest

# 5-string bass B E A D G (spec survey tuning for the tapping études)
BASS5 = InstrumentProfile(name="bass5", tuning=(23, 28, 33, 38, 43), fret_count=24)


def test_low_frets_go_left_high_frets_go_right():
    # A Cmaj7 arpeggio the corpus taps two-handed on two strings.
    # Pitches: C3=36, E3=40, G3=43, B3=47.
    places = two_hand_boxed(BASS5, [36, 40, 43, 47], (1, 2), "arpeggios", "root")
    hands = [h for _s, _f, h in places]
    frets = [f for _s, _f, h in places]
    # every left-hand fret is below every right-hand fret
    left = [f for f, h in zip(frets, hands) if h is Hand.LEFT]
    right = [f for f, h in zip(frets, hands) if h is Hand.RIGHT]
    assert left and right                      # a genuine two-hand split
    assert max(left) < min(right)              # per-string fret region rule
    # pitch preserved: fret == pitch - tuning[string]
    for pitch, (s, f, _h) in zip([36, 40, 43, 47], places):
        assert BASS5.tuning[s] + f == pitch


def test_unboxable_two_hand_spec_raises():
    with pytest.raises(ValueError, match="arpeggios"):
        # a single pitch cannot be a two-hand shape
        two_hand_boxed(BASS5, [36], (1,), "arpeggios", "root")
```

- [ ] **Step 2: Run it and confirm it fails**
  Run: `vrg-container-run -- uv run pytest tests/families/test_shared_two_hand.py -v`
  Expected: FAIL — `ImportError: cannot import name 'two_hand_boxed'`.

- [ ] **Step 3: Implement the minimum that passes**
  Add to `_shared.py` (importing `Hand` from `melete.score`):

```python
def two_hand_boxed(
    profile: InstrumentProfile,
    pitches: Sequence[int],
    strings: tuple[int, ...],
    family: str,
    axes: str,
) -> list[tuple[int, int, Hand]]:
    """Place `pitches` across two fretting hands on `strings` (spec §6).

    Searches anchor pairs (left base < right base): each pitch takes the
    reachable position nearest whichever base is closer, and so is assigned that
    hand. The pair with the lowest total distance wins where both hands fit one
    `position_span`, are non-empty, and no left-hand fret on a string sits above
    a right-hand fret on it. A spec that cannot be boxed under two hands raises —
    the multi-octave climb beyond corpus coverage lands here (spec §11
    decision 11), resampled by §9 rather than forced.
    """
    choices = [_reachable(profile, pitch, strings, family, axes) for pitch in pitches]

    def place(base: int, places: list[tuple[int, int]]) -> tuple[int, int]:
        return min(places, key=lambda p: (abs(p[1] - base), p[0]))

    best: tuple[int, list[tuple[int, int, Hand]]] | None = None
    span = profile.position_span
    for lo in range(profile.fret_count + 1):
        for hi in range(lo + 1, profile.fret_count + 1):
            assigned: list[tuple[int, int, Hand]] = []
            for places in choices:
                lp, rp = place(lo, places), place(hi, places)
                if abs(lp[1] - lo) <= abs(rp[1] - hi):
                    assigned.append((lp[0], lp[1], Hand.LEFT))
                else:
                    assigned.append((rp[0], rp[1], Hand.RIGHT))
            left = [(s, f) for s, f, h in assigned if h is Hand.LEFT]
            right = [(s, f) for s, f, h in assigned if h is Hand.RIGHT]
            if not left or not right:
                continue
            if hand_span(f for _s, f in left) > span:
                continue
            if hand_span(f for _s, f in right) > span:
                continue
            if max(f for _s, f in left) >= min(f for _s, f in right):
                continue
            cost = sum(abs(f - (lo if h is Hand.LEFT else hi)) for _s, f, h in assigned)
            if best is None or cost < best[0]:
                best = (cost, assigned)
    if best is None:
        msg = (
            f"{family}: pitches {list(pitches)} cannot be laid out under two hands on "
            f"strings {list(strings)} of profile {profile.name!r} within a "
            f"{profile.position_span}-fret position each: {axes} cannot all be satisfied. "
            f"§9 resamples this rather than forcing a two-hand fingering"
        )
        raise ValueError(msg)
    return best[1]
```

- [ ] **Step 4: Run it and confirm it passes**, and run the existing `_shared`
  tests to prove `boxed` is untouched:
  `vrg-container-run -- uv run pytest tests/families/ -q`

- [ ] **Step 5: REFACTOR** (see the standing step), then commit
  `vrg-commit --type feat --scope families --message "add two_hand_boxed to _shared (#67)"`

## Task B2: The `tapping.reach` modifier

**Repo:** `mnemosys-project/melete`
**Blocked-by:** B1

**Files:**
- Create: `src/melete/tapping.py`
- Test: `tests/test_tapping.py`

**Interfaces:**
- Consumes: `two_hand_boxed` (B1); `melete.score.Note`, `Hand`, `Attack`,
  `Voice`; `melete.instrument.InstrumentProfile`.
- Produces: `reach(voice, profile, hands) -> Voice`. `hands == 1` is the
  identity. `hands == 2` re-frets every note across two hands on the voice's own
  string set, stamps `hand`/`attack`, and derives legato. Preserves note count,
  order, and the pitch multiset. Raises `ValueError` if any incoming note is
  already articulated — families are tapping-unaware by contract (spec §9).

- [ ] **Step 1: Write the failing test**

```python
# tests/test_tapping.py
from fractions import Fraction
from melete import tapping
from melete.instrument import InstrumentProfile
from melete.score import Attack, Hand, Note

BASS5 = InstrumentProfile(name="bass5", tuning=(23, 28, 33, 38, 43), fret_count=24)


def _note(pitch, string, fret):
    return Note(pitch=pitch, string=string, fret=fret, duration=Fraction(1, 4),
                finger=None, accent=False)


def test_one_hand_is_identity():
    voice = [_note(36, 1, 8), _note(40, 1, 12)]
    assert tapping.reach(voice, BASS5, 1) == voice


def test_two_hand_preserves_pitches_and_splits_hands():
    voice = [_note(36, 1, 8), _note(40, 2, 7), _note(43, 2, 10), _note(47, 3, 9)]
    out = tapping.reach(voice, BASS5, 2)
    assert len(out) == len(voice)
    assert [n.pitch for n in out] == [n.pitch for n in voice]     # multiset+order
    assert {n.hand for n in out} == {Hand.LEFT, Hand.RIGHT}       # both hands used
    assert all(n.attack is not Attack.PLUCKED for n in out)       # decision 12
    for n in out:                                                 # invariant
        assert BASS5.tuning[n.string] + n.fret == n.pitch


def test_articulation_follows_the_same_string_run_rule():
    voice = [_note(36, 1, 8), _note(40, 2, 7), _note(43, 2, 10), _note(47, 3, 9)]
    out = tapping.reach(voice, BASS5, 2)
    assert out[0].attack is Attack.TAPPED           # the first note always attacks
    for prev, cur in zip(out, out[1:]):
        same_run = cur.string == prev.string and cur.hand == prev.hand
        assert (cur.attack is Attack.SLURRED) == same_run


def test_rejects_a_prearticulated_note():
    import pytest
    voice = [Note(pitch=36, string=1, fret=8, duration=Fraction(1, 4), finger=None,
                  accent=False, hand=Hand.RIGHT, attack=Attack.TAPPED)]
    with pytest.raises(ValueError, match="tapping"):
        tapping.reach(voice, BASS5, 2)
```

- [ ] **Step 2: Run it and confirm it fails**
  Run: `vrg-container-run -- uv run pytest tests/test_tapping.py -v`
  Expected: FAIL — `ModuleNotFoundError: No module named 'melete.tapping'`.

- [ ] **Step 3: Write minimal implementation**

```python
"""The tapping modifier — a Voice -> Voice transform over the eligible families.

Tapping is not a family (spec §1); it is a cross-cutting modifier wired into
`pipeline.realize` after the fitter and before `rhythm.restamp`. It takes the
family's pitches, discards their positions, re-lays them out across one or two
fretting hands (spec §6), and stamps articulation (spec §5). Pitch, count and
order are preserved; the pitch multiset before and after is identical.
"""

from __future__ import annotations

from dataclasses import replace
from typing import TYPE_CHECKING

from melete.families._shared import two_hand_boxed
from melete.score import Attack, Hand

if TYPE_CHECKING:
    from melete.instrument import InstrumentProfile
    from melete.score import Voice


def reach(voice: Voice, profile: InstrumentProfile, hands: int) -> Voice:
    """Realize `voice` across `hands` fretting hands (spec §5). Identity at 1."""
    if hands != 2:
        return list(voice)

    notes = list(voice)
    for note in notes:
        if note.hand is not Hand.LEFT or note.attack is not Attack.PLUCKED:
            msg = (
                f"tapping: family emitted an already-articulated note "
                f"({note.hand}, {note.attack}); families are tapping-unaware by contract (§9)"
            )
            raise ValueError(msg)
    strings = tuple(sorted({note.string for note in notes}))
    places = two_hand_boxed(profile, [n.pitch for n in notes], strings,
                            "tapping", "hands and the family's string set")

    out = []
    for index, (note, (string, fret, hand)) in enumerate(zip(notes, places)):
        same_run = (
            index > 0
            and string == out[index - 1].string
            and hand == out[index - 1].hand
        )
        attack = Attack.SLURRED if same_run else Attack.TAPPED
        out.append(replace(note, string=string, fret=fret, hand=hand, attack=attack))
    return out
```

- [ ] **Step 4: Run it and confirm it passes**
  Run: `vrg-container-run -- uv run pytest tests/test_tapping.py -v`

- [ ] **Step 5: REFACTOR** (see the standing step), then commit
  `vrg-commit --type feat --scope tapping --message "add the tapping.reach modifier (#67)"`

## Task B3: Wire `tapping.reach` into `pipeline.realize`

**Repo:** `mnemosys-project/melete`
**Blocked-by:** B2

**Files:**
- Modify: `src/melete/pipeline.py`
- Test: `tests/test_pipeline.py`

**Interfaces:**
- Consumes: `tapping.reach` (B2); the existing `pipeline.realize` stages.
- Produces: `realize` runs `tapping.reach(adjusted, profile, hands)` between
  `layout.plan_voice` and `rhythm.restamp`, reading `hands` from `params` with a
  default of `1`.

- [ ] **Step 1: Write the failing test**

```python
# tests/test_pipeline.py (add)
from melete import pipeline
from melete.instrument import InstrumentProfile
from melete.score import Hand

BASS5 = InstrumentProfile(name="bass5", tuning=(23, 28, 33, 38, 43), fret_count=24)


def test_realize_taps_when_hands_is_two():
    # a maj7 arpeggio spec the arpeggios family can generate; hands=2 taps it.
    params = {"root": 36, "quality": "maj7", "inversion": "root",
              "traversal": "across_strings", "string_set": [1, 2, 3],
              "pattern": "straight", "range_octaves": 1, "direction": "up",
              "hands": 2}
    score, _plan = pipeline.realize(BASS5, "arpeggios", params)
    assert {n.hand for n in score.voice} == {Hand.LEFT, Hand.RIGHT}
```

- [ ] **Step 2: Run it and confirm it fails**
  Run: `vrg-container-run -- uv run pytest tests/test_pipeline.py -k hands_is_two -v`
  Expected: FAIL — all notes still `Hand.LEFT` (tapping not wired in).

- [ ] **Step 3: Write minimal implementation**
  In `pipeline.realize`, import `tapping` and insert the stage between the fitter
  and the restamp:

```python
from melete import layout, rhythm, tapping   # add tapping
...
    score, hints = REGISTRY[family].generate(profile, params)
    adjusted, plan = layout.plan_voice(score.voice, hints)
    tapped = tapping.reach(adjusted, profile, cast("int", params.get("hands", 1)))
    voice = rhythm.restamp(
        tapped,
        plan.subdivision,
        note_value_pattern=cast("str", params.get("note_value_pattern", "straight")),
        accent_pattern=cast("str", params.get("accent_pattern", "none")),
    )
```

- [ ] **Step 4: Run it and confirm it passes**, plus the full suite (single-hand
  draws must be unchanged): `vrg-container-run -- uv run pytest tests/ -q`

- [ ] **Step 5: REFACTOR** (see the standing step), then commit
  `vrg-commit --type feat --scope pipeline --message "run tapping.reach between the fitter and restamp (#67)"`

---

# Phase C — Selection, configuration, and the rhythm interaction

## Task C1: The `hands` axis in configuration (eligibility + opt-in default)

**Repo:** `mnemosys-project/melete`
**Blocked-by:** —

**Files:**
- Modify: `src/melete/config.py`
- Test: `tests/test_config.py`

**Interfaces:**
- Produces: a `hands` integer axis (universe `(1, 2)`) added to the config axis
  list for `arpeggios` and `scales` only, following the existing `range_octaves`
  integer-axis precedent (`_AXES_BY_FAMILY`, the integer `_Axis` element).
  Unconfigured, `hands` defaults to `(1,)` for those families — the one axis that
  defaults, because tapping is opt-in (spec §7). `intervals`/`chromatic` do not
  list it, so `[pool.<them>] hands = …` fails `_reject_unknown`.

- [ ] **Step 1: Write the failing tests**

```python
# tests/test_config.py (add)
import pytest
from melete.config import load  # or the module's existing entry point


def test_hands_defaults_to_off_for_arpeggios(tmp_path):
    cfg = load(_write_min_config(tmp_path, family="arpeggios"))  # no hands key
    assert cfg.pool["arpeggios"].values["hands"] == (1,)


def test_hands_configurable_for_scales(tmp_path):
    cfg = load(_write_min_config(tmp_path, family="scales", extra="hands = [1, 2, 2]"))
    assert cfg.pool["scales"].values["hands"] == (1, 2, 2)


def test_hands_rejected_for_chromatic(tmp_path):
    with pytest.raises(Exception, match="hands"):
        load(_write_min_config(tmp_path, family="chromatic", extra="hands = [2]"))
```

(`_write_min_config` is a helper this test file adds, writing a minimal valid
`config.toml` with the given `[pool.<family>]` section and `extra` lines. Follow
the existing config-test fixtures in `tests/test_config.py` for the minimal
required keys.)

- [ ] **Step 2: Run and confirm failure**
  Run: `vrg-container-run -- uv run pytest tests/test_config.py -k hands -v`
  Expected: FAIL — `hands` not a known axis / KeyError.

- [ ] **Step 3: Implement**
  Add a `hands` `_Axis` (integer element, universe `(1, 2)`) modelled on the
  `range_octaves` axis, register it in `_AXES_BY_FAMILY` for `arpeggios` and
  `scales` only, and in `_pool` default it to `(1,)` when absent for those
  families (the single deliberate default; every other axis stays absent when
  unconfigured). Do **not** add it to `intervals`/`chromatic`, so their
  `_reject_unknown` continues to reject it.

- [ ] **Step 4: Run and confirm pass**, plus the full config suite.

- [ ] **Step 5: REFACTOR** (see the standing step), then commit
  `vrg-commit --type feat --scope config --message "add the opt-in hands axis for arpeggios and scales (#67)"`

## Task C2: Draw `hands` in selection

**Repo:** `mnemosys-project/melete`
**Blocked-by:** C1
**Depends on family axis declaration:** adds `"hands"` to `arpeggios.AXES` and
`scales.AXES`.

**Files:**
- Modify: `src/melete/families/arpeggios.py` (append `"hands"` to `AXES`)
- Modify: `src/melete/families/scales.py` (append `"hands"` to `AXES`)
- Test: `tests/test_selection.py`

**Interfaces:**
- Consumes: `config.pool[family].values["hands"]` (C1); the existing
  `_sample`/`_candidates` machinery.
- Produces: every sampled `arpeggios`/`scales` spec carries a `hands` value in
  `params`, drawn from the configured candidates; it is recorded in the session
  log like any other axis (spec §7). The families' `generate` ignore it (extra
  keys are carried, not read).

- [ ] **Step 1: Write the failing test**

```python
# tests/test_selection.py (add)
def test_sampled_arpeggio_spec_carries_hands(min_config_hands_off):
    # min_config_hands_off: a config whose [pool.arpeggios] hands = [1]
    spec = _sample_one("arpeggios", min_config_hands_off)
    assert spec.params["hands"] == 1
```

- [ ] **Step 2: Run and confirm failure**
  Expected: FAIL — `KeyError: 'hands'` in params.

- [ ] **Step 3: Implement**
  Append `"hands"` to the `AXES` tuple in `arpeggios.py` and `scales.py`. Because
  `hands` is now in `REGISTRY[family].axes`, `_sample`'s family-axis loop draws
  it from `config.pool[family].values["hands"]` (always present after C1). No
  selection code changes are required; the draw rides the existing loop.

- [ ] **Step 4: Run and confirm pass**, plus the full suite — confirm existing
  arpeggio/scale golden and generation tests still pass with `hands` defaulting
  to `1` (identity in `tapping.reach`).

- [ ] **Step 5: REFACTOR** (see the standing step), then commit
  `vrg-commit --type feat --scope selection --message "draw the hands axis for the eligible families (#67)"`

## Task C3: `restamp` never accents a slurred note

**Repo:** `mnemosys-project/melete`
**Blocked-by:** A2

**Files:**
- Modify: `src/melete/rhythm.py`
- Test: `tests/test_rhythm.py`

**Interfaces:**
- Consumes: `melete.score.Attack` (A2).
- Produces: `restamp` clears any accent it would otherwise stamp on a
  `SLURRED` note — an accent marks an attack, a slur has none (spec §9).

- [ ] **Step 1: Write the failing test**

```python
# tests/test_rhythm.py (add)
from fractions import Fraction
from melete import rhythm
from melete.score import Attack, Note


def test_slurred_notes_are_never_accented():
    voice = [
        Note(pitch=60, string=0, fret=5, duration=Fraction(1, 4), finger=None,
             accent=False, attack=Attack.TAPPED),
        Note(pitch=62, string=0, fret=7, duration=Fraction(1, 4), finger=None,
             accent=False, attack=Attack.SLURRED),
    ]
    out = rhythm.restamp(voice, "eighth", accent_pattern="every_3")
    slurred = [n for n in _flatten(out) if n.attack is Attack.SLURRED]
    assert all(not n.accent for n in slurred)
```

(Use the module's own `_flatten` or iterate the returned voice; a period of `3`
with offset `0` would otherwise accent index 0 and, across a longer voice, land
on a slur.)

- [ ] **Step 2: Run and confirm failure**
  Expected: FAIL — a slurred note carries an accent.

- [ ] **Step 3: Implement**
  In `restamp`, after computing `accents = _accents(len(notes), ACCENTS[accent_pattern])`,
  mask out slurred notes so a slur is never accented:

```python
    accents = _accents(len(notes), ACCENTS[accent_pattern])
    accents = [flag and note.attack is not Attack.SLURRED
               for note, flag in zip(notes, accents)]
```

This masks the accent off for any slurred note before the `replace(note,
duration=…, accent=…)` site (`rhythm.py:174-177`) consumes `accents`. Import
`Attack` from `melete.score`.

- [ ] **Step 4: Run and confirm pass**, plus the full rhythm suite.

- [ ] **Step 5: REFACTOR** (see the standing step), then commit
  `vrg-commit --type feat --scope rhythm --message "never accent a slurred note (#67)"`

---

# Phase D — Rendering and integration

## Task D1: Emit `hand`/`attack` as alphaTex note effects

**Repo:** `mnemosys-project/melete`
**Blocked-by:** A1 (tokens), A2 (fields)

**Files:**
- Modify: `src/melete/alphatab/emit.py`
- Test: `tests/alphatab/test_emit_tapping.py`

**Interfaces:**
- Consumes: the confirmed token table from A1; `melete.score.Hand`, `Attack`.
- Produces: `_note_token` appends, into its `effects` list, the tap effect for a
  `TAPPED` note (right- vs left-hand per `note.hand`), the hammer/pull effect for
  a `SLURRED` note, and right-hand fingering for a right-hand-tapped note (in
  place of `lf`). Token strings are module constants set from A1's findings.

- [ ] **Step 1: Write the failing test**

```python
# tests/alphatab/test_emit_tapping.py
from fractions import Fraction
from melete.alphatab.emit import _note_token, _TieState
from melete.score import Attack, Hand, Note


def _token(hand, attack):
    note = Note(pitch=45, string=1, fret=12, duration=Fraction(1, 4), finger=None,
                accent=False, hand=hand, attack=attack)
    return _note_token(note, string_count=5, key=None, signature={},
                       tie=_TieState(), ratio=None)


def test_right_hand_tap_carries_the_tap_effect():
    assert _RH_TAP_TOKEN in _token(Hand.RIGHT, Attack.TAPPED)   # constant from A1


def test_slur_carries_the_hammer_pull_effect():
    assert _SLUR_TOKEN in _token(Hand.LEFT, Attack.SLURRED)     # constant from A1
```

(Import the effect-token constants the implementation defines; their literal
values come from Task A1's findings note.)

- [ ] **Step 2: Run and confirm failure**
  Expected: FAIL — no tap/slur token in the emitted beat.

- [ ] **Step 3: Implement**
  Add the effect appends to `_note_token`'s `effects` block (after the existing
  `acc`/`lf`/`ac` lines), using module constants whose values are the tokens A1
  confirmed:

```python
    if note.attack is Attack.SLURRED:
        effects.append(_SLUR_TOKEN)
    elif note.attack is Attack.TAPPED and note.hand is Hand.RIGHT:
        effects.append(_RH_TAP_TOKEN)
    elif note.attack is Attack.TAPPED and note.hand is Hand.LEFT:
        effects.append(_LH_TAP_TOKEN)
```

  For a right-hand note, emit right-hand fingering instead of `lf` (guard the
  existing `lf` line with `note.hand is Hand.LEFT`, and add the `rf` branch if A1
  confirmed a right-hand-fingering token). If A1's verdict was "needs lower-level
  channel", implement that channel here instead — the task's deliverable is the
  articulation reaching the `.gp`, by whatever route A1 established.

- [ ] **Step 4: Run and confirm pass**, plus the full emit suite (default
  `PLUCKED`/`LEFT` notes must emit exactly as before — the golden alphaTex tests
  are the guard).

- [ ] **Step 5: REFACTOR** (see the standing step), then commit
  `vrg-commit --type feat --scope alphatab --message "emit hand/attack as alphaTex effects (#67)"`

## Task D2: End-to-end tapped-sheet integration test

**Repo:** `mnemosys-project/melete`
**Blocked-by:** B3, C2, C3, D1

**Files:**
- Test: `tests/test_tapping_end_to_end.py`

**Interfaces:**
- Consumes: the whole pipeline through the renderer.
- Produces: a black-box test that a `hands: 2` arpeggio config generates and
  renders to a valid `.gp` whose `Content/score.gpif` carries the `Tapped`
  property — the spec's success criterion, checked the way the R&D survey
  detected tapping.

- [ ] **Step 1: Write the failing/So-far-unwritten test**

```python
# tests/test_tapping_end_to_end.py
import zipfile
from melete import cli  # or the generate entry point the existing e2e test uses


def test_two_hand_arpeggio_renders_a_tapped_gp(tmp_path):
    # Follow tests/alphatab/…::the existing real-alphaTex-to-.gp integration test
    # for how it drives generate+render into a tmp dir. Configure
    # [pool.arpeggios] hands = [2] so every draw taps.
    gp_path = _generate_one_tapped_arpeggio(tmp_path)     # helper per existing e2e
    with zipfile.ZipFile(gp_path) as z:
        gpif = z.read("Content/score.gpif").decode("utf-8", "replace")
    assert 'name="Tapped"' in gpif
```

- [ ] **Step 2: Run and confirm it fails** (or is red until D1 lands)
  Run: `vrg-container-run -- uv run pytest tests/test_tapping_end_to_end.py -v`

- [ ] **Step 3: Make it pass** by ensuring the config fixture and the generate
  path are wired; no new production code beyond Phases B–D should be needed.

- [ ] **Step 4: Run the whole suite** — `vrg-container-run -- vrg-validate`.

- [ ] **Step 5: REFACTOR** (see the standing step), then commit
  `vrg-commit --type test --scope tapping --message "end-to-end tapped .gp integration test (#67)"`

---

# Task Summary

| # | Task | Repo | Blocked-by |
|---|---|---|---|
| A1 | Renderer feasibility spike (go/no-go) | `melete` | — |
| A2 | `Hand`/`Attack` on `Note` | `melete` | — |
| B1 | `two_hand_boxed` in `_shared` | `melete` | A2 |
| B2 | `tapping.reach` modifier | `melete` | B1 |
| B3 | Wire `tapping.reach` into `pipeline.realize` | `melete` | B2 |
| C1 | `hands` axis in config (eligibility + opt-in default) | `melete` | — |
| C2 | Draw `hands` in selection | `melete` | C1 |
| C3 | `restamp` skips accents on slurs | `melete` | A2 |
| D1 | Emit `hand`/`attack` as alphaTex effects | `melete` | A1, A2 |
| D2 | End-to-end tapped-sheet test | `melete` | B3, C2, C3, D1 |

**Genuinely parallel:** A1, A2, and C1 have no blockers and can run at once.
B1→B2→B3 is a strict chain. C3 needs only A2. D1 needs A1+A2. D2 is the join of
everything. **Only looks parallel:** C2 reads config from C1 and must follow it.

## Spec Coverage

| Spec section | Task |
|---|---|
| §1 Overview / reframe | B2, B3 (the modifier and its placement) |
| §2 Scope (in-scope deliverables) | A2, B1, B2, B3, C1, C2, C3, D1 |
| §3 Architecture (pipeline placement) | B3 |
| §4 Data model (`hand`, `attack`, `finger`) | A2; `finger`-per-hand rendered in D1 |
| §5 The tapping modifier (partition/box/articulate/legato) | B2 (using B1) |
| §6 Layout: one or two hands | B1 |
| §7 Selection and configuration | C1, C2 |
| §8 Rendering / spike | A1, D1 |
| §9 Error Handling | B1 (unboxable raises), C1 (eligibility), C3 (slur accents), A1/D1 (renderer) |
| §10 Testing Strategy | every task's tests; central invariant asserted in B2 |
| §11 Recorded Decisions | enforced across tasks via Global Constraints |
| §12 Deferred | not implemented by design (multi-octave climb raises in B1) |

## Evolution during execution

**Required. Append as the epic runs, not at the end.** One entry per deviation —
what changed and, above all, *why*.

<!-- (no entries yet — implementation has not started) -->
