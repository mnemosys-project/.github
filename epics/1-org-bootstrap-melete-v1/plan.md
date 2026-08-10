# Org Bootstrap and Melete v1 — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use `superpowers:subagent-driven-development`
> (recommended) or `superpowers:executing-plans` to implement this plan
> task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Epic:** [`mnemosys-project/.github#1`](https://github.com/mnemosys-project/.github/issues/1)
**Spec:** [`spec.md`](./spec.md)

**Goal:** Bootstrap the `mnemosys-project` organization from zero and deliver
`melete` v1 — a command that generates a printable daily bass practice sheet.

**Architecture:** Three phases. Phase A completes the organization in `.github`
and stands up the `docs` and `melete` repositories. Phase B builds melete inside
out, from pure 12-TET math through the Score IR to the LilyPond emitter, so that
every layer is testable before the layer above it exists. Phase C proves the
result on real hardware and real paper.

**Tech Stack:** Python 3.14, `uv`, pytest, ruff, mypy. One runtime dependency:
the PyPI `lilypond` redistribution (2.25.12). Development inside
`vrg-container-run` against `dev-python`.

## Global Constraints

Every task's requirements implicitly include this section. Values are copied
verbatim from the spec.

- **Python 3.14**, `uv`-managed. Dev group: pytest, ruff, mypy (spec §15).
- **Runtime dependency is `lilypond` and nothing else** (spec §15). Adding any
  other runtime dependency requires revisiting decision #5.
- **No swallowed exceptions, no fallbacks that hide errors** (spec §13). A
  failure in the generator becomes a wrong exercise on the page, which is worse
  than no exercise.
- **`note.pitch == instrument.tuning[note.string] + note.fret`** — the central
  invariant (spec §14). Every family test asserts it for every generated note.
- **`Note.duration` is the *written* value.** Sounding time is derived via
  `Tuplet.ratio`, never stored (spec §6, decision #16).
- **Weight formula:** `w = max(0.05, min(1.0, sessions_since / horizon))`,
  horizon default 14 (spec §9, decision #15). No weight is ever zero.
- **Fret counts are part of the profile definition:** `bass4` = 20, `bass5` = 24,
  `bass6` = 24 (spec §5, decision #18). Never inferred.
- **`bass6` is the default profile** (spec §5).
- **No key signatures by default** — explicit accidentals throughout (spec §10).
- **All parameter identifiers come from `vocabulary.py`** (spec §13,
  decision #19). No module hardcodes an identifier string.
- **Tempo is a per-family default, overridable in config, never sampled**
  (spec §7, decision #20).
- **Families take `params: dict` and return `Score`** (spec §7). `ExerciseSpec`
  and `WeightInputs` live in `selection.py`, upstream of the families
  (decision #21). No family imports from `selection`.
- **An explicit tuning that is not strictly ascending is a load error**, never
  re-sorted (spec §13, decision #22).
- **Validation is `vrg-container-run -- vrg-validate` and nothing else.** Do not
  invoke individual linters.
- **Git and GitHub go through `vrg-git` and `vrg-gh`.** Raw `git`/`gh` are denied.
- **Agents never submit or merge PRs.** Record readiness with
  `vrg-pr-workflow report-ready`; the human runs `vrg-submit-pr`.

## The REFACTOR Step

Every task in Phase B ends with a REFACTOR step **before** its commit. It is
stated once here rather than repeated verbatim in fifteen places, because it is
identical every time and the plan preaches DRY.

Red and green are already explicit in each task's steps: write the failing test,
run it to confirm it fails, implement minimally, run it to confirm it passes.
REFACTOR is the third beat, and the one that gets skipped unless it is written
down:

- [ ] **REFACTOR (standing step for every Phase B task)**
  - Extract duplicated logic. The four families will grow near-identical
    position-selection and direction-handling code — the second time you write
    it, move it to a shared helper.
  - Move hard-coded values to `vocabulary.py` or config. Any bare identifier
    string in a module is a defect under the Global Constraints.
  - Consolidate with existing patterns rather than inventing a parallel one.
  - Improve names, then re-run the task's tests to confirm they still pass.

A task is not complete until this step has been performed and its tests are
green afterwards.

## Placement Law

A task lives in the repo where its closing PR lands. A PR only `Closes` an issue
in its own repo; cross-repo links are `Ref` or comments. Every task below names
its repo, and that is where its issue is filed.

## Human-Gated Preconditions

Two classes of step in this plan are **not agent-performable**, mirroring the
release boundary in `epic-create`. Each is a precondition another task is
`Blocked-by`, attested by the human, never performed by the agent:

| Gate | Why |
|---|---|
| **Repository creation** (`vrg-github-repo-init` for `docs` and `melete`) | Organization and repository creation is a human act. This is the same reason the `.github` bootstrap ran on the host, outside the sandbox (spec, *Epic Scope and Filing Order*). |
| **PR submission and merge** | Standing policy. Agents report ready; humans submit. |

An agent reaching one of these stops, comments "blocked: preconditions not met",
and does not fabricate the step.

---

# Phase A — Organization

Delivers a complete, conventional GitHub organization. Stands alone: even if
melete were abandoned, the org would be correctly formed.

## Task A1: Org metadata and community health files

**Repo:** `mnemosys-project/.github`

**Files:**
- Create: `profile/README.md`
- Create: `CONTRIBUTING.md`
- Create: `CODE_OF_CONDUCT.md`
- Create: `SECURITY.md`
- Create: `SUPPORT.md`
- Create: `.github/pull_request_template.md`
- Create: `.github/ISSUE_TEMPLATE/config.yml`
- Create: `.github/ISSUE_TEMPLATE/task.yml`
- Create: `.github/ISSUE_TEMPLATE/idea.yml`

**Interfaces:**
- Consumes: nothing.
- Produces: the org profile surface. `profile/README.md` renders at
  `github.com/mnemosys-project`.

**Reference:** model on `vergil-project/.github`, which has all of these. Read
them first — do not invent structure.

- [ ] **Step 1: Read the reference org's equivalents**

```bash
vrg-gh api repos/vergil-project/.github/contents/profile/README.md \
  --jq '.content' | base64 -d
vrg-gh api repos/vergil-project/.github/contents/CONTRIBUTING.md \
  --jq '.content' | base64 -d
```

Note structure and tone. Do not copy Vergil-specific policy verbatim — this org
has different tooling expectations.

- [ ] **Step 2: Write `profile/README.md`**

Must state: the org is named for memory; each tool is named for a Muse; link to
`NAMING.md`. Name `melete` (assigned) and `aoede` (reserved). Keep it short —
this is a landing page, not documentation.

- [ ] **Step 3: Write the remaining health files**

`CONTRIBUTING.md` states the epic/task model, the `vrg-*` wrapper policy, and
the worktree convention. `SECURITY.md` gives a reporting address.
`SUPPORT.md` points at issues. `CODE_OF_CONDUCT.md` uses Contributor Covenant
2.1 verbatim.

- [ ] **Step 4: Write the issue and PR templates**

`config.yml` disables blank issues and links to the epic model.
`task.yml` and `idea.yml` mirror the `task` and `idea` labels in the registry.

- [ ] **Step 5: Validate**

```bash
vrg-container-run -- vrg-validate
```
Expected: PASS.

- [ ] **Step 6: Commit and report ready**

```bash
vrg-commit --type docs --scope org \
  --message "add org metadata and community health files"
vrg-pr-workflow report-ready
```

## Task A2: Epic document format standards

**Repo:** `mnemosys-project/.github`

Implements the spec's *Document formats are standardized as part of this epic*.

**Files:**
- Create: `docs/epic-document-formats.md`
- Create: `docs/templates/spec.md`
- Create: `docs/templates/plan.md`
- Create: `docs/templates/retrospective.md`

**Interfaces:**
- Consumes: this epic's own `spec.md` and `plan.md` as the reference instances.
- Produces: the templates every future epic in this org starts from.

- [ ] **Step 1: Derive the spec template from `epics/1-org-bootstrap-melete-v1/spec.md`**

Extract the section skeleton — Overview, Scope, Architecture, the domain
sections, Error Handling, Testing Strategy, Recorded Decisions, Deferred. Keep
the **Recorded Decisions table** mandatory; it is the highest-value section and
the one most likely to be skipped.

- [ ] **Step 2: Derive the plan template from this file**

Mandatory: Global Constraints, Placement Law, per-task Files and Interfaces
blocks, checkbox steps.

- [ ] **Step 3: Write the retrospective template**

Sections: what shipped, what did not, what surprised us, what to change,
follow-on work (§5, per `epic-retrospective`).

- [ ] **Step 4: Write `docs/epic-document-formats.md`**

State the rule: a reader follows **spec → plan → retrospective**; each document
answers a different question (what and why / how and in what order / what
actually happened). Name this epic as the reference implementation.

- [ ] **Step 5: Validate, commit, report ready**

```bash
vrg-container-run -- vrg-validate
vrg-commit --type docs --scope standards \
  --message "add epic document format standards and templates"
vrg-pr-workflow report-ready
```

## Task A3: Integrate the org banner image and its design record

**Repo:** `mnemosys-project/.github`
**Blocks:** A1 — the banner is the top of `profile/README.md`, which A1 writes.

The banner was designed and generated in a separate session during bootstrap.
Seven images, six prompts, and a 578-line design document were left in the plain
parent directory, outside version control. This task brings them in.

**Files:**
- Create: `profile/banner.png` (the accepted v03 generation)
- Create: `docs/branding/banner-design.md`
- Create: `docs/branding/prompts/*.txt` (six prompts, v01–v04)
- Create: `docs/branding/generations/*.png` (six non-accepted generations)
- Modify: `epics/1-org-bootstrap-melete-v1/plan.md` (this file — A3 added,
  former A3/A4 renumbered to A4/A5)

- [ ] **Step 1: Place the accepted image at the path the README will reference**

`mnemosys-project-main-image-v03.png` becomes `profile/banner.png`. It lives
**once**, and is not duplicated into `generations/` — that would carry 3 MB
twice for no gain.

- [ ] **Step 2: Archive every other generation and every prompt**

Roughly 20 MB in total. Keep all of it. The design document's §16 is explicit
that the drift between what was specified and what arrived is the only place the
generator's actual behaviour is recorded, and that record produced three
transferable findings — cold prompts beat deltas, the generator drops predicates
and multiplies nouns, and figure scale was never specified in any prompt.

- [ ] **Step 3: Update the design document's own location notes**

It carries a "provisional location" header saying it belongs in
`mnemosys-project/.github` once bootstrap lands, and a closing "Next: move these
files into version control." Both are now satisfied and must be replaced rather
than left describing a state that no longer holds.

- [ ] **Step 4: Validate, commit, report ready**

```bash
vrg-container-run -- vrg-validate
vrg-pr-workflow report-ready --issue 8 ...
```

## Task A4: Create the `docs` repository

**Repo:** `mnemosys-project/docs`
**Blocked-by:** human-gated repository creation.

- [ ] **Step 1: HUMAN GATE — create the repository**

The human runs:

```bash
vrg-github-repo-init --org mnemosys-project --repo docs
```

Profile to answer with (matching `.github`, which releases nothing):

```toml
repository-type   = "documentation"
versioning-scheme = "none"
branching-model   = "docs-single-branch"
release-model     = "none"
```

The agent verifies and stops if absent:

```bash
vrg-gh repo view mnemosys-project/docs --json name,visibility
```
If this fails: comment "blocked: preconditions not met" and stop.

- [ ] **Step 2: Write the initial site content**

`docs/site/index.md`: what the org is, the memory/Muse naming thesis, the tool
roster with status (melete assigned, aoede reserved). Link to `NAMING.md` in
`.github` rather than duplicating it — one authoritative copy (NAMING.md §5).

- [ ] **Step 3: Validate, commit, report ready**

```bash
vrg-container-run -- vrg-validate
vrg-commit --type docs --scope site --message "add initial org site"
vrg-pr-workflow report-ready
```

## Task A5: Create the `melete` repository

**Repo:** `mnemosys-project/melete`
**Blocked-by:** human-gated repository creation.

- [ ] **Step 1: Cross-check the branching model before creating**

Spec §15 flags `branching-model` as the one value worth verifying. Check a
Logical Minds Foundry application repo:

```bash
vrg-gh api repos/logical-minds-foundry/<repo>/contents/vergil.toml \
  --jq '.content' | base64 -d
```
If a comparable no-deploy application uses something other than
`library-release`, record the finding on issue #1 and adopt theirs.

- [ ] **Step 2: HUMAN GATE — create the repository**

```bash
vrg-github-repo-init --org mnemosys-project --repo melete
```

Profile (spec §15):

```toml
repository-type   = "application"
versioning-scheme = "semver"
branching-model   = "library-release"
release-model     = "tagged-release"
primary-language  = "python"
```

- [ ] **Step 3: Verify the scaffolding landed**

Expected from the wizard (spec §15): `.claude/hooks/guard.sh`,
`.claude/settings.json`, `docs/repository-standards.md`,
`.github/workflows/ci.yml`, `.gitignore` containing `.worktrees/`, `CLAUDE.md`.

```bash
vrg-gh repo view mnemosys-project/melete --json name,defaultBranchRef
```

- [ ] **Step 4: Add the parallel-agent worktree section to `CLAUDE.md`**

Copy the section from `mnemosys-project/.github/CLAUDE.md` verbatim.

- [ ] **Step 5: Pin dependencies**

`pyproject.toml`:

```toml
[project]
name = "melete"
requires-python = ">=3.14"
dependencies = ["lilypond==2.25.12"]

[dependency-groups]
dev = ["pytest", "ruff", "mypy"]
```

The exact pin is deliberate (spec §15, decision #13): the redistribution is not
tracking upstream, and an unpinned range would silently change the renderer.

- [ ] **Step 6: Validate, commit, report ready**

---

# Phase B — Melete

Built inside out. Every task's deliverable is testable without the layer above
it existing. All tasks in this phase are **Repo: `mnemosys-project/melete`** and
**Blocked-by: A5**.

## Task B1: `theory.py` — 12-TET math

**Files:**
- Create: `src/melete/theory.py`
- Test: `tests/test_theory.py`

**Interfaces:**
- Consumes: nothing. This is the base layer.
- Produces:
  - `PITCH_CLASSES: tuple[str, ...]` — 12 names, index = pitch class
  - `SCALES: dict[str, tuple[int, ...]]` — identifier → semitone offsets from root
  - `CHORDS: dict[str, tuple[int, ...]]` — identifier → semitone offsets
  - `scale_pitches(root: int, scale_type: str, octaves: int) -> list[int]`
  - `chord_pitches(root: int, quality: str, inversion: int) -> list[int]`

- [ ] **Step 1: Write the failing test for scale content**

```python
from melete.theory import scale_pitches

def test_ionian_intervals():
    # C4 = 60. One octave, ascending, root inclusive of the octave.
    assert scale_pitches(60, "ionian", 1) == [60, 62, 64, 65, 67, 69, 71, 72]

def test_dorian_differs_from_ionian_at_third_and_seventh():
    ionian = scale_pitches(60, "ionian", 1)
    dorian = scale_pitches(60, "dorian", 1)
    assert dorian[2] == ionian[2] - 1
    assert dorian[6] == ionian[6] - 1

def test_blues_has_six_notes_per_octave():
    assert len(scale_pitches(60, "blues", 1)) == 7  # six + octave
```

- [ ] **Step 2: Run it and confirm it fails**

```bash
vrg-container-run -- uv run pytest tests/test_theory.py -v
```
Expected: FAIL, `ModuleNotFoundError: No module named 'melete.theory'`.

- [ ] **Step 3: Implement**

```python
"""Pitch, interval, scale, and chord math. Pure integers, 12-TET, C4 = 60."""

PITCH_CLASSES = ("C", "Db", "D", "Eb", "E", "F",
                 "Gb", "G", "Ab", "A", "Bb", "B")

SCALES: dict[str, tuple[int, ...]] = {
    "ionian":     (0, 2, 4, 5, 7, 9, 11),
    "dorian":     (0, 2, 3, 5, 7, 9, 10),
    "phrygian":   (0, 1, 3, 5, 7, 8, 10),
    "lydian":     (0, 2, 4, 6, 7, 9, 11),
    "mixolydian": (0, 2, 4, 5, 7, 9, 10),
    "aeolian":    (0, 2, 3, 5, 7, 8, 10),
    "locrian":    (0, 1, 3, 5, 6, 8, 10),
    "major_pentatonic": (0, 2, 4, 7, 9),
    "minor_pentatonic": (0, 3, 5, 7, 10),
    "blues":            (0, 3, 5, 6, 7, 10),
    "whole_tone":       (0, 2, 4, 6, 8, 10),
    # melodic minor, harmonic minor, and their modes, plus the two
    # diminished scales, complete the ~28 of spec section 7.
}


def scale_pitches(root: int, scale_type: str, octaves: int) -> list[int]:
    """Ascending pitches across `octaves`, inclusive of the final octave."""
    if scale_type not in SCALES:
        raise KeyError(
            f"unknown scale_type {scale_type!r}; "
            f"accepted: {sorted(SCALES)}"
        )
    if octaves < 1:
        raise ValueError(f"octaves must be >= 1, got {octaves}")
    offsets = SCALES[scale_type]
    out = [root + o + 12 * n for n in range(octaves) for o in offsets]
    out.append(root + 12 * octaves)
    return out
```

The `KeyError` message names the key and its accepted values, per spec §13.

- [ ] **Step 4: Run tests and confirm they pass**

- [ ] **Step 5: Add the exhaustive sweep**

Spec §14 requires all 12 roots against all scale types:

```python
import pytest
from melete.theory import SCALES, scale_pitches

@pytest.mark.parametrize("root", range(60, 72))
@pytest.mark.parametrize("scale_type", sorted(SCALES))
def test_every_scale_is_strictly_ascending(root, scale_type):
    pitches = scale_pitches(root, scale_type, 2)
    assert pitches == sorted(pitches)
    assert len(set(pitches)) == len(pitches)
    assert pitches[-1] == root + 24
```

- [ ] **Step 6: Commit**

```bash
vrg-commit --type feat --scope theory --message "add 12-TET scale and chord math"
```

## Task B2: `instrument.py` — profiles and the fretboard

**Files:**
- Create: `src/melete/instrument.py`
- Test: `tests/test_instrument.py`

**Interfaces:**
- Consumes: `theory` (for pitch constants only).
- Produces:
  - `InstrumentProfile` (frozen: `name`, `tuning: tuple[int, ...]`, `fret_count: int`)
  - `PROFILES: dict[str, InstrumentProfile]` — `bass4`, `bass5`, `bass6`
  - `positions(profile, pitch) -> list[tuple[int, int]]` — all `(string, fret)`
  - `pitch_at(profile, string, fret) -> int`

- [ ] **Step 1: Write the failing tests**

```python
from melete.instrument import PROFILES, positions, pitch_at

def test_fret_counts_are_declared_not_inferred():
    assert PROFILES["bass4"].fret_count == 20
    assert PROFILES["bass5"].fret_count == 24
    assert PROFILES["bass6"].fret_count == 24

def test_bass6_is_six_strings_low_to_high():
    tuning = PROFILES["bass6"].tuning
    assert len(tuning) == 6
    assert tuning == tuple(sorted(tuning))

def test_open_string_pitch():
    p = PROFILES["bass4"]
    assert pitch_at(p, 0, 0) == p.tuning[0]

def test_positions_are_all_valid():
    p = PROFILES["bass6"]
    for string, fret in positions(p, 48):
        assert 0 <= fret <= p.fret_count
        assert pitch_at(p, string, fret) == 48
```

- [ ] **Step 2: Run and confirm failure**

- [ ] **Step 3: Implement**

```python
from dataclasses import dataclass

# E1 = 28, A1 = 33, D2 = 38, G2 = 43, B0 = 23, C3 = 48
@dataclass(frozen=True)
class InstrumentProfile:
    name: str
    tuning: tuple[int, ...]   # absolute pitches, low to high; index 0 = lowest
    fret_count: int


PROFILES = {
    "bass4": InstrumentProfile("bass4", (28, 33, 38, 43), 20),
    "bass5": InstrumentProfile("bass5", (23, 28, 33, 38, 43), 24),
    "bass6": InstrumentProfile("bass6", (23, 28, 33, 38, 43, 48), 24),
}

DEFAULT_PROFILE = "bass6"


def pitch_at(profile: InstrumentProfile, string: int, fret: int) -> int:
    return profile.tuning[string] + fret


def positions(profile: InstrumentProfile, pitch: int) -> list[tuple[int, int]]:
    """Every (string, fret) that sounds `pitch` on this profile."""
    return [
        (s, pitch - open_pitch)
        for s, open_pitch in enumerate(profile.tuning)
        if 0 <= pitch - open_pitch <= profile.fret_count
    ]
```

- [ ] **Step 4: Run and confirm pass**

- [ ] **Step 5: Add the exhaustive position sweep (spec §14)**

```python
import pytest
from melete.instrument import PROFILES, positions, pitch_at

@pytest.mark.parametrize("name", sorted(PROFILES))
def test_every_reachable_pitch_round_trips(name):
    p = PROFILES[name]
    lo, hi = p.tuning[0], p.tuning[-1] + p.fret_count
    for pitch in range(lo, hi + 1):
        for string, fret in positions(p, pitch):
            assert pitch_at(p, string, fret) == pitch
```

- [ ] **Step 6: Commit**

## Task B3: `vocabulary.py` — the identifier registry

**Files:**
- Create: `src/melete/vocabulary.py`
- Test: `tests/test_vocabulary.py`

**Interfaces:**
- Consumes: `theory.SCALES`, `theory.CHORDS`.
- Produces: `AXES: dict[str, dict[str, str]]` — axis name → {identifier: display}.
  Plus `display(axis, identifier) -> str` and `accepted(axis) -> list[str]`.

Spec §13 and decision #19. One registry, three consumers.

- [ ] **Step 1: Write the failing tests**

```python
from melete import theory
from melete.vocabulary import AXES, display, accepted

def test_every_scale_type_has_a_display_name():
    assert set(AXES["scale_type"]) == set(theory.SCALES)

def test_display_names_are_human_readable():
    assert display("traversal", "three_note_per_string") == "three-notes-per-string"
    assert display("scale_type", "dorian") == "Dorian"

def test_unknown_identifier_names_accepted_values():
    try:
        display("traversal", "nonsense")
    except KeyError as exc:
        assert "nonsense" in str(exc)
        assert "positional" in str(exc)
    else:
        raise AssertionError("expected KeyError")

def test_accepted_is_sorted_for_stable_error_messages():
    assert accepted("traversal") == sorted(accepted("traversal"))
```

- [ ] **Step 2: Run and confirm failure**

- [ ] **Step 3: Implement**

```python
from melete import theory

AXES: dict[str, dict[str, str]] = {
    "family": {
        "chromatic": "chromatic", "scales": "scales",
        "arpeggios": "arpeggios", "intervals": "intervals",
    },
    "scale_type": {
        "ionian": "Ionian", "dorian": "Dorian", "phrygian": "Phrygian",
        "lydian": "Lydian", "mixolydian": "Mixolydian",
        "aeolian": "Aeolian", "locrian": "Locrian",
        "major_pentatonic": "major pentatonic",
        "minor_pentatonic": "minor pentatonic",
        "blues": "blues", "whole_tone": "whole-tone",
    },
    "traversal": {
        "positional": "positional",
        "three_note_per_string": "three-notes-per-string",
        "octave_per_string": "one-octave-per-string",
        "single_string": "single-string linear",
    },
    "pattern": {
        "straight": "straight", "thirds": "thirds", "fourths": "fourths",
        "groups_of_3": "groups of 3", "groups_of_4": "groups of 4",
    },
    "direction": {"up": "ascending", "down": "descending", "up_down": "up-down"},
    "subdivision": {
        "quarter": "quarters", "eighth": "eighths",
        "triplet_eighth": "triplet eighths", "sixteenth": "sixteenths",
        "sextuplet": "sextuplets", "quintuplet": "quintuplets",
    },
    "accent_pattern": {
        "none": "no accents", "every_3": "accent every 3",
        "every_5": "accent every 5", "displaced": "displaced by one",
    },
}


def accepted(axis: str) -> list[str]:
    if axis not in AXES:
        raise KeyError(f"unknown axis {axis!r}; accepted: {sorted(AXES)}")
    return sorted(AXES[axis])


def display(axis: str, identifier: str) -> str:
    values = AXES[axis] if axis in AXES else {}
    if identifier not in values:
        raise KeyError(
            f"unknown {axis} {identifier!r}; accepted: {accepted(axis)}"
        )
    return values[identifier]
```

- [ ] **Step 4: Run and confirm pass**

- [ ] **Step 5: Add the drift guard**

This test is the whole point of the registry — it fails the moment
`theory.SCALES` grows a scale nobody named:

```python
def test_no_scale_type_lacks_a_display_name():
    missing = set(theory.SCALES) - set(AXES["scale_type"])
    assert not missing, f"scale types with no display name: {sorted(missing)}"
```

- [ ] **Step 6: Commit**

## Task B4: `score.py` — the IR and the seam

**Files:**
- Create: `src/melete/score.py`
- Test: `tests/test_score.py`

**Interfaces:**
- Consumes: `instrument.InstrumentProfile`.
- Produces: `Note`, `Tuplet`, `Voice`, `Score`, and
  `sounding_duration(voice) -> Fraction` — the single shared helper of
  decision #16.

- [ ] **Step 1: Write the failing tests, pinning the written-duration contract**

```python
from fractions import Fraction
from melete.score import Note, Tuplet, sounding_duration

def _n(dur):
    return Note(pitch=60, string=0, fret=0, duration=dur,
                finger=None, accent=False)

def test_plain_notes_sound_as_written():
    assert sounding_duration([_n(Fraction(1, 4))] * 4) == Fraction(1)

def test_triplet_notes_are_written_eighths_sounding_a_quarter():
    trip = Tuplet(ratio=(3, 2), notes=[_n(Fraction(1, 8))] * 3)
    # Written: three eighths = 3/8. Sounding: 3/8 * 2/3 = 1/4.
    assert sounding_duration([trip]) == Fraction(1, 4)

def test_mixed_voice_sums_correctly():
    trip = Tuplet(ratio=(3, 2), notes=[_n(Fraction(1, 8))] * 3)
    assert sounding_duration([_n(Fraction(1, 4)), trip]) == Fraction(1, 2)
```

- [ ] **Step 2: Run and confirm failure**

- [ ] **Step 3: Implement**

```python
from dataclasses import dataclass, field
from fractions import Fraction

from melete.instrument import InstrumentProfile


@dataclass(frozen=True)
class Note:
    pitch: int                # absolute semitones, C4 = 60
    string: int               # index into tuning, 0 = lowest
    fret: int                 # 0 = open
    duration: Fraction        # WRITTEN value; 1/4 = quarter note
    finger: int | None        # left hand, 1-4; None = unspecified
    accent: bool


@dataclass(frozen=True)
class Tuplet:
    ratio: tuple[int, int]    # (3, 2) = three in the time of two
    notes: list[Note]


Voice = list[Note | Tuplet]


@dataclass(frozen=True)
class Score:
    title: str
    instruction: str
    instrument: InstrumentProfile
    time_signature: tuple[int, int]
    tempo_range: tuple[int, int]
    voice: Voice
    params: dict = field(default_factory=dict)


def sounding_duration(voice: Voice) -> Fraction:
    """Real elapsed time of a voice. Written durations are scaled by ratio."""
    total = Fraction(0)
    for item in voice:
        if isinstance(item, Tuplet):
            num, den = item.ratio
            written = sum((n.duration for n in item.notes), Fraction(0))
            total += written * den / num
        else:
            total += item.duration
    return total
```

- [ ] **Step 4: Run and confirm pass**

- [ ] **Step 5: Commit**

## Task B5: `families/chromatic.py`

**Files:**
- Create: `src/melete/families/__init__.py`
- Create: `src/melete/families/chromatic.py`
- Test: `tests/families/test_chromatic.py`
- Test: `tests/families/conftest.py` (the shared invariant assertion)

**Interfaces:**
- Consumes: `score`, `instrument`, `vocabulary`.
- Produces: `generate(profile, params) -> Score`, and
  `families.REGISTRY: dict[str, Callable]`.

- [ ] **Step 1: Write the shared invariant helper first**

Every family reuses this. `tests/families/conftest.py`:

```python
from melete.score import Note, Tuplet


def assert_central_invariant(score):
    """spec section 14: pitch must equal open string plus fret, always."""
    for item in score.voice:
        notes = item.notes if isinstance(item, Tuplet) else [item]
        for n in notes:
            assert n.pitch == score.instrument.tuning[n.string] + n.fret, (
                f"pitch {n.pitch} != string {n.string} + fret {n.fret}"
            )
            assert 0 <= n.fret <= score.instrument.fret_count
            assert 0 <= n.string < len(score.instrument.tuning)
```

- [ ] **Step 2: Write the failing family test**

```python
from melete.families.chromatic import generate
from melete.instrument import PROFILES
from tests.families.conftest import assert_central_invariant

PARAMS = {
    "permutation": (1, 2, 3, 4), "start_string": 0, "start_fret": 5,
    "direction": "up", "string_traversal": "adjacent",
    "shift": "none", "span": 4,
}

def test_generates_one_complete_cycle():
    s = generate(PROFILES["bass6"], PARAMS)
    # span 4 strings x 4 fingers = 16 notes, no truncation.
    assert len(s.voice) == 16

def test_obeys_the_central_invariant():
    assert_central_invariant(generate(PROFILES["bass6"], PARAMS))

def test_fingering_is_first_class():
    s = generate(PROFILES["bass6"], PARAMS)
    assert [n.finger for n in s.voice[:4]] == [1, 2, 3, 4]

def test_params_travel_inside_the_score():
    s = generate(PROFILES["bass6"], PARAMS)
    assert s.params == PARAMS

def test_default_tempo_range_is_slow_for_finger_independence():
    # spec section 7: chromatic work is deliberate; speed defeats it.
    assert generate(PROFILES["bass6"], PARAMS).tempo_range == (60, 80)
```

- [ ] **Step 3: Run and confirm failure**

- [ ] **Step 4: Implement `generate` as a pure function**

No I/O, no randomness — the selector chooses params, the family realizes them
(spec §7). Walk strings from `start_string` for `span`, and at each string emit
one note per finger in `permutation` order at `start_fret + finger - 1`.

- [ ] **Step 5: Run and confirm pass**

- [ ] **Step 6: Add the property sweep**

```python
import pytest
from melete.instrument import PROFILES

@pytest.mark.parametrize("profile_name", sorted(PROFILES))
@pytest.mark.parametrize("start_fret", range(0, 13))
def test_invariant_holds_across_the_sweep(profile_name, start_fret):
    profile = PROFILES[profile_name]
    params = {**PARAMS, "start_fret": start_fret,
              "span": min(4, len(profile.tuning))}
    assert_central_invariant(generate(profile, params))
```

- [ ] **Step 7: Commit**

## Task B6: `families/scales.py`

**Files:**
- Create: `src/melete/families/scales.py`
- Test: `tests/families/test_scales.py`

**Interfaces:**
- Consumes: `theory.scale_pitches`, `instrument.positions`, `vocabulary`.
- Produces: `generate(profile, params) -> Score`. Default tempo 80–100
  (spec §7, decision #20).

The largest family — roughly 24,000 variants before rhythm (spec §7).

- [ ] **Step 1: Write the failing tests**

```python
from melete.families.scales import generate
from melete.instrument import PROFILES
from tests.families.conftest import assert_central_invariant

BASE = {
    "root": 33, "scale_type": "ionian", "traversal": "positional",
    "string_set": (0, 1, 2, 3), "pattern": "straight",
    "range_octaves": 2, "direction": "up",
}

def test_two_octave_ionian_ascending_note_count():
    s = generate(PROFILES["bass6"], BASE)
    assert len(s.voice) == 15               # 2 x 7 + closing octave

def test_up_down_returns_without_repeating_the_apex():
    s = generate(PROFILES["bass6"], {**BASE, "direction": "up_down"})
    assert len(s.voice) == 29               # 15 up + 14 down

def test_thirds_pattern_produces_more_notes_than_straight():
    straight = generate(PROFILES["bass6"], BASE)
    thirds = generate(PROFILES["bass6"], {**BASE, "pattern": "thirds"})
    assert len(thirds.voice) > len(straight.voice)

def test_three_note_per_string_puts_exactly_three_notes_on_each_string():
    s = generate(PROFILES["bass6"],
                 {**BASE, "traversal": "three_note_per_string"})
    from collections import Counter
    counts = Counter(n.string for n in s.voice)
    assert set(counts.values()) == {3}

def test_scale_content_matches_theory():
    s = generate(PROFILES["bass6"], BASE)
    assert {n.pitch % 12 for n in s.voice} == {(33 + o) % 12
                                               for o in (0, 2, 4, 5, 7, 9, 11)}

def test_stays_within_the_declared_string_set():
    s = generate(PROFILES["bass6"], BASE)
    assert {n.string for n in s.voice} <= set(BASE["string_set"])

def test_default_tempo_range():
    assert generate(PROFILES["bass6"], BASE).tempo_range == (80, 100)

def test_obeys_the_central_invariant():
    assert_central_invariant(generate(PROFILES["bass6"], BASE))
```

- [ ] **Step 2: Run and confirm failure**

```bash
vrg-container-run -- uv run pytest tests/families/test_scales.py -v
```
Expected: FAIL, no module `melete.families.scales`.

- [ ] **Step 3: Implement**

Position selection is the family's job, not the emitter's (spec §6). For
`positional`, choose the position minimizing total fret travel within
`string_set`. For `three_note_per_string`, take exactly three consecutive scale
degrees per string. `pattern` reorders the realized degree sequence before
positions are assigned — `thirds` emits 1-3-2-4-3-5…, `groups_of_3` emits
1-2-3, 2-3-4, 3-4-5….

- [ ] **Step 4: Run and confirm pass**

- [ ] **Step 5: Add the exhaustive sweep (spec §14)**

```python
import pytest
from melete import theory

@pytest.mark.parametrize("root", range(24, 36))
@pytest.mark.parametrize("scale_type", sorted(theory.SCALES))
def test_invariant_across_every_root_and_scale(root, scale_type):
    params = {**BASE, "root": root, "scale_type": scale_type}
    assert_central_invariant(generate(PROFILES["bass6"], params))
```

- [ ] **Step 6: REFACTOR** (see the standing step), then commit

## Task B6a: shared family helpers

**Files:**
- Create: `src/melete/families/_shared.py`
- Test: `tests/families/test_shared.py`

Extracted during B6's REFACTOR, once the duplication between chromatic and
scales is real rather than anticipated. Do not write this before B6.

**Interfaces:**
- Produces: `apply_direction(pitches, direction)`,
  `assign_positions(profile, pitches, string_set, traversal)`.

- [ ] **Step 1: Write the failing test for `apply_direction`**

```python
def test_up_down_does_not_repeat_the_apex():
    assert apply_direction([1, 2, 3], "up_down") == [1, 2, 3, 2, 1]

def test_down_reverses():
    assert apply_direction([1, 2, 3], "down") == [3, 2, 1]
```

- [ ] **Step 2: Confirm failure, extract from B5/B6, confirm both families' tests still pass**

- [ ] **Step 3: REFACTOR, then commit**

## Task B7: `families/arpeggios.py`

**Files:**
- Create: `src/melete/families/arpeggios.py`
- Test: `tests/families/test_arpeggios.py`

**Interfaces:**
- Consumes: `theory.chord_pitches`, `instrument.positions`, `_shared`.
- Produces: `generate(profile, params) -> Score`. Default tempo 80–100.

Axes per spec §7: `root`, `quality`, `inversion`, `traversal`, `string_set`,
`pattern`, `range_octaves`, `direction`.

- [ ] **Step 1: Write the failing tests**

```python
from melete.families.arpeggios import generate
from melete.instrument import PROFILES
from tests.families.conftest import assert_central_invariant

BASE = {
    "root": 33, "quality": "maj7", "inversion": 0,
    "traversal": "across_strings", "string_set": (0, 1, 2, 3),
    "pattern": "straight", "range_octaves": 1, "direction": "up",
}

def test_maj7_has_four_chord_tones_plus_the_octave():
    s = generate(PROFILES["bass6"], BASE)
    assert len(s.voice) == 5

def test_chord_tone_content_is_root_third_fifth_seventh():
    s = generate(PROFILES["bass6"], BASE)
    assert {n.pitch % 12 for n in s.voice} == {(33 + o) % 12
                                               for o in (0, 4, 7, 11)}

def test_min7_flattens_the_third_and_seventh():
    maj = generate(PROFILES["bass6"], BASE)
    mn = generate(PROFILES["bass6"], {**BASE, "quality": "min7"})
    assert sorted(n.pitch for n in mn.voice)[1] == \
           sorted(n.pitch for n in maj.voice)[1] - 1

def test_first_inversion_starts_on_the_third():
    root_pos = generate(PROFILES["bass6"], BASE)
    first = generate(PROFILES["bass6"], {**BASE, "inversion": 1})
    assert first.voice[0].pitch % 12 == (root_pos.voice[0].pitch + 4) % 12

def test_broken_pattern_reorders_without_changing_content():
    straight = generate(PROFILES["bass6"], BASE)
    broken = generate(PROFILES["bass6"], {**BASE, "pattern": "broken"})
    assert sorted(n.pitch for n in broken.voice) != \
           [n.pitch for n in broken.voice]
    assert {n.pitch for n in broken.voice} == {n.pitch for n in straight.voice}

def test_default_tempo_range():
    assert generate(PROFILES["bass6"], BASE).tempo_range == (80, 100)

def test_obeys_the_central_invariant():
    assert_central_invariant(generate(PROFILES["bass6"], BASE))
```

- [ ] **Step 2: Run and confirm failure**

- [ ] **Step 3: Implement**

`inversion` rotates the chord-tone sequence before octave expansion.
`traversal = "across_strings"` assigns one chord tone per string where the
`string_set` allows; `positional` keeps the hand in one position.

- [ ] **Step 4: Run and confirm pass**

- [ ] **Step 5: Add the exhaustive sweep**

```python
import pytest
from melete.theory import CHORDS

@pytest.mark.parametrize("root", range(24, 36))
@pytest.mark.parametrize("quality", sorted(CHORDS))
@pytest.mark.parametrize("inversion", (0, 1, 2, 3))
def test_invariant_across_roots_qualities_and_inversions(root, quality, inversion):
    params = {**BASE, "root": root, "quality": quality, "inversion": inversion}
    assert_central_invariant(generate(PROFILES["bass6"], params))
```

Inversions beyond a triad's chord-tone count must raise, not wrap silently —
an inversion of 3 on a triad is a caller bug (spec §13).

- [ ] **Step 6: REFACTOR, then commit**

## Task B8: `families/intervals.py`

**Files:**
- Create: `src/melete/families/intervals.py`
- Test: `tests/families/test_intervals.py`

**Interfaces:**
- Consumes: `theory`, `instrument.positions`, `_shared`.
- Produces: `generate(profile, params) -> Score`. Default tempo 70–90.

This family exists because tablature makes string topology expressible
(spec §7); these exercises cannot be described by pitch alone. Its tests must
therefore assert **string** relationships, not only pitch — that is the whole
justification for decision #4, and the assertions below are where it is proven.

- [ ] **Step 1: Write the failing tests**

```python
from melete.families.intervals import generate
from melete.instrument import PROFILES
from tests.families.conftest import assert_central_invariant

BASE = {
    "interval": 3, "context": "diatonic", "root": 33,
    "scale_type": "ionian", "string_skip": 0,
    "string_set": (0, 1, 2, 3), "direction": "up",
    "pattern": "ascending_pairs",
}

def test_diatonic_thirds_are_three_or_four_semitones():
    s = generate(PROFILES["bass6"], BASE)
    pitches = [n.pitch for n in s.voice]
    for a, b in zip(pitches[::2], pitches[1::2]):
        assert b - a in (3, 4)          # major or minor third in context

def test_chromatic_context_makes_every_interval_exact():
    s = generate(PROFILES["bass6"], {**BASE, "context": "chromatic"})
    pitches = [n.pitch for n in s.voice]
    for a, b in zip(pitches[::2], pitches[1::2]):
        assert b - a == 4               # a literal third, unmodified by key

def test_string_skip_1_never_uses_adjacent_strings():
    s = generate(PROFILES["bass6"], {**BASE, "string_skip": 1})
    strings = [n.string for n in s.voice]
    for a, b in zip(strings, strings[1:]):
        assert abs(a - b) != 1

def test_string_skip_2_leaves_two_strings_between():
    s = generate(PROFILES["bass6"], {**BASE, "string_skip": 2})
    strings = [n.string for n in s.voice]
    assert any(abs(a - b) >= 3 for a, b in zip(strings, strings[1:]))

def test_adjacent_skip_zero_uses_neighbouring_strings():
    s = generate(PROFILES["bass6"], {**BASE, "string_skip": 0})
    strings = [n.string for n in s.voice]
    assert all(abs(a - b) <= 1 for a, b in zip(strings, strings[1:]))

def test_descending_pairs_invert_the_pair_order():
    up = generate(PROFILES["bass6"], BASE)
    down = generate(PROFILES["bass6"],
                    {**BASE, "pattern": "descending_pairs"})
    assert [n.pitch for n in down.voice] != [n.pitch for n in up.voice]
    assert {n.pitch for n in down.voice} == {n.pitch for n in up.voice}

def test_default_tempo_range():
    assert generate(PROFILES["bass6"], BASE).tempo_range == (70, 90)

def test_obeys_the_central_invariant():
    assert_central_invariant(generate(PROFILES["bass6"], BASE))
```

- [ ] **Step 2: Run and confirm failure**

- [ ] **Step 3: Implement**

`string_skip` is a hard constraint on position assignment, not a preference. A
pitch pair that cannot be placed at the required string distance within
`string_set` makes the specification invalid — raise, and let §9's validity gate
resample it. Never quietly fall back to an adjacent string; that would produce a
plausible exercise that is not the one requested (spec §13).

- [ ] **Step 4: Run and confirm pass**

- [ ] **Step 5: Add the sweep across intervals and skips**

```python
import pytest

@pytest.mark.parametrize("interval", range(2, 11))
@pytest.mark.parametrize("string_skip", (0, 1, 2))
def test_invariant_across_intervals_and_skips(interval, string_skip):
    params = {**BASE, "interval": interval, "string_skip": string_skip}
    try:
        score = generate(PROFILES["bass6"], params)
    except ValueError:
        return          # over-constrained: section 9 resamples, not a failure
    assert_central_invariant(score)
```

- [ ] **Step 6: REFACTOR, then commit**

## Task B9: `rhythm.py` — the cross-cutting modifier

**Files:** `src/melete/rhythm.py`, `tests/test_rhythm.py`

**Interfaces:** Consumes `score`. Produces `apply(score, params) -> Score`.

Rhythm is a modifier of type `Score -> Score`, not a family (spec §8,
decision #3).

- [ ] **Step 1: Write the failing tests**

```python
from fractions import Fraction
from melete.rhythm import apply
from melete.score import sounding_duration

def test_triplet_eighths_produce_tuplets_with_written_eighths():
    out = apply(base_score, {"subdivision": "triplet_eighth",
                             "time_signature": (4, 4),
                             "accent_pattern": "none",
                             "note_value_pattern": "straight"})
    trip = out.voice[0]
    assert trip.ratio == (3, 2)
    assert all(n.duration == Fraction(1, 8) for n in trip.notes)

def test_every_written_duration_is_a_representable_notehead():
    out = apply(base_score, {"subdivision": "sextuplet", ...})
    for dur in written_durations(out):
        assert dur.numerator in (1, 3, 7)      # plain, dotted, double-dotted
        assert (dur / dur.numerator).denominator in (1, 2, 4, 8, 16, 32, 64)

def test_accent_every_3():
    out = apply(base_score, {"accent_pattern": "every_3", ...})
    accents = [n.accent for n in flatten(out.voice)]
    assert accents[::3] == [True] * len(accents[::3])

def test_long_short_alternates_written_durations():
    out = apply(base_score, {"subdivision": "eighth",
                             "note_value_pattern": "long_short", ...})
    durs = [n.duration for n in flatten(out.voice)]
    assert durs[0] > durs[1]
    assert durs[0::2] == [durs[0]] * len(durs[0::2])

def test_long_short_preserves_total_sounding_duration():
    straight = apply(base_score, {"note_value_pattern": "straight", ...})
    swung = apply(base_score, {"note_value_pattern": "long_short", ...})
    assert sounding_duration(swung.voice) == sounding_duration(straight.voice)

def test_short_long_is_the_mirror_of_long_short():
    ls = apply(base_score, {"note_value_pattern": "long_short", ...})
    sl = apply(base_score, {"note_value_pattern": "short_long", ...})
    assert [n.duration for n in flatten(sl.voice)][:2] == \
           list(reversed([n.duration for n in flatten(ls.voice)][:2]))
```

The second assertion is the one that matters: a note-value pattern redistributes
time within the pattern, it does not add or remove any. If total sounding
duration changes, the exercise no longer fits the cycle length that §7's
`max_notes` gate and §14's duration test both reason about.

- [ ] **Step 2: Confirm failure**

- [ ] **Step 3: Implement**

Map each `subdivision` to a written duration and, where it is a tuplet
subdivision, a ratio:

```python
SUBDIVISIONS = {
    "quarter":        (Fraction(1, 4),  None),
    "eighth":         (Fraction(1, 8),  None),
    "sixteenth":      (Fraction(1, 16), None),
    "triplet_eighth": (Fraction(1, 8),  (3, 2)),
    "sextuplet":      (Fraction(1, 16), (6, 4)),
    "quintuplet":     (Fraction(1, 16), (5, 4)),
}
```

- [ ] **Step 4: Confirm pass**

- [ ] **Step 5: Add the cycle-length assertion (spec §14, decision #16)**

```python
def test_sounding_duration_equals_cycle_length():
    out = apply(base_score, {"subdivision": "triplet_eighth", ...})
    note_count = len(flatten(out.voice))
    assert sounding_duration(out.voice) == note_count * Fraction(1, 12)
```

- [ ] **Step 6: Commit**

## Task B10: `selection.py` — coverage-aware sampling

The highest-risk module in the plan. Spec §9 and decisions #14, #15, #17.

**Files:** `src/melete/selection.py`, `tests/test_selection.py`

**Interfaces:**
- Consumes: `vocabulary.AXES`, `instrument`, family registry.
- Produces:
  - `ExerciseSpec` (frozen: `family: str`, `params: dict`)
  - `WeightInputs` (frozen: `distances: dict[str, dict[str, int | None]]`)
  - `weight(sessions_since: int | None, horizon: int) -> float`
  - `select(config, history, rng) -> list[tuple[ExerciseSpec, WeightInputs]]`

Both types live here, upstream of the families (decision #21). Families keep
their `params: dict` signature and never import from this module.

- [ ] **Step 0: Define the two types**

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class ExerciseSpec:
    family: str          # key into the family REGISTRY
    params: dict         # passed verbatim to that family's generate()


@dataclass(frozen=True)
class WeightInputs:
    """Per-axis recency distances that produced one draw.

    Recorded into session.json so replay reconstructs the draw without
    recomputing against a log that has since grown (decision #14).
    """
    distances: dict[str, dict[str, int | None]]   # axis -> value -> distance
```

The caller dispatches with `REGISTRY[spec.family](profile, spec.params)` — which
is why the family signature stays a plain dict and the two sides compose.

- [ ] **Step 1: Write the failing weight tests — the whole of decision #15**

```python
import pytest
from melete.selection import weight, FLOOR

def test_never_used_weighs_one():
    assert weight(None, 14) == 1.0

def test_used_today_is_the_floor_not_zero():
    assert weight(0, 14) == FLOOR

def test_no_weight_is_ever_zero():
    assert all(weight(d, 14) >= FLOOR for d in range(0, 100))

def test_weight_is_monotonic_non_decreasing():
    ws = [weight(d, 14) for d in range(0, 30)]
    assert ws == sorted(ws)

def test_saturates_at_the_horizon():
    assert weight(14, 14) == 1.0
    assert weight(99, 14) == 1.0
```

- [ ] **Step 2: Confirm failure**

- [ ] **Step 3: Implement the weight function**

```python
FLOOR = 0.05
DEFAULT_HORIZON = 14


def weight(sessions_since: int | None, horizon: int = DEFAULT_HORIZON) -> float:
    """spec section 9. One expression, floor applied last. Never zero."""
    if sessions_since is None:
        return 1.0
    return max(FLOOR, min(1.0, sessions_since / horizon))
```

- [ ] **Step 4: Confirm pass**

- [ ] **Step 5: Write the pathological-pool test (the crash decision #15 fixed)**

```python
def test_exhausted_pool_still_selects():
    """shape scales=3 over a two-element pool: every candidate at distance 0."""
    config = make_config(shape={"scales": 3},
                         scale_types=["ionian", "dorian"])
    picks = select(config, history=[], rng=Random(0))
    assert len(picks) == 3          # must not divide by zero
```

- [ ] **Step 6: Implement per-axis sampling, the validity gate, and within-session pushdown**

Each axis sampled independently and weighted by recency. Each sampled spec is
gated on the instrument profile **and** `max_notes` (spec §9, decision #17);
invalid specs are resampled to a bounded retry count; exhausting retries raises
a loud error naming the over-constrained axis. As each exercise is selected, its
values are pushed onto history at distance 0.

- [ ] **Step 7: Write the 200-session statistical test (spec §14)**

```python
def test_two_hundred_sessions_distribute_near_uniformly():
    history, rng = [], Random(12345)
    counts = Counter()
    for _ in range(200):
        picks = select(config, history, rng)
        for spec, _ in picks:
            counts[spec.params["root"]] += 1
        history.append(picks)
    lo, hi = min(counts.values()), max(counts.values())
    assert hi / lo < 2.0, f"clumping: {counts}"
```

- [ ] **Step 8: Commit**

## Task B11: `config.py`

**Files:** `src/melete/config.py`, `tests/test_config.py`

**Interfaces:** Consumes `vocabulary`, `instrument`. Produces
`load(path) -> Config`.

- [ ] **Step 1: Write the failing tests — spec §13's error contract**

```python
def test_misspelled_key_fails_loudly_and_names_accepted_values():
    with pytest.raises(ValueError) as exc:
        load_string('[pool.scales]\nscale_types = ["dorain"]')
    assert "dorain" in str(exc.value)
    assert "dorian" in str(exc.value)      # from the registry

def test_never_falls_back_to_a_default_for_an_unknown_key():
    with pytest.raises(ValueError):
        load_string("[session]\ncont = 5")   # typo of `count`

def test_max_notes_defaults_are_present():
    assert load_string("").session.max_notes == 96

def test_explicit_tuning_is_accepted():
    cfg = load_string(
        '[instrument]\n'
        'profile = { name = "drop_d", tuning = [26, 33, 38, 43], '
        'fret_count = 20 }'
    )
    assert cfg.instrument.tuning == (26, 33, 38, 43)
    assert cfg.instrument.fret_count == 20

def test_non_ascending_tuning_is_a_load_error_never_re_sorted():
    """decision #22: sorting would engrave the wrong instrument convincingly."""
    with pytest.raises(ValueError) as exc:
        load_string(
            '[instrument]\n'
            'profile = { name = "bad", tuning = [33, 26, 38], fret_count = 20 }'
        )
    assert "1" in str(exc.value)        # names the offending index

def test_explicit_tuning_without_fret_count_is_rejected():
    with pytest.raises(ValueError):
        load_string(
            '[instrument]\n'
            'profile = { name = "bad", tuning = [26, 33, 38, 43] }'
        )

def test_family_tempo_override():
    cfg = load_string("[pool.chromatic]\ntempo = [50, 70]")
    assert cfg.pool["chromatic"].tempo == (50, 70)

def test_family_tempo_defaults_when_unset():
    assert load_string("").pool["chromatic"].tempo == (60, 80)
```

- [ ] **Step 2: Run and confirm failure**

- [ ] **Step 3: Implement against `vocabulary.accepted`**

Every rejection message lists accepted values pulled from the registry, never
a hardcoded list (Global Constraints).

- [ ] **Step 4: Run and confirm pass**

- [ ] **Step 5: REFACTOR, then commit**

## Task B12: `lilypond/emit.py` — Score to LilyPond text

**Files:** `src/melete/lilypond/__init__.py`, `src/melete/lilypond/emit.py`,
`tests/lilypond/test_emit.py`, `tests/lilypond/golden/*.ly`

**Interfaces:** Consumes `score`, `vocabulary`. Produces
`emit_score(score, staves) -> str` and `emit_book(scores, cover) -> str`.

Golden-file tests on the emitted **text**. No rendering (spec §14).

- [ ] **Step 1: Write the duration-token test first — it is the fiddliest piece**

```python
from fractions import Fraction
from melete.lilypond.emit import duration_token

def test_plain_durations():
    assert duration_token(Fraction(1, 4)) == "4"
    assert duration_token(Fraction(1, 8)) == "8"
    assert duration_token(Fraction(1, 16)) == "16"

def test_dotted_durations():
    assert duration_token(Fraction(3, 8)) == "4."
    assert duration_token(Fraction(7, 16)) == "4.."

def test_unrepresentable_duration_raises():
    with pytest.raises(ValueError):
        duration_token(Fraction(1, 12))     # no twelfth note exists
```

That last test is decision #16 enforced at the boundary.

- [ ] **Step 2: Write the string-numbering test — the off-by-one that will bite**

LilyPond numbers strings with **1 as the highest**; our IR indexes **0 as the
lowest**. The mapping is not identity:

```python
def test_string_index_maps_to_lilypond_string_number():
    # bass6: IR index 5 (highest, C) is LilyPond string 1.
    assert lily_string_number(index=5, string_count=6) == 1
    assert lily_string_number(index=0, string_count=6) == 6
```

- [ ] **Step 3: Confirm failure, implement both, confirm pass**

```python
def duration_token(written: Fraction) -> str:
    for dots, mult in ((0, Fraction(1)), (1, Fraction(3, 2)), (2, Fraction(7, 4))):
        base = written / mult
        if base.numerator == 1 and (base.denominator & (base.denominator - 1)) == 0:
            return f"{base.denominator}" + "." * dots
    raise ValueError(f"{written} is not a representable note value")


def lily_string_number(index: int, string_count: int) -> int:
    return string_count - index
```

- [ ] **Step 4: Write the golden-file test for `staves = "both"`**

```python
def test_both_mode_golden(tmp_path):
    out = emit_score(sample_score(), staves="both")
    assert out == (GOLDEN / "both.ly").read_text()
```

- [ ] **Step 5: Write the golden-file test for `staves = "tab"` — the emitter branch**

Spec §10: `TabStaff` suppresses stems and beams by default, so `tab` mode must
**explicitly enable rhythm display** or the exercise is unreadable. This is a
branch, not a flag:

```python
def test_tab_only_mode_enables_full_rhythm_notation():
    out = emit_score(sample_score(), staves="tab")
    assert "\\tabFullNotation" in out

def test_both_mode_does_not_enable_it():
    out = emit_score(sample_score(), staves="both")
    assert "\\tabFullNotation" not in out
```

- [ ] **Step 6: Assert the notation conventions**

```python
def test_no_key_signature_and_explicit_accidentals():
    out = emit_score(sample_score(), staves="both")
    assert "\\accidentalStyle forget" in out

def test_exercise_ends_with_a_final_barline():
    assert '\\bar "|."' in emit_score(sample_score(), staves="both")

def test_instruction_never_reaches_an_exercise_page():
    s = sample_score(instruction="Keep the plucking hand even.")
    assert "plucking" not in emit_score(s, staves="both")
```

The last one is spec §12: `instruction` is cover-page-only.

- [ ] **Step 7: Implement `emit_book` with the cover page**

The cover page is a LilyPond markup bookpart inside the same book, so the
session is one document from one render call (spec §12). Cover entries are
generated from each score's `params` via `vocabulary.display`.

- [ ] **Step 8: Commit**

## Task B13: `lilypond/render.py` — the blast door

**Files:** `src/melete/lilypond/render.py`, `tests/lilypond/test_render.py`

**Interfaces:** Consumes nothing but a path and text. Produces
`render(ly_text, out_dir) -> Path`.

The **only** module aware that a LilyPond binary exists (spec §4).

- [ ] **Step 1: Write the failure-path tests first — spec §13 is explicit here**

```python
def test_render_failure_keeps_the_ly_on_disk(tmp_path):
    with pytest.raises(RenderError) as exc:
        render("\\this is not lilypond", tmp_path)
    assert (tmp_path / "practice.ly").exists()      # never clean up on failure
    assert "error" in str(exc.value).lower()        # stderr surfaced verbatim

def test_missing_binary_gives_a_resolution_not_a_stack_trace(monkeypatch):
    monkeypatch.setattr("shutil.which", lambda _: None)
    with pytest.raises(RenderError) as exc:
        render("{ c4 }", tmp_path)
    assert "uv sync" in str(exc.value)
```

- [ ] **Step 2: Confirm failure, implement, confirm pass**

- [ ] **Step 3: Add the one integration test (spec §14)**

```python
@pytest.mark.integration
def test_renders_a_multi_page_pdf(tmp_path):
    pdf = render((GOLDEN / "book.ly").read_text(), tmp_path)
    assert pdf.exists()
    assert pdf.read_bytes().count(b"/Type /Page") >= 2
```

- [ ] **Step 4: Commit**

## Task B14: `session.py` — the log and replay

**Files:** `src/melete/session.py`, `tests/test_session.py`

**Interfaces:** Consumes `score`, `selection`. Produces `write(dir, picks, seed,
weight_inputs)` and `read(date) -> Session`, `history(n) -> list[Session]`.

Decision #14 lives here.

- [ ] **Step 1: Write the failing tests**

```python
def test_session_json_records_seed_config_hash_params_and_weight_inputs():
    data = json.loads((d / "session.json").read_text())
    assert set(data) >= {"seed", "config_hash", "exercises", "weight_inputs"}

def test_corrupt_history_entry_is_a_hard_error_naming_the_file():
    (d / "session.json").write_text("{ not json")
    with pytest.raises(ValueError) as exc:
        read(d)
    assert "session.json" in str(exc.value)

def test_existing_session_directory_is_refused_without_force():
    with pytest.raises(FileExistsError):
        write(existing_dir, ...)
```

- [ ] **Step 2-4: Confirm failure, implement, confirm pass**

- [ ] **Step 5: Write the replay test (spec §14) — the proof of decision #14**

```python
def test_replay_is_byte_identical_after_the_log_moves_on(tmp_path):
    first = generate_session(date="2026-08-09", root=tmp_path)
    for later in ("2026-08-10", "2026-08-11", "2026-08-12"):
        generate_session(date=later, root=tmp_path)
    again = replay_session(date="2026-08-09", root=tmp_path)
    assert again.exercises == first.exercises
```

- [ ] **Step 6: Commit**

## Task B15a: `cli.py` — `generate` and its flags

**Files:** `src/melete/cli.py`, `tests/test_cli_generate.py`

**Interfaces:** Consumes everything in Phase B. Produces the console entry
point `melete` and the `generate` subcommand.

Covers every flag in spec §11's `generate` line: `--date`, `--seed`,
`--dry-run`, `--staves`, `--count`, `--force`, `--split`.

- [ ] **Step 1: Write the failing tests, one per documented flag**

```python
from datetime import date
import pytest

def test_dry_run_prints_selections_and_renders_nothing(tmp_path):
    result = run(["generate", "--dry-run"], cwd=tmp_path)
    assert result.exit_code == 0
    assert not list(tmp_path.glob("**/*.pdf"))
    assert "Dorian" in result.stdout or "chromatic" in result.stdout

def test_generate_refuses_an_existing_session_without_force(tmp_path):
    run(["generate"], cwd=tmp_path)
    assert run(["generate"], cwd=tmp_path).exit_code != 0
    assert run(["generate", "--force"], cwd=tmp_path).exit_code == 0

def test_date_writes_to_that_dated_directory(tmp_path):
    run(["generate", "--date", "2026-08-10"], cwd=tmp_path)
    assert (tmp_path / "sessions" / "2026-08-10").is_dir()

def test_count_overrides_the_configured_exercise_count(tmp_path):
    run(["generate", "--count", "6"], cwd=tmp_path)
    session = read_session(tmp_path, date.today())
    assert len(session.exercises) == 6

def test_staves_override_reaches_the_emitter(tmp_path):
    run(["generate", "--staves", "tab"], cwd=tmp_path)
    src = (tmp_path / "sessions" / date.today().isoformat() / "src")
    assert "\\tabFullNotation" in (src / "book.ly").read_text()
```

- [ ] **Step 2: Write the `--split` test — the flag with no implementation today**

```python
def test_split_emits_one_pdf_per_exercise_plus_the_book(tmp_path):
    run(["generate", "--count", "3", "--split"], cwd=tmp_path)
    d = tmp_path / "sessions" / date.today().isoformat()
    assert (d / "practice.pdf").exists()
    assert len(list(d.glob("exercise-*.pdf"))) == 3

def test_without_split_only_the_combined_pdf_is_written(tmp_path):
    run(["generate", "--count", "3"], cwd=tmp_path)
    d = tmp_path / "sessions" / date.today().isoformat()
    assert not list(d.glob("exercise-*.pdf"))
```

- [ ] **Step 3: Run and confirm every test fails**

- [ ] **Step 4: Implement**

`--split` renders each exercise's `Score` through `emit_score` in addition to
the combined book — the pieces already exist from B12, this wires them.

- [ ] **Step 5: Run and confirm pass**

- [ ] **Step 6: Add the end-to-end smoke test (spec §14)**

The `src/` assertion counts sources rather than checking the directory exists —
§12 requires LilyPond source *per exercise and for the book*, and an
`is_dir()` check cannot fail when that output is missing:

```python
@pytest.mark.integration
def test_end_to_end_produces_a_practice_pdf(tmp_path):
    assert run(["generate", "--count", "5"], cwd=tmp_path).exit_code == 0
    d = tmp_path / "sessions" / date.today().isoformat()
    assert (d / "practice.pdf").exists()
    assert (d / "session.json").exists()

    ly_files = list((d / "src").glob("*.ly"))
    assert len(ly_files) == 6          # five exercises, plus the book
```

- [ ] **Step 7: REFACTOR, then commit**

## Task B15b: `cli.py` — `replay`, `show`, `families`, `vocabulary`

**Files:** `src/melete/cli.py` (extend), `tests/test_cli_query.py`

**Blocked-by:** B15a

The read-only commands. Separated from B15a because a reviewer can sensibly
accept generation and reject these, which is the test for whether a split earns
its keep.

- [ ] **Step 1: Write the failing tests**

```python
def test_replay_reproduces_a_past_session(tmp_path):
    run(["generate", "--date", "2026-08-09"], cwd=tmp_path)
    first = read_session(tmp_path, "2026-08-09")
    for later in ("2026-08-10", "2026-08-11"):
        run(["generate", "--date", later], cwd=tmp_path)
    assert run(["replay", "2026-08-09"], cwd=tmp_path).exit_code == 0
    assert read_session(tmp_path, "2026-08-09").exercises == first.exercises

def test_show_summarizes_a_past_session(tmp_path):
    run(["generate", "--date", "2026-08-09"], cwd=tmp_path)
    out = run(["show", "2026-08-09"], cwd=tmp_path).stdout
    assert "2026-08-09" in out
    assert "bass6" in out

def test_show_on_a_missing_session_fails_loudly(tmp_path):
    result = run(["show", "1999-01-01"], cwd=tmp_path)
    assert result.exit_code != 0
    assert "1999-01-01" in result.stderr

def test_families_lists_all_four_with_their_axes():
    out = run(["families"]).stdout
    for family in ("chromatic", "scales", "arpeggios", "intervals"):
        assert family in out
    assert "permutation" in out          # a chromatic axis

def test_vocabulary_lists_every_axis():
    out = run(["vocabulary"]).stdout
    for axis in AXES:
        assert axis in out
```

- [ ] **Step 2: Run and confirm failure**
- [ ] **Step 3: Implement**
- [ ] **Step 4: Run and confirm pass**
- [ ] **Step 5: REFACTOR, then commit**

---

# Phase C — Proof

## Task C1: Deploy melete into daily use

**Repo:** `mnemosys-project/melete` · **Kind:** `deployment`
**Blocked-by:** B15b

Merged is not deployed (per `epic-create`). This task's closure **is** the
"melete is usable every morning" signal that C2 depends on. It implements spec
§15 *Installation for daily use*: development happens in the container, daily
use does not.

**Precondition self-check:** `vrg-gh api repos/mnemosys-project/melete/commits/develop`
includes the B15b merge commit.

**Procedure:** `uv tool install` melete from the repository onto the host —
outside `vrg-container-run`, since the morning command must not require a
container. Place a `config.toml` with the `bass6` profile. Run `melete generate`
once successfully.

**Acceptance:** `melete generate` produces `sessions/<today>/practice.pdf` on the
author's own machine. Record `Outcome: SUCCESS` or `FAILURE` as a comment.

## Task C2: Validate the practice sheet on paper

**Repo:** `mnemosys-project/melete` · **Kind:** `validation`
**Blocked-by:** C1

The pipeline's own tests cannot check this: the success criterion in spec §1 is
that the sheet is *good enough to hand to a bass instructor*, which is a
judgment about printed output.

**Procedure:** generate sheets on five consecutive days. Print each.

**Acceptance, all required:**
- Every sheet prints as one document, cover page first (spec §12).
- Tablature and notation agree on every exercise — spot-check the central
  invariant by eye on a sample.
- Across the five days, keys and modes are visibly varied, with no value
  repeating on consecutive days (spec §9).
- No exercise is truncated mid-pattern (spec §7).
- The author judges at least one sheet fit to hand to his instructor.

Record `Outcome: SUCCESS` or `FAILURE` with the five dates as a comment. On
failure the task stays open and the epic stays open.

---

# Task Summary

| # | Task | Repo | Blocked-by |
|---|---|---|---|
| — | Documentation (#2) | `.github` | — |
| A1 | Org metadata and health files | `.github` | A3 |
| A2 | Epic document format standards | `.github` | — |
| A3 | Banner image and design record | `.github` | — |
| A4 | Create `docs` repo and site | `docs` | human gate |
| A5 | Create `melete` repo | `melete` | human gate |
| B1 | `theory.py` | `melete` | A5 |
| B2 | `instrument.py` | `melete` | A5 |
| B3 | `vocabulary.py` | `melete` | B1 |
| B4 | `score.py` | `melete` | B2 |
| B5 | `families/chromatic.py` | `melete` | B4 |
| B6 | `families/scales.py` | `melete` | B4, B3 |
| B6a | shared family helpers (extracted) | `melete` | B5, B6 |
| B7 | `families/arpeggios.py` | `melete` | B4, B3, B6a |
| B8 | `families/intervals.py` | `melete` | B4, B3, B6a |
| B9 | `rhythm.py` | `melete` | B4 |
| B10 | `selection.py` | `melete` | B5-B9 |
| B11 | `config.py` | `melete` | B3 |
| B12 | `lilypond/emit.py` | `melete` | B4 |
| B13 | `lilypond/render.py` | `melete` | — |
| B14 | `session.py` | `melete` | B10 |
| B15a | `cli.py` — generate and its flags | `melete` | B11-B14 |
| B15b | `cli.py` — replay, show, families, vocabulary | `melete` | B15a |
| C1 | Deploy into daily use | `melete` | B15b |
| C2 | Validate on paper | `melete` | C1 |
| — | Documentation review (#3) | `.github` | all above |
| — | Retrospective (#4) | `.github` | #3 |

B1/B2 and B13 have no dependency on each other and can run in parallel. B5 and
B6 are independent once B4 lands; B6a extracts their shared helpers afterwards,
so B7 and B8 follow it rather than racing it. Extracting on the second
occurrence rather than the first is deliberate — writing `_shared.py` before two
families exist would be the anticipatory abstraction the REFACTOR step warns
against.

## Spec Coverage

| Spec section | Task |
|---|---|
| §4 Architecture / module layout | B1–B15b (one task per module) |
| §4 `ExerciseSpec`, `WeightInputs` placement | B10 Step 0 |
| §5 Instrument model, fret counts | B2 |
| §5 User-defined tunings | B11 |
| §6 Score IR, written durations | B4 |
| §7 Four families, length bound, terminal bar | B5–B8, B10, B12 |
| §7 Per-family tempo defaults | B5–B8, B11 |
| §8 Rhythm modifier, all four axes | B9 |
| §9 Selection, weighting, validity, replay | B10, B14 |
| §10 Configuration | B11 |
| §11 CLI — `generate` and flags | B15a |
| §11 CLI — `replay`, `show`, `families`, `vocabulary` | B15b |
| §12 Output, cover page, session log, per-exercise sources | B12, B14, B15a |
| §13 Error handling, vocabulary registry | B3, B11, B13, B14 |
| §14 Testing strategy | every task's test steps |
| §15 Repo and Vergil integration | A5 |
| *Org banner and its design record* | A3 |
| §15 Installation for daily use | C1 |
| *Document formats* | A2 |
