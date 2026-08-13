# Even-measure exercise layout — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Every generated exercise engraves as a whole number of complete
measures (never a partial/red bar), wrapped in repeats, with a simple
low-denominator time signature derived from the pattern.

**Architecture:** A new renderer-agnostic **layout fitter** (`src/melete/layout.py`)
consumes a family's `(voice, LayoutHints)` and derives `(subdivision, time
signature, bar count)` plus any note-count adjustments and a legibility trace,
guaranteeing the voice tiles into whole bars. It replaces the *sampling* of meter
and subdivision in `rhythm.py`. Repeat barlines become a first-class `Score`
property emitted by `alphatab/emit.py`. Families gain full-cycle rules (chromatic
all-strings; scales two-octaves-from-lowest on bass6; `up_down` default) and emit
`LayoutHints`.

**Tech Stack:** Python 3.12+, `dataclasses`, `fractions.Fraction`, pytest. The
vendored `melete-render/` Node tool (alphaTab → `.gp`) is unchanged except for
verifying repeat-token syntax.

**Spec:** `epics/57-even-measure-layout/spec.md` (this epic). The plan argues from
the spec; executors read both.

## Global Constraints

- **Renderer boundary (CLAUDE.md, `docs/design.md`):** all fitter/layout logic is
  renderer-agnostic and lives outside `src/melete/alphatab/`. `layout.py` must not
  import `alphatab`. Only repeat-token emission touches `emit.py`.
- **Written durations only (`score.py` decision #16):** `Note.duration` is the
  engraved value; tuplet scaling lives in `Tuplet.ratio`; sounding time comes only
  from `score.sounding_duration`. Never store sounding time.
- **No silent failures (user global policy):** a pattern that cannot be fit, or a
  scale that cannot span two octaves, raises or records a declared fallback in the
  trace — never a silently wrong sheet.
- **Instrument:** target `bass6` — tuning `(23, 28, 33, 38, 43, 48)` = B E A D G C,
  `len(profile.tuning) == 6` is the string count, `fret_count == 24`.
- **Meter denominator stays `4`** in the common scheme (a beat is a quarter; the
  subdivision absorbs density). `/8` only for a deliberate compound feel; **never
  `/16` or finer**.
- **Git/tooling:** use `vrg-git`, `vrg-commit --type … --scope … --message …`,
  and validate only with `vrg-container-run -- vrg-validate`. Work in the
  per-issue worktree; the main worktree is read-only.
- **Sane beats-per-bar whitelist:** `b ∈ {2, 3, 4, 6}` is a hard filter (spec
  §4.5); it outranks the even-bar-count preference, and the note-count lever fires
  before either is sacrificed.

---

## File structure

| File | Responsibility | Tasks |
|---|---|---|
| `src/melete/layout.py` *(new)* | `LayoutHints`, `Lever`, `LayoutPlan`, the fitter `fit()`, tiling + ladder + levers + trace. Renderer-agnostic core. | 1, 3, 4 |
| `src/melete/score.py` | Add the exercise-level `repeat` property to `Score`. | 2 |
| `src/melete/alphatab/emit.py` | Emit repeat-barline tokens. | 2 |
| `src/melete/families/__init__.py` | Widen `Generate` type + `REGISTRY` to `(Score, LayoutHints)`. | 5 |
| `src/melete/families/chromatic.py` | All-strings cycle; apex-repeat; emit `LayoutHints`. | 6 |
| `src/melete/families/scales.py` | Two-octaves-from-lowest; `up_down` default; conditional fallback; emit `LayoutHints`. | 7 |
| `src/melete/families/{arpeggios,intervals}.py` | Emit minimal `LayoutHints`; widen return. | 5 |
| `src/melete/families/_shared.py` | Shared `layout_hints(...)` helper if useful. | 5 |
| `src/melete/rhythm.py` | Stop sampling meter/subdivision; accept them from the fitter; keep accent/note-value overlays. | 8 |
| `src/melete/cli.py` | Pipeline: call `generate → fit → restamp`, attach repeat + trace. | 8 |
| `src/melete/config.py` | Drop `time_signatures`/`subdivisions` from `_RHYTHM_AXES`. | 9 |
| `src/melete/vocabulary.py` | Remove the now-unsampled `time_signature`/`subdivision` axis entries (if no longer read). | 9 |
| `examples/config.toml` | Remove the two rhythm keys. | 9 |
| `MEMORY.md` *(new)* | Repo memory: policy header + build-output convention. | 10 |
| `tests/test_layout.py` *(new)* | Fitter unit + property tests. | 1, 3, 4 |
| `tests/alphatab/golden/*.atex` | Repeat + regenerated-exercise goldens. | 2, 11 |

---

## Task 1: `LayoutHints` data contract and apex realization

**Files:**
- Create: `src/melete/layout.py`
- Test: `tests/test_layout.py`

**Interfaces:**
- Produces: `Lever` (`enum.Enum` with members `APEX_REPEAT`, `APEX_OMIT`,
  `ADD_ONE`, `DROP_ONE`); `LayoutHints(cell: int, seam: int | None, levers:
  tuple[Lever, ...])`; `realize_lever(voice: Voice, lever: Lever, seam: int |
  None) -> Voice`.

- [ ] **Step 1: Write the failing test**

```python
# tests/test_layout.py
from fractions import Fraction
import pytest
from melete.layout import Lever, LayoutHints, realize_lever
from melete.score import Note

def _n(pitch):  # a bare note; only identity/count matters for lever tests
    return Note(pitch=pitch, string=0, fret=pitch - 23, duration=Fraction(1, 4),
                finger=None, accent=False)

def test_layout_hints_defaults_and_validation():
    h = LayoutHints(cell=4, seam=24, levers=(Lever.APEX_REPEAT,))
    assert h.cell == 4 and h.seam == 24 and Lever.APEX_REPEAT in h.levers
    with pytest.raises(ValueError):
        LayoutHints(cell=0, seam=None, levers=())      # cell must be >= 1

def test_apex_repeat_duplicates_the_seam_note():
    voice = [_n(23), _n(24), _n(25)]                   # seam at the apex index 2
    out = realize_lever(voice, Lever.APEX_REPEAT, seam=2)
    assert [n.pitch for n in out] == [23, 24, 25, 25]  # apex played twice

def test_drop_one_removes_the_last_note():
    voice = [_n(23), _n(24), _n(25)]
    out = realize_lever(voice, Lever.DROP_ONE, seam=None)
    assert [n.pitch for n in out] == [23, 24]
```

- [ ] **Step 2: Run test to verify it fails**

Run: `vrg-container-run -- python -m pytest tests/test_layout.py -q`
Expected: FAIL with `ModuleNotFoundError: melete.layout`.

- [ ] **Step 3: Write minimal implementation**

```python
# src/melete/layout.py
"""The layout fitter (spec §4): derive a legible, whole-bar engraving.

Renderer-agnostic (must not import ``alphatab``). A family emits a ``Score`` plus
``LayoutHints``; ``fit`` turns the note count and hints into a ``LayoutPlan`` —
subdivision, time signature, bar count, any note-count levers applied, and a
human-readable trace — such that the voice tiles into whole measures.
"""
from __future__ import annotations

import enum
from dataclasses import dataclass
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from melete.score import Voice


class Lever(enum.Enum):
    """A musically-legal note-count adjustment the fitter may apply (spec §4.6)."""
    APEX_REPEAT = "apex_repeat"   # play the turnaround note/cell twice
    APEX_OMIT = "apex_omit"       # play the turnaround note once
    ADD_ONE = "add_one"           # repeat the final note
    DROP_ONE = "drop_one"         # trim the final note


@dataclass(frozen=True)
class LayoutHints:
    """What the fitter needs from a family that the voice alone does not carry.

    ``cell`` is the natural group size ``g`` (spec §4.3): the run of notes that
    forms one beat. ``seam`` is the note index of the musical turnaround (an
    up/down pattern's apex), or ``None``. ``levers`` are the adjustments legal for
    this pattern.
    """
    cell: int
    seam: int | None
    levers: tuple[Lever, ...]

    def __post_init__(self) -> None:
        if self.cell < 1:
            msg = f"cell (group size g) must be at least 1, got {self.cell}"
            raise ValueError(msg)


def realize_lever(voice: Voice, lever: Lever, seam: int | None, cell: int = 1) -> Voice:
    """Apply ``lever`` to ``voice``, returning a new list (spec §4.6).

    Apex levers act on a whole *cell* — the ``cell`` notes ending at ``seam`` —
    so repeating the apex of a 4-note chromatic group adds four notes, while a
    single-note scale apex (``cell=1``) adds one. ``ADD_ONE``/``DROP_ONE`` always
    act on a single trailing note.
    """
    if lever is Lever.APEX_REPEAT:
        if seam is None:
            msg = "APEX_REPEAT needs a seam index; none was supplied"
            raise ValueError(msg)
        apex = voice[seam - cell + 1 : seam + 1]
        return [*voice[: seam + 1], *apex, *voice[seam + 1 :]]
    if lever is Lever.APEX_OMIT:
        if seam is None:
            msg = "APEX_OMIT needs a seam index; none was supplied"
            raise ValueError(msg)
        return [*voice[: seam - cell + 1], *voice[seam + 1 :]]
    if lever is Lever.ADD_ONE:
        return [*voice, voice[-1]]
    if lever is Lever.DROP_ONE:
        return list(voice[:-1])
    msg = f"unknown lever {lever!r}"          # no silent fallthrough
    raise ValueError(msg)
```

- [ ] **Step 4: Run test to verify it passes**

Run: `vrg-container-run -- python -m pytest tests/test_layout.py -q`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
vrg-git add src/melete/layout.py tests/test_layout.py
vrg-commit --type feat --scope layout --message "add LayoutHints and note-count lever realization (#57)"
```

---

## Task 2: Repeat-barline support (Score model + emitter)

**Files:**
- Modify: `src/melete/score.py` (the `Score` dataclass, ~line 246-253)
- Modify: `src/melete/alphatab/emit.py` (`_exercise_directives`, `_exercise_bars`)
- Test: `tests/test_score.py`, `tests/alphatab/test_alphatex_emit.py`
- Create: `tests/alphatab/golden/repeat.atex`

**Interfaces:**
- Consumes: nothing from Task 1.
- Produces: `Score.repeat: bool` (default `False`); when `True`, `emit_score`
  brackets the exercise's bars with alphaTab repeat tokens.

- [ ] **Step 1: Spike — confirm the alphaTex repeat tokens**

alphaTab's alphaTex uses bar-level repeat markers. Confirm the exact tokens
against the vendored renderer before coding:

Run: `vrg-container-run -- node melete-render/render.js --help` (or inspect
`melete-render/render.js` and the bundled alphaTab) and render a tiny probe:

```
\ts 4 4 \clef bass \tempo 60 \ro 0.6.4 0.6.4 0.6.4 0.6.4 | 0.6.4 0.6.4 0.6.4 0.6.4 \rc 2
```

Expected: a two-bar phrase marked to repeat twice, loadable as `.gp`. Record the
confirmed opener/closer tokens (expected: `\ro` on the first bar, `\rc 2` on the
last). **If the tokens differ, use the confirmed ones in Step 4 and note them in
the commit body.** Do not proceed on assumption.

- [ ] **Step 2: Write the failing test**

```python
# tests/test_score.py  (add)
def test_score_repeat_defaults_false_and_accepts_true():
    from tests.alphatab_helpers import minimal_score  # or build inline as other tests do
    s = minimal_score()
    assert s.repeat is False
    assert s.__class__(**{**s.__dict__, "repeat": True}).repeat is True
```

```python
# tests/alphatab/test_alphatex_emit.py  (add)
def test_repeat_tokens_bracket_the_exercise():
    score = _exercise_score(repeat=True)               # local builder, repeat=True
    text = emit.emit_score(score)
    assert "\\ro" in text                              # opener on the first bar
    assert "\\rc 2" in text                            # close, two passes
    # opener precedes the first note token; closer is at the very end
    assert text.index("\\ro") < text.index("0.")
    assert text.rstrip().endswith("\\rc 2") or "\\rc 2" in text.split("|")[-1]
```

*(Match the existing emit tests' Score-construction style in
`tests/alphatab/test_alphatex_emit.py`; add a `repeat=` argument to the local
builder.)*

- [ ] **Step 3: Run tests to verify they fail**

Run: `vrg-container-run -- python -m pytest tests/test_score.py tests/alphatab/test_alphatex_emit.py -q`
Expected: FAIL — `Score` has no `repeat`; no `\ro`/`\rc` emitted.

- [ ] **Step 4: Implement**

In `src/melete/score.py`, add the field to `Score` (after `params`):

```python
    params: dict[str, object] = field(default_factory=dict)
    repeat: bool = False  # wrap the whole exercise in a repeat (spec §5)
```

In `src/melete/alphatab/emit.py`, extend `_exercise_directives` to open the
repeat and `_exercise_bars` to close it. Add near the other tokens:

```python
REPEAT_OPEN = "\\ro"
REPEAT_CLOSE = "\\rc 2"   # two passes; confirmed against alphaTab in Step 1
```

In `_exercise_directives`, append the opener when the score repeats (it leads the
first bar, before the note tokens):

```python
    directives.append(f"\\tempo {slowest}")
    if score.repeat:
        directives.append(REPEAT_OPEN)
    return " ".join(directives)
```

In `_exercise_bars`, after the bar bodies are built and directives prepended to
`bodies[0]`, append the closer to the last bar:

```python
    if score.repeat:
        bodies[-1] = f"{bodies[-1]} {REPEAT_CLOSE}"
    return bodies
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `vrg-container-run -- python -m pytest tests/test_score.py tests/alphatab/test_alphatex_emit.py -q`
Expected: PASS.

- [ ] **Step 6: End-to-end golden**

Freeze a repeat golden and render it through `melete-render` to confirm a
loadable `.gp` (mirror the existing `tests/alphatab/test_render.py` black-box
test). Save the confirmed alphaTex to `tests/alphatab/golden/repeat.atex`.

Run: `vrg-container-run -- python -m pytest tests/alphatab/ -q`
Expected: PASS.

- [ ] **Step 7: Commit**

```bash
vrg-git add src/melete/score.py src/melete/alphatab/emit.py tests/ 
vrg-commit --type feat --scope alphatab --message "wrap exercises in repeat barlines (#57)"
```

---

## Task 3: The fitter — tiling and the meter ladder (no levers yet)

**Files:**
- Modify: `src/melete/layout.py`
- Test: `tests/test_layout.py`

**Interfaces:**
- Consumes: `LayoutHints` (Task 1).
- Produces: `LayoutPlan(subdivision: str, time_signature: tuple[int, int], bars:
  int, levers_applied: tuple[Lever, ...], trace: str)`; `fit(note_count: int,
  hints: LayoutHints) -> LayoutPlan`. In this task `fit` raises when no
  lever-free candidate tiles; Task 4 adds the lever search.

- [ ] **Step 1: Write the failing tests (the worked examples become assertions)**

```python
# tests/test_layout.py  (add)
from melete.layout import fit, LayoutPlan

def test_chromatic_48_notes_cell_4_is_6_4_by_2_bars():
    plan = fit(48, LayoutHints(cell=4, seam=24, levers=()))
    assert plan.subdivision == "sixteenth"
    assert plan.time_signature == (6, 4)
    assert plan.bars == 2
    assert plan.levers_applied == ()

def test_even_split_prefers_seam_aligned_larger_bar():
    # 32 eighths, cell 2 -> B=16 beats; even sane candidates 4/4x4 and 2/4x8;
    # seam at beat 8 aligns both, larger b wins -> 4/4 x 4.
    plan = fit(32, LayoutHints(cell=2, seam=16, levers=()))
    assert plan.time_signature == (4, 4)
    assert plan.bars == 4

def test_subdivision_follows_cell_size():
    assert fit(24, LayoutHints(cell=3, seam=None, levers=())).subdivision == "triplet_eighth"
    assert fit(24, LayoutHints(cell=6, seam=None, levers=())).subdivision == "sextuplet"

def test_no_lever_free_fit_raises_without_levers():
    # B = 14 beats: only sane-b even option is 7/4 (eccentric, rejected); no levers.
    import pytest
    with pytest.raises(ValueError, match="no whole-bar fit"):
        fit(28, LayoutHints(cell=2, seam=None, levers=()))
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `vrg-container-run -- python -m pytest tests/test_layout.py -q`
Expected: FAIL — `fit` not defined.

- [ ] **Step 3: Implement the tiling, candidate enumeration, and ladder**

```python
# src/melete/layout.py  (add)

#: Group size g -> the subdivision key (rhythm.SUBDIVISIONS) whose g notes fill
#: one quarter-note beat. Denominator therefore stays 4 (spec §4.1).
_SUBDIVISION_FOR_CELL: dict[int, str] = {
    1: "quarter", 2: "eighth", 3: "triplet_eighth",
    4: "sixteenth", 5: "quintuplet", 6: "sextuplet",
}

#: Sane beats-per-bar; a hard filter that outranks even-M (spec §4.5).
_SANE_BEATS: tuple[int, ...] = (2, 3, 4, 6)


@dataclass(frozen=True)
class LayoutPlan:
    """The fitter's decision for one exercise (spec §4)."""
    subdivision: str
    time_signature: tuple[int, int]
    bars: int
    levers_applied: tuple[Lever, ...]
    trace: str


def _candidates(beats: int, seam_beat: int | None) -> list[tuple[int, int]]:
    """Every ``(b, M)`` with ``b*M == beats`` and ``b`` sane, best-first.

    Ranked (spec §4.5): even ``M`` first, then a bar boundary on the seam, then
    larger ``b`` (fuller bars, fewer lines).
    """
    pairs = [(b, beats // b) for b in _SANE_BEATS if beats % b == 0]

    def rank(pair: tuple[int, int]) -> tuple[int, int, int]:
        b, m = pair
        even = 0 if m % 2 == 0 else 1
        on_seam = 0 if (seam_beat is not None and seam_beat % b == 0) else 1
        return (even, on_seam, -b)

    return sorted(pairs, key=rank)


def fit(note_count: int, hints: LayoutHints) -> LayoutPlan:
    """Derive a whole-bar layout for ``note_count`` notes (spec §4).

    This lever-free core tiles ``note_count / cell`` beats into sane bars. It
    raises when nothing tiles; Task 4 wraps it with the note-count lever search.
    """
    return _fit_fixed(note_count, hints, applied=())


def _fit_fixed(
    note_count: int, hints: LayoutHints, applied: tuple[Lever, ...]
) -> LayoutPlan:
    cell = hints.cell
    if cell not in _SUBDIVISION_FOR_CELL:
        msg = f"cell {cell} has no subdivision mapping; expected 1-6"
        raise ValueError(msg)
    if note_count % cell != 0:
        msg = (
            f"no whole-bar fit: {note_count} notes is not a multiple of the "
            f"cell {cell}, so beats are not whole (spec §4.6 lever needed)"
        )
        raise ValueError(msg)

    beats = note_count // cell
    seam_beat = None if hints.seam is None else hints.seam // cell
    ranked = _candidates(beats, seam_beat)
    if not ranked:
        msg = (
            f"no whole-bar fit: {beats} beats has no sane beats-per-bar in "
            f"{_SANE_BEATS} (spec §4.5); a note-count lever is required"
        )
        raise ValueError(msg)

    b, m = ranked[0]
    subdivision = _SUBDIVISION_FOR_CELL[cell]
    trace = (
        f"{note_count} notes / cell {cell} = {beats} beats; chose {b}/4 x {m} "
        f"(subdivision {subdivision}); candidates {ranked}; levers {list(applied)}"
    )
    return LayoutPlan(
        subdivision=subdivision,
        time_signature=(b, 4),
        bars=m,
        levers_applied=applied,
        trace=trace,
    )
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `vrg-container-run -- python -m pytest tests/test_layout.py -q`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
vrg-git add src/melete/layout.py tests/test_layout.py
vrg-commit --type feat --scope layout --message "fit notes into whole sane bars via the meter ladder (#57)"
```

---

## Task 4: The fitter — note-count levers and the fallback

**Files:**
- Modify: `src/melete/layout.py` (`fit`)
- Test: `tests/test_layout.py`

**Interfaces:**
- Consumes: `_fit_fixed`, `Lever`, `realize_lever` (Tasks 1, 3).
- Produces: `fit(note_count, hints)` now searches the legal levers (smallest
  effect first) before accepting an odd-but-sane count, and never returns an
  eccentric meter. Adds `plan_voice(voice, hints) -> tuple[Voice, LayoutPlan]`
  which realizes any chosen lever on the actual voice.

- [ ] **Step 1: Write the failing tests**

```python
# tests/test_layout.py  (add)
from melete.layout import plan_voice

def test_lever_fires_before_an_eccentric_meter():
    # B=14 -> only even sane meter is 7/4 (eccentric, rejected). ADD_ONE ->
    # 30 notes/2 = 15 beats (odd, no even sane); DROP_ONE -> 26/2 = 13 (prime).
    # Legal APEX_REPEAT changes by a cell; here use ADD_ONE/DROP_ONE to reach 12
    # or 16. 28 -> DROP twice not allowed (one lever); so ADD_ONE*? Keep it to the
    # reachable case: cell 2, allow DROP_ONE and ADD_ONE up to 2 notes.
    plan = fit(28, LayoutHints(cell=2, seam=None, levers=(Lever.DROP_ONE, Lever.ADD_ONE)))
    assert plan.time_signature[0] in _SANE_BEATS
    assert plan.time_signature != (7, 4)
    assert plan.bars % 2 == 0
    assert plan.levers_applied  # at least one lever was needed

def test_prefers_zero_levers_when_a_clean_fit_exists():
    plan = fit(48, LayoutHints(cell=4, seam=24, levers=(Lever.APEX_REPEAT, Lever.DROP_ONE)))
    assert plan.levers_applied == ()

def test_plan_voice_applies_the_chosen_lever_to_the_notes():
    notes = [ _n(23 + i) for i in range(47) ]           # 47 notes, cell 4 -> not whole
    out, plan = plan_voice(notes, LayoutHints(cell=4, seam=23, levers=(Lever.ADD_ONE,)))
    assert len(out) == 48
    assert plan.levers_applied == (Lever.ADD_ONE,)
```

*(`_SANE_BEATS` and `_n` are imported/defined as in Tasks 1 and 3.)*

- [ ] **Step 2: Run tests to verify they fail**

Run: `vrg-container-run -- python -m pytest tests/test_layout.py -q`
Expected: FAIL — `plan_voice` missing; `fit` doesn't search levers.

- [ ] **Step 3: Implement the lever search**

Replace `fit` with a search that tries no-lever first, then each legal single
lever (smallest count change first), preferring the result with a whitelisted `b`
**and** even `M`, then fewest levers:

```python
# src/melete/layout.py  (replace `fit`, add `plan_voice`)

#: Effect of each lever on the note count, smallest-magnitude first.
_LEVER_DELTA: dict[Lever, int] = {
    Lever.DROP_ONE: -1, Lever.ADD_ONE: +1,
    Lever.APEX_OMIT: -1, Lever.APEX_REPEAT: +1,
}


def _try(note_count: int, hints: LayoutHints, applied: tuple[Lever, ...]):
    try:
        return _fit_fixed(note_count, hints, applied)
    except ValueError:
        return None


def _quality(plan: LayoutPlan) -> tuple[int, int]:
    """Lower is better: prefer even bar count, then fewer levers."""
    return (0 if plan.bars % 2 == 0 else 1, len(plan.levers_applied))


def fit(note_count: int, hints: LayoutHints) -> LayoutPlan:
    """Whole-bar layout, engaging one legal lever only when needed (spec §4.5-4.6)."""
    best = _try(note_count, hints, ())
    if best is not None and best.bars % 2 == 0:
        return best  # clean, even, no lever — the ideal

    # For each legal lever (a cell-sized apex toggle counts as +/- cell), the new
    # count is note_count + delta (apex levers move by a whole cell).
    for lever in sorted(hints.levers, key=lambda lv: abs(_LEVER_DELTA[lv])):
        step = hints.cell if lever in (Lever.APEX_REPEAT, Lever.APEX_OMIT) else 1
        delta = (1 if _LEVER_DELTA[lever] > 0 else -1) * step
        candidate = _try(note_count + delta, hints, (lever,))
        if candidate is None:
            continue
        if best is None or _quality(candidate) < _quality(best):
            best = candidate

    if best is None:
        msg = (
            f"no whole-bar fit for {note_count} notes with cell {hints.cell} and "
            f"levers {list(hints.levers)} (spec §4.6): the pattern needs a new "
            f"lever declared by its family"
        )
        raise ValueError(msg)
    return best


def plan_voice(voice: Voice, hints: LayoutHints) -> tuple[Voice, LayoutPlan]:
    """Fit ``voice`` and realize any chosen lever on the notes (spec §4)."""
    plan = fit(len(voice), hints)
    adjusted = list(voice)
    for lever in plan.levers_applied:
        adjusted = realize_lever(adjusted, lever, hints.seam, hints.cell)
    return adjusted, plan
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `vrg-container-run -- python -m pytest tests/test_layout.py -q`
Expected: PASS.

- [ ] **Step 5: Add the no-partial property test**

```python
# tests/test_layout.py  (add)
import pytest
from melete.score import sounding_duration, Note
from fractions import Fraction
from melete import rhythm

@pytest.mark.parametrize("count,cell", [(48, 4), (30, 3), (26, 2), (27, 3), (16, 4)])
def test_fit_tiles_with_zero_remainder(count, cell):
    hints = LayoutHints(cell=cell, seam=None,
                        levers=(Lever.ADD_ONE, Lever.DROP_ONE))
    notes = [ _n(23 + (i % 12)) for i in range(count) ]
    adjusted, plan = plan_voice(notes, hints)
    # restamp to the plan's subdivision, then confirm bars * capacity == total
    stamped = rhythm.restamp(adjusted, plan.subdivision)   # added in Task 8
    beats, denom = plan.time_signature
    capacity = plan.bars * beats * Fraction(1, denom)
    assert sounding_duration(stamped) == capacity
```

*(If Task 8's `rhythm.restamp` is not yet available when this task runs, assert
on beat counts instead: `len(adjusted) % (cell) == 0` and
`(len(adjusted)//cell) == beats*bars`. Prefer the sounding-duration form once
Task 8 lands.)*

- [ ] **Step 6: Commit**

```bash
vrg-git add src/melete/layout.py tests/test_layout.py
vrg-commit --type feat --scope layout --message "engage note-count levers before eccentric meters (#57)"
```

---

## Task 5: Widen `generate` to `(Score, LayoutHints)` across the registry

**Files:**
- Modify: `src/melete/families/__init__.py` (`Generate` type)
- Modify: `src/melete/families/arpeggios.py`, `src/melete/families/intervals.py`
  (return hints)
- Modify: `src/melete/families/_shared.py` (optional `layout_hints` helper)
- Test: `tests/families/test_registry.py`, `tests/families/test_arpeggios.py`,
  `tests/families/test_intervals.py`

**Interfaces:**
- Consumes: `LayoutHints`, `Lever` (Task 1).
- Produces: every family's `generate(profile, params)` returns `tuple[Score,
  LayoutHints]`; `Generate = Callable[[InstrumentProfile, Params], tuple[Score,
  LayoutHints]]`.

- [ ] **Step 1: Write the failing test**

```python
# tests/families/test_registry.py  (add)
from melete.layout import LayoutHints
from melete.families import REGISTRY
from melete.instrument import PROFILES

def test_every_family_returns_score_and_hints(minimal_params_for):
    for name, family in REGISTRY.items():
        score, hints = family.generate(PROFILES["bass6"], minimal_params_for(name))
        assert isinstance(hints, LayoutHints)
        assert hints.cell >= 1
```

*(Reuse each family test's existing param builders for `minimal_params_for`; if
none exists, inline a per-family dict as the current family tests do.)*

- [ ] **Step 2: Run test to verify it fails**

Run: `vrg-container-run -- python -m pytest tests/families/test_registry.py -q`
Expected: FAIL — `generate` returns a bare `Score`.

- [ ] **Step 3: Implement**

In `families/__init__.py`:

```python
from melete.layout import LayoutHints  # add import
type Generate = Callable[[InstrumentProfile, Params], tuple["Score", "LayoutHints"]]
```

In `_shared.py`, add a helper families can call:

```python
from melete.layout import LayoutHints, Lever

def layout_hints(cell: int, seam: int | None, levers: tuple[Lever, ...]) -> LayoutHints:
    """Assemble a family's LayoutHints (spec §4.2)."""
    return LayoutHints(cell=cell, seam=seam, levers=levers)
```

In `arpeggios.py` and `intervals.py`, change the `return Score(...)` to
`return Score(...), _shared.layout_hints(cell=<cell>, seam=<seam or None>,
levers=(Lever.ADD_ONE, Lever.DROP_ONE))`, where `<cell>` is that family's natural
group (arpeggios: the arpeggio window length; intervals: the interval-pair
length — read the family's window constant). `seam` is the turnaround index when
the family's `direction` is `up_down` (the ascending-half length), else `None`.

- [ ] **Step 4: Run tests to verify they pass**

Run: `vrg-container-run -- python -m pytest tests/families/ -q`
Expected: PASS (existing family tests that unpack `generate` must be updated to
`score, _ = family.generate(...)` — fix them in this step).

- [ ] **Step 5: Commit**

```bash
vrg-git add src/melete/families/ tests/families/
vrg-commit --type refactor --scope families --message "return LayoutHints from every family (#57)"
```

---

## Task 6: Chromatic — all-strings cycle, apex-repeat lever, hints

**Files:**
- Modify: `src/melete/families/chromatic.py` (`_strings`, `generate`)
- Test: `tests/families/test_chromatic.py`

**Interfaces:**
- Consumes: `_shared.layout_hints`, `Lever` (Tasks 1, 5).
- Produces: `chromatic.generate` returns `(Score, LayoutHints)` with `cell ==
  len(permutation)`; the `up_down` base cycle is the **apex-once** `there_and_back`
  (the apex string's cell is NOT pre-repeated — that is the fitter's
  `APEX_REPEAT` lever), with `seam` at the last note of the ascending half and
  `levers=(Lever.APEX_REPEAT, Lever.APEX_OMIT)`. When the profile requests it, the
  chromatic string walk spans **all** strings.

- [ ] **Step 1: Write the failing test**

```python
# tests/families/test_chromatic.py  (add)
from melete.instrument import PROFILES
from melete.layout import Lever, plan_voice

def test_chromatic_all_strings_baseline_is_apex_once_fitter_reaches_48():
    profile = PROFILES["bass6"]
    params = {  # all-strings, up_down, adjacent, 1-2-4-3
        "permutation": "1_2_4_3", "start_string": "0", "start_fret": "1",
        "direction": "up_down", "string_traversal": "adjacent",
        "shift": "none", "span": "all",
    }
    score, hints = chromatic.generate(profile, params)
    # BASE cycle: there_and_back over 6 strings = 11 string-groups x 4 = 44 notes
    assert len(score.voice) == 44
    assert hints.cell == 4
    assert hints.levers == (Lever.APEX_REPEAT, Lever.APEX_OMIT)
    assert hints.seam == 23                     # last note of the 6-string ascent
    # the FITTER, not the family, repeats the apex cell to reach a clean fit:
    fitted, plan = plan_voice(score.voice, hints)
    assert len(fitted) == 48                    # APEX_REPEAT added the 4-note apex cell
    assert plan.levers_applied == (Lever.APEX_REPEAT,)
    assert plan.time_signature == (6, 4)
    assert plan.bars == 2
```

- [ ] **Step 2: Run test to verify it fails**

Run: `vrg-container-run -- python -m pytest tests/families/test_chromatic.py -q`
Expected: FAIL — no `"all"` span; `generate` returns a bare Score.

- [ ] **Step 3: Implement**

Allow `span == "all"` to mean "every string from `start_string`," and keep the
apex-once turnaround (`there_and_back`) as the base cycle — the fitter, not the
family, decides whether to repeat the apex. In `_strings`, resolve `"all"` to the
string count and leave `up_down` on `there_and_back`:

```python
def _strings(direction, traversal, start_string, span, string_count):
    step = -_STRING_STEP[traversal] if direction == "down" else _STRING_STEP[traversal]
    reach = string_count if span == "all" else span
    walk = [start_string + cycle * step for cycle in range(reach)]
    return there_and_back(walk) if direction == "up_down" else walk   # apex once
```

*(`span` may now be an int-string or `"all"`; read it via `read.value("span")` and
branch. `string_count = len(profile.tuning)`. The base cycle is unchanged in kind
from today — `there_and_back` — only the reach widens to all strings.)*

In `generate`, compute `permutation`, build `strings` with `string_count`, then
return hints. The seam is the last note of the ascending half; for a
`there_and_back` of `reach` strings the ascent is `reach` string-groups, so the
apex note is at index `reach * cell - 1`:

```python
    permutation = _permutation(read)
    reach = len(profile.tuning) if span == "all" else int(span)
    strings = _strings(direction, traversal, start_string, span, len(profile.tuning))
    # ... positions/voice as today ...
    cell = len(permutation)
    seam = reach * cell - 1 if direction == "up_down" else None
    hints = _shared.layout_hints(cell=cell, seam=seam,
                                 levers=(Lever.APEX_REPEAT, Lever.APEX_OMIT))
    return Score(..., time_signature=DEFAULT_TIME_SIGNATURE, ...), hints
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `vrg-container-run -- python -m pytest tests/families/test_chromatic.py -q`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
vrg-git add src/melete/families/chromatic.py tests/families/test_chromatic.py
vrg-commit --type feat --scope chromatic --message "span all strings and emit apex-repeat hints (#57)"
```

---

## Task 7: Scales — two octaves from the lowest string, `up_down` default, fallback

**Files:**
- Modify: `src/melete/families/scales.py` (`generate`, `_places`/traversal helpers)
- Test: `tests/families/test_scales.py`

**Interfaces:**
- Consumes: `_shared.layout_hints`, `Lever` (Tasks 1, 5).
- Produces: `scales.generate` returns `(Score, LayoutHints)`; two-octave span is
  attempted from the lowest string of `string_set`; where a `positional`
  traversal cannot hold two octaves on bass6, a declared fallback applies and is
  recorded in `params["layout_fallback"]`; `direction` defaults to `up_down`.

- [ ] **Step 1: Write the failing tests**

```python
# tests/families/test_scales.py  (add)
from melete.layout import Lever

def test_scales_default_direction_is_up_down():
    profile = PROFILES["bass6"]
    params = _scale_params(direction=None)             # omit direction
    score, hints = scales.generate(profile, params)
    # ascending then descending -> a seam roughly mid-voice
    assert hints.seam is not None
    assert Lever.APEX_REPEAT in hints.levers or Lever.APEX_OMIT in hints.levers

def test_positional_two_octave_fallback_is_recorded_not_silent():
    profile = PROFILES["bass6"]
    params = _scale_params(root=8, scale_type="ionian",  # Ab Ionian
                           traversal="positional", range_octaves=2)
    score, hints = scales.generate(profile, params)
    # either two octaves fit, or a declared fallback is recorded
    fallback = score.params.get("layout_fallback")
    assert fallback in (None, "one_octave_up_down", "switch_traversal")
    # never a silent position shift: positional stays within one position
```

*(`_scale_params` is a small local builder over the scales AXES; base it on the
existing `test_scales.py` param construction.)*

- [ ] **Step 2: Run tests to verify they fail**

Run: `vrg-container-run -- python -m pytest tests/families/test_scales.py -q`
Expected: FAIL — `direction` has no default; no fallback recorded; bare Score.

- [ ] **Step 3: Implement**

- Default `direction` to `up_down` when absent: read it with a default rather than
  `read.identifier("direction")` when the key is missing.
- After computing `places` for `positional`, check feasibility: if `boxed(...)`
  would exceed `profile.position_span` for two octaves, apply the declared
  fallback — retry at one octave (`octave_count = 1`) with `up_down`, and set
  `params["layout_fallback"] = "one_octave_up_down"`. Wrap the existing
  `boxed(...)` call (which already raises on span overflow) in a try:

```python
    fallback = None
    try:
        places = _places(profile, pitches, strings, traversal, scale_type, octave_count)
    except ValueError:
        if traversal == "positional" and octave_count > 1:
            octave_count = 1
            pitches = theory.scale_pitches(root, scale_type, octave_count)
            places = _places(profile, pitches, strings, traversal, scale_type, octave_count)
            fallback = "one_octave_up_down"
        else:
            raise
```

- Compute `cell` from the chosen `pattern` window length
  (`len(_PATTERN_WINDOWS[pattern])`; `straight` → 1 means the uniform-pulse
  fallback — treat `cell = 1` as eighths by mapping straight scales to a uniform
  eighth pulse via `LayoutHints(cell=2, ...)` **only** when notes-per-beat is 2;
  otherwise keep `cell = len(window)`). Record `direction`'s seam (the ascending
  half length) for `up_down`.
- Return `(Score(..., params={**dict(params), **({"layout_fallback": fallback} if
  fallback else {})}), hints)`.

*(The uniform-pulse fallback for unequal positional groups, spec §4.3: when the
pattern is `straight` and the traversal yields unequal per-string counts, set
`cell = 2` (eighths) so the fitter tiles by total count and leans on levers. This
is the one place cell selection is not simply the window length; comment it.)*

- [ ] **Step 4: Run tests to verify they pass**

Run: `vrg-container-run -- python -m pytest tests/families/test_scales.py -q`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
vrg-git add src/melete/families/scales.py tests/families/test_scales.py
vrg-commit --type feat --scope scales --message "two octaves from the low string with a declared fallback (#57)"
```

---

## Task 8: Pipeline — the fitter drives meter and subdivision

**Files:**
- Modify: `src/melete/rhythm.py` (split sampling from restamping)
- Modify: `src/melete/cli.py` (`_generate` pipeline: `generate → fit → restamp`)
- Test: `tests/test_rhythm.py`, `tests/test_cli_generate.py`

**Interfaces:**
- Consumes: `plan_voice`, `LayoutPlan` (Task 4); `(Score, LayoutHints)` from
  families (Tasks 5-7).
- Produces: `rhythm.restamp(voice: Voice, subdivision: str, *, note_value_pattern:
  str = "straight", accent_pattern: str = "none") -> Voice` — the duration/tuplet
  stamping, with the subdivision **supplied**, not sampled. The pipeline sets
  `Score.time_signature` from the fitter and `Score.repeat = True`, and stores the
  trace in `params["layout_trace"]`.

- [ ] **Step 1: Write the failing test**

```python
# tests/test_rhythm.py  (add)
def test_restamp_uses_the_supplied_subdivision_not_a_sampled_one():
    from melete import rhythm
    notes = [ _q(60) for _ in range(6) ]               # six quarter notes
    out = rhythm.restamp(notes, "triplet_eighth")
    # 6 triplet-eighths -> two (3,2) tuplets, each note written 1/8
    from melete.score import Tuplet
    assert all(isinstance(x, Tuplet) for x in out)
    assert sum(len(t.notes) for t in out) == 6
```

```python
# tests/test_cli_generate.py  (add — an integration assertion)
def test_generated_exercise_has_no_partial_measure(project, node, run):
    run("generate", "--date", "2026-08-13")
    # load the emitted book.atex source and assert every exercise tiles: the
    # fitter set the meter, so bar() leaves no short final measure.
    # (Assert via the session's src/*.atex: each exercise's total sounding time
    #  is an integer multiple of its bar capacity.)
    ...
```

*(Flesh out the CLI assertion using the existing `sources(project, on)` helper to
read `src/exercise-NN.atex`; parse the `\ts` and confirm bar count is integral.
If parsing alphaTex in a test is heavy, assert instead that `bar(score.voice,
score.time_signature)` on the in-memory Score yields measures whose last measure
`sounding_duration == capacity`, exercised through a thin pipeline entry point.)*

- [ ] **Step 2: Run tests to verify they fail**

Run: `vrg-container-run -- python -m pytest tests/test_rhythm.py -q`
Expected: FAIL — `rhythm.restamp` not defined.

- [ ] **Step 3: Refactor `rhythm.py`**

Extract the duration/accent/grouping half of `apply` into a `restamp` that takes
the subdivision (and optional overlays) as arguments instead of reading
`time_signature`/`subdivision` from `params`:

```python
def restamp(voice, subdivision, *, note_value_pattern="straight", accent_pattern="none"):
    """Stamp durations/tuplets onto `voice` at a *given* subdivision (spec §7).

    The subdivision and meter are chosen by the layout fitter, not sampled here.
    Accent and note-value overlays remain grid-preserving (module docstring).
    """
    written, ratio = SUBDIVISIONS[subdivision]
    notes = list(_flatten(voice))
    if not notes:
        msg = "cannot restamp an empty voice; a family emits one full cycle (§7)"
        raise ValueError(msg)
    group = ratio[0] if ratio is not None else len(notes)
    pattern = NOTE_VALUE_PATTERNS[note_value_pattern]
    durations = _durations(len(notes), written, pattern, group)
    accents = _accents(len(notes), ACCENTS[accent_pattern])
    restamped = [replace(n, duration=d, accent=a)
                 for n, d, a in zip(notes, durations, accents, strict=True)]
    return _grouped(restamped, ratio)
```

Keep `apply` only if another caller needs the old sampled form; otherwise remove
it and update `AXES` (Task 9). The `time_signature`/`subdivision` reads leave
`rhythm.py` — they now come from the fitter.

- [ ] **Step 4: Wire the pipeline in `cli._generate`**

Where the pipeline currently produces each Score (and calls `rhythm.apply`),
replace with: unpack `(score, hints)`, run the fitter, restamp, and set meter +
repeat + trace. Locate the call site first:

Run: `vrg-container-run -- grep -rn "rhythm.apply\|\.generate(" src/melete/cli.py src/melete/selection.py`

Then, per drawn spec:

```python
from melete import layout, rhythm
from dataclasses import replace

score, hints = family.generate(profile, params)
adjusted, plan = layout.plan_voice(score.voice, hints)
voice = rhythm.restamp(adjusted, plan.subdivision,
                       note_value_pattern=params.get("note_value_pattern", "straight"),
                       accent_pattern=params.get("accent_pattern", "none"))
score = replace(score, voice=voice, time_signature=plan.time_signature,
                repeat=True, params={**score.params, "layout_trace": plan.trace,
                                     **{k: v for k, v in (("layout_fallback",
                                        score.params.get("layout_fallback")),) if v}})
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `vrg-container-run -- python -m pytest tests/test_rhythm.py tests/test_cli_generate.py -q`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
vrg-git add src/melete/rhythm.py src/melete/cli.py tests/
vrg-commit --type feat --scope pipeline --message "derive meter and subdivision from the fitter (#57)"
```

---

## Task 9: Retire the sampled meter/subdivision axes

**Files:**
- Modify: `src/melete/config.py` (`_RHYTHM_AXES`)
- Modify: `src/melete/rhythm.py` (`AXES`)
- Modify: `src/melete/vocabulary.py` (remove `time_signature`/`subdivision`
  entries **iff** nothing else reads them)
- Modify: `examples/config.toml` (drop the two keys)
- Test: `tests/test_config.py`, `tests/test_vocabulary.py`, `tests/test_rhythm.py`

**Interfaces:**
- Consumes: Task 8 (nothing samples these axes anymore).
- Produces: `[pool.rhythm]` accepts only `accent_patterns` and
  `note_value_patterns`; `time_signatures`/`subdivisions` in a config are rejected
  as unknown keys.

- [ ] **Step 1: Write the failing test**

```python
# tests/test_config.py  (add)
def test_rhythm_pool_rejects_the_retired_time_signature_key(tmp_path):
    import pytest
    from melete import config
    cfg = _write_config(tmp_path, rhythm='time_signatures = ["4_4"]')
    with pytest.raises(ValueError, match="unknown"):
        config.load(cfg)
```

```python
# tests/test_rhythm.py  (add)
def test_rhythm_axes_no_longer_sample_meter_or_subdivision():
    from melete import rhythm
    assert "time_signature" not in rhythm.AXES
    assert "subdivision" not in rhythm.AXES
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `vrg-container-run -- python -m pytest tests/test_config.py tests/test_rhythm.py -q`
Expected: FAIL — both keys still accepted/sampled.

- [ ] **Step 3: Implement**

- In `config.py`, remove `_TIME_SIGNATURE` and `_SUBDIVISION` from the
  `_RHYTHM_AXES` tuple (the `_registry("time_signatures", …)` /
  `_registry("subdivisions", …)` entries). `_reject_unknown` then rejects those
  TOML keys.
- In `rhythm.py`, change `AXES = ("accent_pattern", "note_value_pattern")` (drop
  the two). If `apply` was removed in Task 8, delete any now-dead references.
- In `vocabulary.py`, remove the `time_signature` and `subdivision` entries from
  `AXES` **only if** no reader remains (grep first:
  `grep -rn '"time_signature"\|"subdivision"' src/`). If `cli` display or a
  family still reads them, keep the entries and note why in the commit.
- In `examples/config.toml`, delete `subdivisions = …` and `time_signatures = …`
  from `[pool.rhythm]`, leaving `accent_patterns` and `note_value_patterns`.

- [ ] **Step 4: Run the full suite**

Run: `vrg-container-run -- vrg-validate`
Expected: PASS (this catches every test that constructed rhythm params with the
retired axes — fix those to the new shape as they surface).

- [ ] **Step 5: Commit**

```bash
vrg-git add src/melete/config.py src/melete/rhythm.py src/melete/vocabulary.py examples/config.toml tests/
vrg-commit --type refactor --scope config --message "retire the sampled meter and subdivision axes (#57)"
```

---

## Task 10: Development output discipline and repo `MEMORY.md`

**No code change to the output path.** `melete generate` writing to
`./sessions/<date>/` relative to the working directory is correct and stays
(spec §9). This task records a *development-time* convention.

**Files:**
- Create: `MEMORY.md`
- Modify: `docs/` (a short note on running from `build/` during development)
- (Manual, outside the repo) remove the sibling `../sample-gp/` artifacts

**Interfaces:** none (documentation + memory).

- [ ] **Step 1: Confirm `build/` is gitignored (no path reroute needed)**

Run: `vrg-container-run -- grep -n "build/\|sessions/" .gitignore`
Expected: both present. No `.gitignore` and no CLI/path change — the runtime
behavior is intentionally unchanged; the discipline is *where we run* the
generator in development.

- [ ] **Step 2: Create `MEMORY.md` with the human-approval policy header and the entry**

Use the vergil policy header (via `/vergil:memory-init` if available, or write it
directly), then add the approved entry:

```markdown
- **In development, run melete from `build/`; never write generated artifacts to
  the repo base or a sibling repo.** Runtime writes generated practice artifacts
  (`.gp`, `.atex`, `session.json`) to `./sessions/<date>/` relative to the working
  directory — intentional and unchanged. When running the generator inside the
  repo during development, run it from the gitignored `build/` directory so output
  lands in `build/sessions/…`. Never write generated artifacts into the repo base
  directory, the VM scratchpad/temp (invisible from the user's macOS host), or a
  sibling directory one level above the repo (those are separate git repositories
  — the `sample-gp/` mistake). The flat-file `sessions/` model is acknowledged
  tech debt to be rethought for scale in a future effort.
```

- [ ] **Step 3: Add the convention to the docs**

In the CLI/usage doc (e.g. `docs/reference/cli.md` or wherever `generate`'s output
paths are documented), add a sentence: during development run melete from `build/`
so artifacts land in `build/sessions/<date>/`; never commit generated output.

- [ ] **Step 4: Remove the misplaced sibling artifacts (manual)**

The `../sample-gp/` directory is a **sibling of the repo**, not tracked by melete;
it cannot be removed by a PR. Note it for the human to delete:
`rm -rf /Users/pmoore/dev/projects/mnemosys-project/sample-gp`. Do not delete it
from an agent session without explicit confirmation.

- [ ] **Step 5: Commit**

```bash
vrg-git add MEMORY.md docs/
vrg-commit --type docs --scope repo --message "record the build/ development output discipline in memory (#57)"
```

---

## Task 11: Regenerate the five exercises as acceptance goldens

**Files:**
- Create: `tests/alphatab/golden/practice-2026-08-13/*.atex` (five exercises)
- Test: `tests/alphatab/test_practice_goldens.py` *(new)*

**Interfaces:** consumes the whole pipeline (Tasks 2, 6-8).

- [ ] **Step 1: Generate the day and inspect**

Run: `vrg-container-run -- python -m melete generate --date 2026-08-13`
Then read `sessions/2026-08-13/src/*.atex`. Confirm by eye against spec §8:

- exercise 01 chromatic → `\ts 6 4`, sixteenths, 2 bars, `\ro …\rc 2`;
- A♭ Ionian two-octave scale → `\ts 4 4`, 4 bars (or the fitter's recorded meter);
- every exercise has `\ro`/`\rc 2` and **no short final bar**.

- [ ] **Step 2: Write the golden test**

```python
# tests/alphatab/test_practice_goldens.py
from pathlib import Path
GOLDEN = Path(__file__).parent / "golden" / "practice-2026-08-13"

def test_each_exercise_repeats_and_has_no_partial_measure():
    for atex in sorted(GOLDEN.glob("exercise-*.atex")):
        text = atex.read_text()
        assert "\\ro" in text and "\\rc 2" in text
        # every '|'-separated bar carries a full beat count for the stated \ts:
        # (assert via the shared bar-parsing helper, or freeze exact bytes below)
```

- [ ] **Step 3: Freeze the goldens**

Copy the confirmed `src/exercise-01.atex … exercise-05.atex` into
`tests/alphatab/golden/practice-2026-08-13/`. These are the regression artifacts.

- [ ] **Step 4: Validate end to end**

Run: `vrg-container-run -- vrg-validate`
Expected: PASS. Render the book once more and confirm the `.gp` loads with no red
bars (this is also the epic's `validation` task, seeded at plan time — see below).

- [ ] **Step 5: Commit**

```bash
vrg-git add tests/alphatab/golden/practice-2026-08-13/ tests/alphatab/test_practice_goldens.py
vrg-commit --type test --scope alphatab --message "freeze the regenerated five exercises as goldens (#57)"
```

---

## Operational task to seed at plan time

- **Validation (`--kind validation`):** regenerate the 2026-08-13 practice book
  and confirm in Guitar Pro that every exercise loads with **no red/partial bars**
  and repeats correctly. This is a live visual check the pipeline's tests cannot
  fully make (spec §10, §11). Seed it blocked-by Task 11.

---

## Self-review

**Spec coverage:**

- §3 derive-don't-sample → Tasks 8, 9. ✓
- §4 fitter (cell pulse, ladder, levers, trace) → Tasks 1, 3, 4. ✓
- §4.2 LayoutHints contract, `generate → (Score, LayoutHints)` → Tasks 1, 5-7. ✓
- §5 repeat barlines → Task 2. ✓
- §6 chromatic all-strings; scales two-octave-from-lowest; `up_down` default;
  conditional fallback → Tasks 6, 7. ✓
- §8 worked examples → Tasks 3 (chromatic 6/4×2, scale 4/4×4 as tests), 11
  (all five as goldens). ✓
- §9 build/ cleanup + memory → Task 10. ✓
- §10 testing (fitter units, no-partial property, repeat golden, five goldens,
  output-path) → Tasks 1, 3, 4 (property), 2, 11. ✓
- §11 task breakdown → matches this plan; bookends already seeded (#58, #110, #59);
  validation seeded above. ✓

**Placeholder scan:** the two integration assertions in Task 8 Step 1 and Task 11
Step 2 intentionally leave the *parsing* approach to the implementer, with a
concrete fallback stated (assert on `bar()`'s last-measure `sounding_duration`).
Every other step carries real code. No `TODO`/`TBD` remain.

**Type consistency:** `LayoutHints(cell, seam, levers)`, `Lever` members,
`LayoutPlan(subdivision, time_signature, bars, levers_applied, trace)`, `fit(int,
LayoutHints) -> LayoutPlan`, `plan_voice(Voice, LayoutHints) -> (Voice,
LayoutPlan)`, `realize_lever(Voice, Lever, int|None) -> Voice`, and
`rhythm.restamp(Voice, str, *, note_value_pattern, accent_pattern) -> Voice` are
used consistently across Tasks 1, 3, 4, 5-8. `generate -> (Score, LayoutHints)`
is uniform from Task 5 on.

**Known latitude (heuristic engine, spec §3, §12):** the ladder's tie-breaks and
the `cell = 2` uniform-pulse fallback for unequal positional groups are the
corner-case surface the spec expects to tune. Tasks 3-4 pin the *committed*
behavior with tests (the worked examples); later corner cases extend the trace and
the tests, not the architecture.
