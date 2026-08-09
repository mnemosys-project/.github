# Melete — Practice Exercise Generator

**Design specification, v1.0**
**Date:** 2026-08-09
**Org:** `mnemosys-project`
**Repository:** `mnemosys-project/melete`
**Epic:** [`mnemosys-project/.github#1`](https://github.com/mnemosys-project/.github/issues/1)

## Table of Contents

- [Epic Scope and Filing Order](#epic-scope-and-filing-order)
- [1. Overview](#1-overview)
- [2. Name and Lineage](#2-name-and-lineage)
- [3. Scope](#3-scope)
- [4. Architecture](#4-architecture)
- [5. Instrument Model](#5-instrument-model)
- [6. The Score IR](#6-the-score-ir)
- [7. Exercise Families](#7-exercise-families)
- [8. Rhythm Modifier](#8-rhythm-modifier)
- [9. Coverage-Aware Selection](#9-coverage-aware-selection)
- [10. Configuration](#10-configuration)
- [11. Command-Line Interface](#11-command-line-interface)
- [12. Output and Session Log](#12-output-and-session-log)
- [13. Error Handling](#13-error-handling)
- [14. Testing Strategy](#14-testing-strategy)
- [15. Repository and Vergil Integration](#15-repository-and-vergil-integration)
- [16. Recorded Decisions](#16-recorded-decisions)
- [17. Deferred to v2](#17-deferred-to-v2)

## Epic Scope and Filing Order

This section is specific to epic #1 and is stated explicitly rather than left to
be inferred from §15 and decision #12.

### Organization bootstrap and melete v1 are one epic

The scope of epic #1 is **both** the bootstrap of the `mnemosys-project`
organization **and** the delivery of `melete` v1. This is a deliberate scope
decision, not an accident of sequencing.

`mnemosys-project` was empty as of 2026-08-09 — no repositories at all. Vergil's
first invariant is that epics live in the organization's `.github`, so the melete
epic had nowhere to live until that repository existed. The bootstrap is
therefore a genuine prerequisite of the deliverable rather than adjacent work
that merely happens to be concurrent.

Splitting them into two epics would add ceremony without adding clarity. It would
also leave the bootstrap epic without a deliverable to justify it: an
organization scaffolded for its own sake, with the thing it exists to hold filed
somewhere else. The bootstrap earns its place by being the first work of the epic
that ships melete.

This is recorded as decision #12 in §16.

### This epic was filed retroactively, by design

Epic #1 was filed **into the repository it describes creating**. Read in
isolation that looks like a process violation. It is not, and the exception
applies exactly once.

The epic home must exist before an epic can be filed into it. Bootstrapping an
organization from zero therefore has an unavoidable ordering exception at its
root: `mnemosys-project/.github` had to be created first, outside the normal
flow, so that the epic describing its own creation had somewhere to live.

Two consequences follow, both carried forward deliberately:

- **The groundwork was performed on the host, outside the VM sandbox**, under
  direct human authorization, because organization and repository creation is a
  human act. Those artifacts — the repository, the GitHub App, the org secrets,
  and triage issue `vergil-project/.github#270` — are authored by the **human**
  account rather than the user agent.
- **Everything from epic #1 forward is agent-authored inside the sandbox.** The
  human-authored artifacts above are the complete and closed set.

No future epic in this organization should reproduce this ordering. Once
`.github` exists, the normal flow applies without exception.

## 1. Overview

Melete is a standalone command-line tool that generates daily bass practice
sheets. It reads a configuration file describing an instrument and a pool of
exercise parameters, selects a small set of exercises with deliberate variety,
and renders them as engraving-quality PDFs containing tablature and standard
notation.

The success criterion is practical: run one command each morning and get a
printable practice sheet good enough to hand to a bass instructor.

Melete descends conceptually from the MNEMOSYS project
(`wphillipmoore/mnemosys-core`, February 2026), which specified a deterministic,
fatigue-aware framework for generating structured practice sessions. That project
became an infrastructure exercise — SQLAlchemy, Alembic, FastAPI, AWS — and was
set aside. Melete keeps the conceptual seeds and discards all of the
infrastructure.

Inherited from MNEMOSYS:

- The canonical exercise library and its domain taxonomy
- Overload dimensions: orthogonal parameter axes that generate variants from a
  single canonical exercise
- The principle that exercises are abstract and instruments are configurations
- Deterministic, explainable generation

Explicitly not inherited: the database, the API, the container and cloud
infrastructure, the fatigue and mastery state model.

## 2. Name and Lineage

In the older Boeotian tradition recorded by Pausanias (*Description of Greece*
9.29.2), three elder Muses were worshipped on Mount Helicon:

- **Mneme** (Μνήμη) — memory
- **Aoede** (Ἀοιδή) — song
- **Melete** (Μελέτη) — practice, study, deliberate exercise

The organization carries the memory name. Each tool under it is one of the
Muses.

- **`melete`** (MEL-uh-tee) — this tool. Practice exercise generation.
- **`aoede`** (ay-EE-dee) — **reserved**. The future repertoire-management tool,
  descended from the MNEMOSYS RPM design. Not in scope here; the name is claimed
  so it is not rediscovered later.

Both names are available on PyPI as of 2026-08-09. `mneme`, `euterpe`, and
`kithara` are taken.

## 3. Scope

### In scope for v1

- Four generative exercise families, each deeply parameterized
- Rhythm applied as a cross-cutting modifier over all families
- Configurable instrument profiles: 4-, 5-, and 6-string bass
- Coverage-aware random selection that spreads across parameter axes
- LilyPond rendering to a single combined PDF per day
- A cover page summarizing the day's session
- A machine-readable session log recording every parameter of every pick

### Non-goals

These are excluded deliberately, not overlooked:

- **Progression, overload advancement, mastery estimates, rolling volume,
  fatigue budgeting** — deferred to v2
- **Practice logging, note-taking, outcome recording** — deferred to v2
- **Database, server, API, web UI** — permanently out of scope for this tool
- **Audio, MIDI, playback**
- **Repertoire management** — that is `aoede`
- **GUI** — the configuration file is the interface

### Audience

The primary user is the author, who is a bass **student**. His instructor is a
separate person whose review is a feedback gate once a working version exists.
Design decisions favor the author's needs (he reads tablature, not standard
notation) while keeping both renderings available, because instruction materials
frequently require both.

## 4. Architecture

### Pipeline

```
config.toml
    |
    v
Selector ---------> ExerciseSpec ------> Family generator ------> Score IR
(coverage-aware,     (family +            (pure function          (pitch +
 reads session log)   concrete params)     per family)             string/fret +
                                                                   duration)
                                                                       |
                                                                       v
                                                              Rhythm modifier
                                                              (Score -> Score)
                                                                       |
                                                                       v
                                                              LilyPond emitter
                                                              (Staff / TabStaff)
                                                                       |
                                                                       v
                                                              Renderer -> PDF
                                                                       |
                                                                       v
                                                              Session writer
```

### Module layout

```
src/melete/
  instrument.py    InstrumentProfile and fretboard queries
  score.py         The IR: Note, Tuplet, Voice, Score. Pure data.
  theory.py        Pitch, interval, scale, and chord math (12-TET integers)
  families/
    __init__.py    Family registry
    chromatic.py
    scales.py
    arpeggios.py
    intervals.py
  rhythm.py        Cross-cutting modifier: Score -> Score
  selection.py     Coverage-aware sampling; reads session history
  lilypond/
    emit.py        Score -> LilyPond source text
    render.py      Adapter over the LilyPond binary
  session.py       Writes and reads sessions/YYYY-MM-DD/
  config.py        Loads and validates config.toml
  cli.py           Argument parsing; wires the pipeline
```

### The two load-bearing boundaries

**`score.py` is the seam.** Families produce a `Score`; the emitter consumes
one. Neither imports the other. A family never learns that LilyPond exists; the
emitter never learns what a Dorian mode is. Every family is therefore a pure
function testable without rendering anything.

**`render.py` is the blast door.** It is the only module aware that a LilyPond
binary exists. If the LilyPond distribution changes, exactly one file changes.

`theory.py` and `instrument.py` are the most reused and the most exhaustively
testable modules. They are built first.

## 5. Instrument Model

Exercises reference string **indices** (0 … n-1, low to high) and interval
relationships. No exercise hardcodes a string name, a tuning, or a string count.
This is the MNEMOSYS principle that instruments are configurations, and it is
what makes multiple bass profiles nearly free.

```python
@dataclass(frozen=True)
class InstrumentProfile:
    name: str
    tuning: tuple[int, ...]   # absolute pitches, low to high
    fret_count: int
```

Built-in profiles:

| Profile  | Strings | Tuning  | Notes             |
|----------|---------|---------|-------------------|
| `bass4`  | 4       | E A D G | standard          |
| `bass5`  | 5       | B E A D G | standard        |
| `bass6`  | 6       | B E A D G C | **default**   |

Users may define explicit tunings in configuration.

A generated exercise specification is validated against the active profile. A
specification requiring string indices or a fret range the profile cannot supply
is invalid and is resampled — for example, a six-string-spanning skip pattern on
`bass4`.

## 6. The Score IR

```python
@dataclass(frozen=True)
class Note:
    pitch: int              # absolute semitones, C4 = 60
    string: int             # index into tuning, 0 = lowest
    fret: int               # 0 = open
    duration: Fraction      # 1/4 = quarter note
    finger: int | None      # left hand, 1-4; None = unspecified
    accent: bool

@dataclass(frozen=True)
class Tuplet:
    ratio: tuple[int, int]  # (3, 2) = triplet
    notes: list[Note]

Voice = list[Note | Tuplet]

@dataclass(frozen=True)
class Score:
    title: str
    instruction: str                  # one-line focus cue (cover page only)
    instrument: InstrumentProfile
    time_signature: tuple[int, int]
    tempo_range: tuple[int, int]
    voice: Voice
    params: dict                      # exact parameters that produced this
```

Four decisions embedded here:

**Pitch and position are both stored, never derived at render time.** A family
decides both which note to play and where to play it. Fretboard position is
musical information, not a rendering detail. This is the entire justification for
a hand-rolled IR over an existing library.

**`finger` is first-class.** In chromatic permutation work the fingering *is*
the exercise. LilyPond renders fingering marks natively.

**One level of nesting, no more.** A voice is a flat sequence with `Tuplet` as
the single nested form, mapping directly onto LilyPond's `\tuplet 3/2 { ... }`.
Measures are **not** modeled — durations imply barlines and LilyPond inserts
them. Grouping against the meter (fives over 4/4) therefore requires no
bar-splitting logic.

**`params` travels inside the Score.** The session log receives the exact
parameter dictionary that produced each exercise, so any sheet is reproducible
and the selector reads history without a separate bookkeeping path.

## 7. Exercise Families

Each family is a pure function `params -> Score`. No I/O and no randomness: the
selector chooses parameters, the family realizes them.

The four families were chosen from the MNEMOSYS canonical library, filtered for
those expressible as concrete notes with fretboard positions.

### `chromatic` — finger-independence permutations

Derived from MNEMOSYS T1.

| Axis               | Values                                        |
|--------------------|-----------------------------------------------|
| `permutation`      | the 24 orderings of fingers 1-2-3-4           |
| `start_string`     | string index                                  |
| `start_fret`       | fret number                                   |
| `direction`        | ascending, descending, both                   |
| `string_traversal` | adjacent, skip-1, single-string               |
| `shift`            | none, +1 fret per cycle, +1 position per cycle |
| `span`             | number of strings covered                     |

### `scales` — modes and scale patterns

Derived from MNEMOSYS H1, H2, H3, H5.

| Axis            | Values                                                      |
|-----------------|-------------------------------------------------------------|
| `root`          | 12 pitch classes                                            |
| `scale_type`    | 7 major modes, 7 melodic minor modes, 7 harmonic minor modes, major and minor pentatonic, blues, whole-tone, two diminished (~28) |
| `traversal`     | positional (boxed), three-notes-per-string, one-octave-per-string, single-string linear |
| `string_set`    | contiguous or non-contiguous subsets                        |
| `pattern`       | straight, thirds, fourths, groups-of-3, groups-of-4, numeric permutations (1-2-3-5) |
| `range_octaves` | 1, 2, 3                                                     |
| `direction`     | up, down, up-down                                           |

This family alone yields roughly 24,000 variants before rhythm is applied.

### `arpeggios` — chord tones

Derived from MNEMOSYS A1, A2.

| Axis            | Values                                                      |
|-----------------|-------------------------------------------------------------|
| `root`          | 12 pitch classes                                            |
| `quality`       | maj, min, dim, aug, maj7, min7, dom7, m7b5, dim7, minMaj7, 6, m6 |
| `inversion`     | root, first, second, third                                  |
| `traversal`     | positional, across-strings, single-string                   |
| `string_set`    | subsets                                                     |
| `pattern`       | straight, 1-3-5-3, broken, sweep-ordered                    |
| `range_octaves` | 1, 2, 3                                                     |
| `direction`     | up, down, up-down                                           |

### `intervals` — interval and string-skipping sequences

Derived from MNEMOSYS P2, T4. This family exists because tablature makes string
topology expressible; these exercises cannot be described by pitch alone.

| Axis           | Values                                       |
|----------------|----------------------------------------------|
| `interval`     | 2nd through 10th                             |
| `context`      | chromatic, or diatonic within root + scale   |
| `string_skip`  | 0 (adjacent), 1, 2                           |
| `string_set`   | subsets                                      |
| `direction`    | up, down, up-down                            |
| `pattern`      | ascending pairs, descending pairs, alternating |

### Exercise length

One exercise is **one complete cycle of its pattern** — a full two-octave scale
traversal, a full permutation cycle, whatever the pattern's natural unit is. An
exercise is never truncated mid-pattern.

## 8. Rhythm Modifier

Rhythm is **not** a fifth family. It is a cross-cutting modifier of type
`Score -> Score` that applies to all four families, following the MNEMOSYS
principle that overload dimensions are orthogonal. Treating rhythm as a
parameter axis multiplies the variant space rather than adding to it, and keeps
the rhythm logic in one shared component instead of four copies.

Derived from MNEMOSYS R1, R2, R3, R4.

| Axis                 | Values                                                  |
|----------------------|---------------------------------------------------------|
| `subdivision`        | quarters, eighths, triplet eighths, sixteenths, sextuplets, quintuplets |
| `time_signature`     | 4/4, 3/4, 5/4, 6/8, 7/8, 12/8                           |
| `accent_pattern`     | none, every-3, every-5, displaced-by-one                 |
| `note_value_pattern` | straight, long-short, short-long                         |

## 9. Coverage-Aware Selection

### The problem

Drawing exercises uniformly at random from the pool treats each specification as
a single draw, so nothing prevents three D-rooted exercises in one session or a
week of nothing but Dorian. Clumping is what uniform randomness does. The
requirement is variety that *feels* varied.

### The mechanism

Every parameter is an axis: `family`, `root`, `scale_type`, `traversal`,
`pattern`, `string_set`, `subdivision`, `accent_pattern`. **Each axis is sampled
independently, weighted by recency.**

For each axis, the selector reads the last N sessions from the log and computes,
for each candidate value, how many sessions have passed since it was last used:

```
w(value) = 1.0                                       # never used
w(value) = min(1.0, sessions_since / horizon)        # horizon default 14
w(value) = 0.05                                      # used today or yesterday
```

Values are then drawn proportional to their weights. Because axes are drawn
independently, roots spread across the chromatic scale on their own schedule
while modes spread on theirs. That independence is what produces "a mix of keys
and a mix of modes" rather than one lucky draw.

The floor of 0.05 rather than 0.0 is deliberate: a recently used value becomes
unlikely, never impossible.

### Within-session diversity

As each exercise is selected, its values are pushed onto the history at distance
zero. The next slot in the same session therefore actively avoids them. D Dorian
and D Phrygian will not appear on the same page.

### Session shape

The mix of families is configuration, not chance. The user declares a shape —
for example one `chromatic`, two `scales`, one `arpeggios`, one `intervals` —
and the selector fills each slot. Leaving the shape unset weights families
instead.

### Validity

Validity is a hard gate, not a weight. Each sampled specification is checked
against the active instrument profile. Invalid specifications are resampled up
to a bounded retry count; exhausting retries is a loud error naming the
over-constrained axis, never a silent fallback.

### Determinism

The seed derives from the date plus a hash of the configuration and is written
into `session.json`. `melete generate --seed <n>` reproduces a day exactly.
Randomness is real but never irreproducible.

### Tunability

The weighting function and horizon live in one module behind one entry point.
Replacing linear decay with exponential decay must be a small, local change. The
author intends to tune this empirically through daily use.

## 10. Configuration

A single TOML file. The `[pool.*]` sections are the primary tuning surface.

```toml
[instrument]
profile = "bass6"                    # bass4 | bass5 | bass6, or explicit tuning

[output]
staves = "both"                      # both | tab | notation
key_signatures = false               # explicit accidentals throughout

[session]
count = 5
horizon = 14
shape = { chromatic = 1, scales = 2, arpeggios = 1, intervals = 1 }

[pool.scales]
roots = "all"
scale_types = ["ionian", "dorian", "phrygian", "major_pentatonic", "blues"]
patterns = ["straight", "thirds", "groups_of_3", "groups_of_4"]
traversals = ["positional", "three_note_per_string"]
octaves = [1, 2]

[pool.rhythm]
subdivisions = ["eighth", "triplet_eighth", "sixteenth"]
accent_patterns = ["none", "every_3"]
```

### Notation conventions

**No key signatures by default.** Modal exercises are notated in C with explicit
accidentals throughout. A key signature implies a tonal center that modal
practice material should not assert, and explicit accidentals force the reader to
see each altered tone. Configurable via `key_signatures`.

**Staff mode is a switch.** `both` (default), `tab`, or `notation`.

This carries a genuine emitter consequence: LilyPond's `TabStaff` suppresses
stems and beams by default, assuming a notation staff above supplies the rhythm.
In `tab` mode the emitter must therefore **explicitly enable rhythm display** —
Guitar Pro-style tablature with stems — or the exercise is unreadable. In `both`
mode plain tablature is correct. This is a branch in the emitter, not a flag.

## 11. Command-Line Interface

```
melete generate                  # today's session
melete generate --date 2026-08-10
melete generate --seed 12345     # reproduce a session exactly
melete generate --dry-run        # print selections, render nothing
melete generate --staves tab     # override staff mode for one run
melete generate --count 6
melete generate --force          # overwrite an existing session directory
melete generate --split          # also emit one PDF per exercise
melete show 2026-08-09           # summarize a past session
melete families                  # list families and their parameter axes
```

## 12. Output and Session Log

```
sessions/2026-08-09/
  practice.pdf        cover page + exercises, one printable document
  session.json        every parameter of every selection, plus the seed
  src/                generated LilyPond source, per exercise and for the book
```

### The combined document

A day's output is **one PDF**: a cover page followed by the exercises. This is
the printing unit.

### The cover page

All instructional prose lives on the cover page, not on the engraved exercises.
Text overlaid on notation clutters the page and competes with the notes.

The cover page is rendered as a LilyPond markup page inside the same book, so
the session remains one document produced by one render call with no additional
dependency for text-to-PDF conversion.

It carries the date, the instrument profile, and one entry per exercise. Each
entry is generated from that exercise's `params` dictionary, rendered into plain
language, optionally followed by the `Score.instruction` focus cue when the
family supplies one:

> 3. D Dorian, three-notes-per-string, ascending thirds, strings 2-5, triplet
>    eighths, 80-100 bpm
>    *Keep the plucking hand even through the string crossings.*

Exercise pages themselves carry only a title and minimal annotation. The
`instruction` field is never rendered onto an exercise page.

### The session log

`session.json` records the seed, the configuration hash, and the full parameter
dictionary for every exercise. It is the history the selector reads back. It is
human-readable, git-committable, and hand-editable.

## 13. Error Handling

No swallowed exceptions and no fallbacks that hide errors. A failure in the
generator becomes a wrong exercise on the page, which is worse than no exercise.

| Failure | Behavior |
|---|---|
| Malformed or invalid configuration | Fail at load, naming the exact key and its accepted values. Never fall back to a default for a misspelled key. |
| Pool over-constrained | Hard error naming the axis that could not be satisfied — for example, "no valid `string_set` for `bass4` with `octaves = 3`". |
| Family emits a note outside the fretboard | A bug, not user error. Raise. |
| LilyPond render fails | Surface LilyPond's stderr verbatim and **keep the generated `.ly` on disk** for inspection and manual re-run. Never clean up on failure. |
| LilyPond binary missing | Explicit error stating the resolution, not a stack trace. |
| Session directory exists | **Refuse.** `--force` overwrites and must be asked for explicitly. |
| Corrupt session-history entry | Hard error naming the file. Silently skipping a bad entry would degrade variety invisibly. |

## 14. Testing Strategy

The architecture was chosen partly for testability: almost nothing requires a
rendered PDF.

| Component | Approach |
|---|---|
| `theory.py`, `instrument.py` | Exhaustive. All 12 roots against all ~28 scale types, verified against known interval content. Every pitch maps to valid positions on every profile. |
| Families | Property-based over a wide parameter sweep. |
| `rhythm.py` | Durations sum correctly; tuplet ratios well-formed. |
| `selection.py` | Statistical, deterministic under a fixed seed. |
| `lilypond/emit.py` | Golden-file tests on the emitted `.ly` **text**. No rendering. |
| `lilypond/render.py` | One integration test invoking LilyPond, asserting a multi-page PDF. |
| `cli.py` | One end-to-end smoke test into a temporary directory. |

### The central invariant

Every family test asserts, for every generated note:

```
note.pitch == instrument.tuning[note.string] + note.fret
```

This single property catches nearly every fretboard-positioning bug. Alongside
it: all frets within range, note count matching the declared pattern, and one
complete cycle emitted.

### The selection test

Simulate 200 sessions under a fixed seed; assert each axis distributes near
uniformly and that no value recurs within the horizon more often than the
weighting permits. This is the test that proves the clumping problem is solved.

## 15. Repository and Vergil Integration

### Organization bootstrap

`mnemosys-project` is empty as of 2026-08-09 — no repositories at all. Both
comparable active organizations (`vergil-project`, `logical-minds-foundry`)
follow the same shape: a `.github` repository holding epics and org metadata,
and a `docs` repository holding the site.

Vergil's first invariant is that all epics live in the organization's `.github`.
The melete epic therefore has nowhere to live until that repository exists.
Organization bootstrap is consequently **in scope for this epic**, as its first
work, in this order:

1. **`mnemosys-project/.github`** — epic home, label registry, org metadata,
   shared authorization and secrets context. **Repository, GitHub App, org
   secrets, and label registry are complete** (see "Epic Scope and Filing Order"
   above); the remaining org metadata files are tasks under epic #1. Profile
   modeled on `logical-minds-foundry/.github` rather than `vergil-project/.github`,
   whose `semver` + `library-release` configuration is stale on a repository that
   releases nothing.
2. **`mnemosys-project/docs`** — the org site. Both active orgs have one;
   cheaper to establish now than to retrofit.
3. **`mnemosys-project/melete`** — created via `vrg-github-repo-init`.

### Repository profile

```toml
repository-type   = "application"
versioning-scheme = "semver"
branching-model   = "library-release"
release-model     = "tagged-release"
primary-language  = "python"
```

`branching-model = "library-release"` is chosen over `application-promotion`
because melete has no deployment pipeline and no environments to promote
through, despite being an application rather than a library. To be cross-checked
against the Logical Minds Foundry repositories during bootstrap.

### Dependencies

Runtime dependency is **`lilypond` and nothing else** — the PyPI redistribution
of the LilyPond binary, installed into the project virtual environment by `uv`.

This is what allows the tool to run inside the standard Vergil `dev-python`
container with **no change to the base image** and no host-level installation.

**On the redistribution's maintenance status:** the PyPI `lilypond` package is
version 2.25.12, last published 2024-02-02, and is not actively tracking
upstream. This is accepted rather than mitigated. It is the only viable option
for installing LilyPond into a virtual environment, LilyPond's input syntax is
highly stable, and this version renders correctly.

If the package proves inadequate, the response is to **fork it and contribute
back** — it is open source, the packaging layer around the binary is thin, and
the author has the Python expertise to stand behind upstream submissions. An
apparently unmaintained dependency of this shape is an opportunity, not a
liability.

Containment remains cheap regardless: `lilypond/render.py` is the only module
aware of the binary, so migrating to a fork, a container-level system
installation, or a newer redistribution is a single-file change.

The development dependency group follows the Vergil Python standard: pytest,
ruff, mypy. Python 3.14, uv-managed. Development occurs inside
`vrg-container-run` against `dev-python`.

### Rejected alternatives

**`abjad`** (3.31) is a maintained Python API for building LilyPond files with
solved duration, tuplet, and beaming arithmetic. Rejected because it has **no
fretboard or string model** — the entire positioning layer, which is the core
value of this tool, would still be hand-written, and string assignments would
then have to be forced through abjad's indicator system. Its benefit lands on
the part that is easiest to hand-roll, while adding a large conceptual surface
and a second LilyPond version coupling.

**`music21`** (10.5.0) is a musicology analysis toolkit whose LilyPond export is
a secondary feature and weak for tablature. It would drag numpy, matplotlib,
requests, and joblib into a sheet-music renderer. Rejected.

### Standard Vergil scaffolding

`.claude/hooks/guard.sh`, `.claude/settings.json` enabling the plugin,
`docs/repository-standards.md`, `.github/workflows/ci.yml` invoking
`standards-compliance`, `.worktrees/` gitignored, and the parallel-agent
worktree section in `CLAUDE.md`.

## 16. Recorded Decisions

| # | Decision | Rationale |
|---|---|---|
| 1 | Name the tool `melete`; reserve `aoede` for repertoire | Muse lineage under the Mnemosys (memory) organization; Melete means *practice*. Gives the org a naming scheme with room to grow. |
| 2 | Four families, deeply parameterized, not one or twenty | Proves the parameter-to-notation pipeline end to end while producing genuinely varied daily sheets from the first run. |
| 3 | Rhythm is a modifier, not a family | Orthogonal overload dimensions multiply the variant space instead of adding to it, and keep rhythm logic in one component. |
| 4 | Notation **and** tablature; fretboard modeled in the core | Several families (string crossing, string skipping, boxed versus shifting) are defined by string topology and cannot be expressed as pitch alone. Retrofitting would mean rewriting every generator. |
| 5 | Hand-rolled IR over `abjad` | The fretboard layer is the tool's core value and no library provides it. Total control over positioning; a one-package dependency surface; pure-function testability. |
| 6 | Session log, no progression model | Variety now, progression later. The log is the substrate progression plugs into and is required anyway to save each day's sheets. |
| 7 | Coverage-aware per-axis sampling | Uniform random clumps. Independent per-axis recency weighting is what makes variety feel varied. |
| 8 | Instrument profiles configurable from v1 | MNEMOSYS principle: exercises are abstract, instruments are configurations. Nearly free if designed in, expensive to retrofit. |
| 9 | No key signatures by default | Modal exercises should not imply a tonal center; explicit accidentals force the reader to see each altered tone. |
| 10 | Instructional prose on a cover page, not on the exercises | Text overlaid on engraved notation clutters the page and competes with the notes. |
| 11 | One combined `practice.pdf` per day | The printing unit is the day, not the exercise. |
| 12 | Organization bootstrap folded into this epic | The org is empty; `.github` is a hard prerequisite for the epic model. This is a from-scratch bootstrap, and splitting it would add ceremony without clarity. |
| 13 | Accept the unmaintained `lilypond` redistribution; fork it if needed | It is the only way to install LilyPond into a virtual environment, and it is open source with a thin packaging layer. If it breaks, fork and contribute back upstream rather than route around it. |

## 17. Deferred to v2

The following are explicitly planned but out of scope, and the v1 design leaves
room for each:

- **Practice logging and note-taking.** How a session is recorded after it is
  played: what was practiced, at what tempo, how it went. The author expects
  this to be the v2 brainstorm.
- **Progression and progressive overload.** Tempo and complexity advancing over
  time. Requires the logging layer first; the session log is its substrate.
- **Measurement dimensions.** Mastery estimation, rolling volume, fatigue
  budgeting — the full MNEMOSYS state model.
- **Additional families.** Voice-led arpeggios (A3), grouping-based patterns
  (P3), legato mechanics (T5), and others from the canonical library.
- **`aoede`** — repertoire management, descended from the MNEMOSYS RPM design.

---

**Status:** Design approved 2026-08-09. Filed as epic
[`mnemosys-project/.github#1`](https://github.com/mnemosys-project/.github/issues/1)
on 2026-08-09. The authoritative naming convention referenced in §2 lives at
[`NAMING.md`](../../NAMING.md) in this repository.
