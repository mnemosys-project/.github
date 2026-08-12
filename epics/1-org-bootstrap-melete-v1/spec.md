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
- [10a. Accidental Spelling](#10a-accidental-spelling)
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

### Document formats are standardized as part of this epic

Because this epic establishes the organization's epic structure from nothing, it
also **standardizes the formats of the documents that structure carries** —
`spec.md`, `plan.md`, and `retrospective.md` — rather than leaving each future
epic to invent its own shape.

This document is the first instance of the spec format and therefore doubles as
its reference implementation. The standardized formats land in `.github`
alongside the other org metadata, and conformance is the documentation-review
bookend's concern.

The value is compounding rather than immediate: a reader who has followed one
epic's spec → plan → retrospective should be able to follow every subsequent
one without relearning the layout, and an agent picking up an epic should be
able to locate a section by name rather than by search.

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

Explicitly not inherited: the database, the API, and the container and cloud
infrastructure. Those are refused, and the refusal is the reason this project
exists. The fatigue and mastery state model is also absent from v1, but that is
an ordering rather than a refusal — §17 defers it.

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
- Rendering to a single combined PDF per day — LilyPond in v1, and the renderer
  is being replaced (§4, *The renderer boundary*)
- A cover page summarizing the day's session
- A machine-readable session log recording every parameter of every pick

### Non-goals

These are excluded deliberately, not overlooked. They are excluded for two
different reasons, and the difference is stated rather than left to the reader,
because a refusal and an ordering are not the same commitment.

#### Permanently out of scope

- **Database, server, API, web UI.** These are what the predecessor project
  became before it was set aside (§1). Melete's entire state is a directory of
  files on the author's machine, and there is no version of this design in
  which a service appears. Refusing them is the point of the tool.
- **GUI.** The configuration file is the interface.
- **Repertoire management.** That is `aoede` (§2) — a separate tool under the
  same organization, not a feature this one grows.

#### Outside v1, deliberately not ruled out

Each of these has a home in §17, which says what would have to exist first.

- **Progression, overload advancement, mastery estimates, rolling volume,
  fatigue budgeting**
- **Practice logging, note-taking, outcome recording**
- **Audio, MIDI, playback, recording a performance and scoring it**

The last is the one most easily mistaken for a permanent exclusion, so the
reasoning is stated here. The domain is retention, and the hard problem is
decay rather than acquisition. Decay is entirely self-reported: the author
writes down that a scale was played at 140, nothing verifies it, and "played at
140" says nothing about how well. The long-term direction is for the exercises
to be played, recorded, and scored objectively — how accurately was that scale
played at that tempo, and is that accuracy decaying. That is the retention
thesis made measurable rather than asserted.

Nothing in v1 implements any of it, and nothing here is a commitment to a
schedule. What it does rule out is designing as though the capability can never
exist, since it is the one thing that would make the tool's central claim
testable.

### Audience

The primary user is the author, who is a bass **student**. His instructor is a
separate person whose review is a feedback gate once a working version exists.
Design decisions favor the author's needs (he reads tablature, not standard
notation) while keeping both renderings available, because instruction materials
frequently require both.

## 4. Architecture

### Pipeline

```text
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

```text
src/melete/
  instrument.py    InstrumentProfile and fretboard queries
  score.py         The IR: Note, Tuplet, Voice, Score. Pure data.
  theory.py        Pitch, interval, scale, and chord math (12-TET integers)
  vocabulary.py    Canonical parameter identifiers and their display names
  families/
    __init__.py    Family registry
    chromatic.py
    scales.py
    arpeggios.py
    intervals.py
  rhythm.py        Cross-cutting modifier: Score -> Score
  selection.py     Coverage-aware sampling; ExerciseSpec and WeightInputs
  lilypond/
    emit.py        Score -> LilyPond source text
    render.py      Adapter over the LilyPond binary
  session.py       Writes and reads sessions/YYYY-MM-DD/
  config.py        Loads and validates config.toml
  cli.py           Argument parsing; wires the pipeline
```

### Where `ExerciseSpec` lives

`ExerciseSpec` — the family name plus its concrete parameters — and
`WeightInputs` — the per-axis recency distances that produced a draw — both live
in `selection.py`, upstream of the families.

They are deliberately **not** part of the Score IR. A family's contract stays
`params -> Score` (§7): it receives a plain parameter dictionary and knows
nothing about the selector that chose it, exactly as it knows nothing about the
emitter downstream. The caller unwraps an `ExerciseSpec` and dispatches. Putting
these types in `score.py` would blur the one boundary §6 exists to keep sharp.

`WeightInputs` is the structure §9 records into `session.json` for replay.

### The two load-bearing boundaries

**`score.py` is the seam.** Families produce a `Score`; the emitter consumes
one. Neither imports the other. A family never learns that LilyPond exists; the
emitter never learns what a Dorian mode is. Every family is therefore a pure
function testable without rendering anything.

**`render.py` is the blast door.** It is the only module aware that a LilyPond
binary exists. If the LilyPond distribution changes, exactly one file changes.
That claim is narrower than it looks; *The renderer boundary* below says how.

`theory.py` and `instrument.py` are the most reused and the most exhaustively
testable modules. They are built first.

### The renderer boundary

Melete v1 engraves through LilyPond, and **that renderer is being replaced.**
The evaluation that decided it is
[`melete#71`](https://github.com/mnemosys-project/melete/issues/71), which
records what LilyPond cost, where it falls short, and what any successor has to
do. The display targets surveyed as successors are in melete's `docs/reports/`.
A migration epic follows this one.

Nothing in this specification is withdrawn on that account. What the change
requires is that a reader can tell **at a glance which parts of the design
outlive the renderer and which do not** — because that distinction is the main
technical asset this epic hands to the migration, and until this section existed
it was implicit in the module layout rather than stated anywhere.

#### What survives the renderer change

| Design element | Specified in |
|---|---|
| 12-TET pitch, interval, scale and chord math | §4 (`theory.py`) |
| The accidental spelling model — `Key`, the three tiers, the implied-parent table, `SpelledPitch` | §10a |
| The instrument profile and the fretboard model | §5 |
| The Score IR, the written-duration contract, the one-level-nesting rule | §6 |
| All four exercise families and their parameter axes | §7 |
| Exercise length, `positional`, the per-family tempo ranges | §7 |
| The rhythm modifier and its four axes | §8 |
| Coverage-aware selection, the weighting expression, the validity gate, determinism | §9 |
| The configuration file and every key in it | §10 |
| The session log, its history, and exact replay | §9, §12 |
| The command-line interface | §11 |
| Error handling and the vocabulary registry | §13 |
| The naming convention and the epic document formats | [`NAMING.md`](../../NAMING.md), [`docs/epic-document-formats.md`](../../docs/epic-document-formats.md) |

That is nearly the whole design. It survives because `SpelledPitch` is
notation-neutral by construction (§10a, *Where spelling lives*) and the IR
carries no LilyPond at all, so the spelling model — the part that took the most
design effort — lands on any renderer that accepts a letter and an alteration
rather than an integer.

#### What is LilyPond-specific

**None of this is deleted or deprecated.** It is the record of what was learned
about generating engraved output from this IR, and it is the input the migration
starts from.

| Element | Recorded in |
|---|---|
| `lilypond/emit.py` — the only module that knows LilyPond syntax | §4, §14 |
| `lilypond/render.py` — the only module that knows a binary exists | §4, §13 |
| The golden `.ly` files, and golden-file testing as the verification strategy | §14 |
| The `\tabFullNotation` branch for `staves = "tab"` | §10, *Notation conventions* |
| Clef selection, and the written-pitch convention | decision #39 |
| `\key <tonic> <mode>` and LilyPond's mode keywords | §10a, tier 1 |
| `\bar "\|."`, `\accidentalStyle forget`, `\tuplet n/m` | §6, §7, §10 |
| The markup `bookpart` carrying the cover page | §12 |
| LilyPond as a binary prerequisite, and everything that followed from it | §15 |
| The seven constructs written blind and accepted on first contact | `melete#71` |

#### The blast door is narrower than its name

The claim above — that a change of LilyPond *distribution* touches one file —
held, and was paid out once: decision #23 replaced the Python package with a
system binary and cost a single line of `pyproject.toml` (§15, *Containment
held*).

It does **not** hold for a change of *renderer*. `render.py` isolates the
binary; `emit.py` isolates the syntax, and `emit.py` is by far the larger of the
two. A reader who takes "blast door" to mean a one-file swap will underestimate
the migration substantially. The measured figure is in `melete#71`: roughly 210
statements across the two modules, plus every golden file, out of a codebase in
which every other module is renderer-agnostic.

The boundary still did its job. Confining the damage to two modules is what
makes replacing the renderer a bounded project rather than a rewrite — decision #5
paying out a second time, in a form it was not chosen for.

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
    position_span: int = 4    # how much neck one hand covers (§7, §10)
```

Built-in profiles:

| Profile  | Strings | Tuning      | Frets | Notes         |
|----------|---------|-------------|-------|---------------|
| `bass4`  | 4       | E A D G     | 20    | standard      |
| `bass5`  | 5       | B E A D G   | 24    | standard      |
| `bass6`  | 6       | B E A D G C | 24    | **default**   |

Users may define explicit tunings and fret counts in configuration.

`fret_count` is **not cosmetic and must never be inferred**. §9 rejects and
resamples any specification requiring a fret range the profile cannot supply, so
the fret count determines which specifications are valid, which determines the
candidate pool, which determines what the selector draws. Two installations that
disagreed on a profile's fret count would produce different sheets from the same
seed, silently defeating the reproducibility guarantee in §9. The values above
are therefore part of the profile definition, not a rendering default.

A generated exercise specification is validated against the active profile. A
specification requiring string indices or a fret range the profile cannot supply
is invalid and is resampled — for example, a six-string-spanning skip pattern on
`bass4`.

`position_span` is a declared number for the same reason `fret_count` is, plus
one of its own. How many frets fall under one hand follows from fret spacing,
which is a fact about the instrument; and a bound the families themselves must
respect has nowhere else to live, since a family is a pure `params -> Score`
function handed nothing but its parameters and the profile. It carries a default
rather than being required, because most instruments agree about it. §7 states
what it means and §10 states how to override it.

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

Five decisions embedded here:

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
bar-splitting logic. The rule is **enforced at runtime, not merely stated**:
`score.py` rejects a `Tuplet` element that is not a `Note`, and a voice element
that is not a `Note` or a `Tuplet`, with a `TypeError` naming the offending
index (decision #26).

**`Note.duration` is always the *written* value; `Tuplet.ratio` supplies the
scaling.** A triplet of eighths is three notes of duration `1/8` inside a
`Tuplet` with ratio `(3, 2)`. Sounding time is derived, never stored:

```text
sounding(note) = note.duration * ratio[1] / ratio[0]     # inside a Tuplet
sounding(note) = note.duration                           # otherwise
```

This is the contract at the `score.py` seam, and it must be explicit because
`rhythm.py` and `lilypond/emit.py` sit on opposite sides of it and would
otherwise be written against different assumptions — with every tuplet rendering
at the wrong note value as the result.

Written durations are the correct choice rather than an arbitrary one. Sounding
durations inside a tuplet are not representable as noteheads at all: a triplet
eighth is `1/12`, and there is no twelfth note. Storing sounding time would
force `lilypond/emit.py` to recover a writable value by inverting the ratio,
pushing arithmetic and a new failure mode into the one module deliberately kept
free of both. Anything needing real time — measure math, tempo estimates, the
length gate in §9 — applies the formula above through a single shared helper.

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
| `scale_type`    | 7 major modes, 7 melodic minor modes, 7 harmonic minor modes, major and minor pentatonic, blues, whole-tone, two diminished (27) |
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
| `quality`       | maj, min, dim, aug, maj7, min7, dom7, m7b5, dim7, min_maj7, maj6, min6 |
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
| `root`         | 12 pitch classes                             |
| `scale_type`   | as `scales`; **read only when `context` is diatonic** (§10) |
| `string_skip`  | 0 (adjacent), 1, 2                           |
| `string_set`   | subsets                                      |
| `direction`    | up, down, up-down                            |
| `pattern`      | ascending pairs, descending pairs, alternating |

`root` and `scale_type` are what "diatonic within root + scale" refers to; they
are axes of this family like any other and its `[pool.intervals]` section
declares them. `scale_type` is the one axis in the system whose declaration is
conditional — see §10.

### Exercise length

One exercise is **one complete cycle of its pattern** — a full two-octave scale
traversal, a full permutation cycle, whatever the pattern's natural unit is. An
exercise is never truncated mid-pattern.

### Bounding the length

A cycle is not a fixed quantity. `range_octaves`, `pattern`, and `direction`
multiply, and §9 samples them independently: a one-octave pentatonic ascending
straight is six notes, while a three-octave scale in thirds, up-down, is close to
ninety. That is roughly a twentyfold spread across draws that are all legal under
the §10 example pool, and it makes the length of the printed sheet an
uncontrolled output when §1's success criterion is a *printable* morning sheet.

`max_notes` (§10, `[session]`) bounds a single exercise. A sampled specification
whose realized cycle exceeds it is **resampled through the same validity
machinery §9 already applies to instrument-profile violations**, and exhausting
the retry budget is the same loud error naming the over-constrained axis. This
adds a bound, not a model — rolling volume and fatigue budgeting remain deferred
to v2 per §17.

### `positional` means a position

`positional` is a traversal that makes a physical claim: every note of the
exercise falls under one hand, with no shift. A specification whose content
cannot fit within one position is therefore **invalid rather than
approximated**. The family raises, naming the axes that could not all be
satisfied, and §9's gate resamples the draw.

The positional layout is chosen by minimizing total fret travel, and an argmin
has no floor of its own. Two octaves of a pentatonic across three strings cannot
fit under a hand on any tuning, so the minimization returns the least bad answer
— and reporting that as success printed a cover page reading "G♭ major
pentatonic, **positional**, ascending groups of 4" above notation demanding a
fourteen-fret reach. That is worse than an unplayable exercise. An unplayable
exercise announces itself at the first attempt; a mislabelled one does not,
because the label is the part a student trusts, and it teaches the reader that
this is what a position is.

Refusing converts a silent mislabelling into an over-constrained specification,
which §9 already knows how to resample. `range_octaves = 2` over a three-string
set simply stops drawing as positional, which is correct: it is not a positional
exercise.

How much neck one hand covers is `position_span` on the instrument profile
(§10). It lives there because fret spacing is what decides the reach, and
because a bound a family must respect has nowhere else to live — a family is a
pure `params -> Score` function and the profile is the only thing it is handed
besides its parameters.

### The terminal measure

Because §6 does not model measures, a cycle's sounding duration is generically
not a whole number of bars: twenty-four notes at sixteenths in 7/8 is one bar
plus ten sixteenths. **The short final measure is accepted and closed with
`\bar "|."`.**

This is consistent with §6's deliberate stance on grouping against the meter, and
it is preferable to padding with rests. Rests are notation; a player reading a
practice sheet would reasonably read them as musical content rather than as
filler.

### Tempo

`Score.tempo_range` (§6) is supplied by the **family**, not sampled as an axis.
Each family declares a default range, overridable per family in configuration:

| Family | Default | Reasoning |
|---|---|---|
| `chromatic` | 60–120 | Finger-independence work starts slow and deliberate, but the same permutation is played at speed once it is clean. |
| `scales` | 80–140 | Three-notes-per-string patterns are practiced in triplets near the top of this range; the bottom of it is a warm-up. |
| `arpeggios` | 80–140 | Comparable demand to scales. |
| `intervals` | 70–130 | String crossing and skipping cost accuracy at speed, so the range starts lower — but not by as much as first assumed. |

These ranges were revised upward after playing against them. The original set
(60–80, 80–100, 80–100, 70–90) was reasoned from categories rather than from an
instrument — chromatic work is deliberate, scales are faster — and the result was
too slow to be useful. 80–100 for a three-notes-per-string scale is where the
author warms up, not where he practices; those get played in triplets at 120–140.

The revised numbers are better, not authoritative. They are one player's ranges
on one instrument, and they are a **starting point rather than a prescription**.
Every one of them is overridable per family in configuration — `[pool.<family>]
tempo`, shown below — and a reader whose hands disagree with the table should
override it rather than read it as a claim about how fast the exercise ought to
be played.

The durable answer is not a better default. It is the measurement and logging
layer §17 defers to v2: once the tool records what was actually played and at
what tempo, a per-family range is derived from the player's own history instead
of declared in advance, and the number shipped here stops mattering. Until then
the default's job is to be a reasonable place to start and easy to change.

Tempo is deliberately **not** a sampled axis. It is a difficulty parameter, and
letting it vary randomly across sessions would be progressive overload arriving
through the back door — which §17 defers to v2. A static per-family range keeps
v1 free of any progression model while still making §12's cover page, which
prints a tempo per exercise, producible as specified.

Configuration overrides it per family:

```toml
[pool.chromatic]
tempo = [50, 70]
```

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

```text
FLOOR   = 0.05
HORIZON = 14                                    # default; configurable

w(value) = 1.0                                  # never used
w(value) = max(FLOOR, min(1.0, sessions_since / HORIZON))
```

The weight is **one expression with the floor applied last**, deliberately, and
not a ladder of special cases. An earlier formulation carried a separate
"used today or yesterday" rule alongside the ratio, which made the two disagree:
at distance 0 the ratio yields 0.0 while the special case says 0.05, and at
distance 1 the ratio yields ≈0.071, so 0.05 was acting as a ceiling rather than
the floor it was described as.

That ambiguity was not cosmetic. Read with the ratio taking precedence, anything
used today weighs 0.0 — *impossible*, which is the opposite of the intent — and
because the within-session rule below pushes each selection onto the history at
distance 0, a small pool reaches a state where every candidate weighs 0.0. A
weighted draw over an all-zero vector has no defined result. With `shape =
{ scales = 3 }` over `scale_types = ["ionian", "dorian"]`, the third slot hits
exactly that, on a configuration that is otherwise perfectly legal.

The single expression is monotonic, never zero, and keeps the whole weighting
policy in one place — which is what §9's tunability requirement below actually
needs, since swapping linear decay for exponential must remain a local change.

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

**A `[pool.<family>]` section that declares no axis is a family opted out of an
unshaped draw.** With no shape, `family` is itself a weighted axis, and its
candidates are exactly the families whose pool section configures something.
Declaring the section *is* the opt-in; omitting it, or leaving it empty, says
this configuration does not practise that family. If no section declares any
axis and no shape is declared, there is nothing to draw from and the run stops
with an error saying so.

Naming a family in `shape` is the other way in, and it takes the other route.
A family named in the shape whose pool is empty or half-written reaches the
undeclared-axis error in §10 rather than being quietly dropped here — an
explicit request is answered, or refused by name, never silently ignored.

### Validity

Validity is a hard gate, not a weight. Each sampled specification is checked
against the active instrument profile, **against `max_notes`** (§7) and
**against `max_fret_span`** (§10). Invalid specifications are resampled up to a
bounded retry count; exhausting retries is a loud error naming the
over-constrained axis, never a silent fallback.

All three checks run through the same gate. A cycle that is too long, a string
set the profile cannot supply and a reach no hand has are the same kind of
failure — a specification the instrument or the session cannot accommodate — and
none of them is ever quietly adjusted into something renderable. The fretboard
is checked twice over deliberately: the family checks that a note is *on* the
neck, and only the span bound checks that a player can get to it.

**Hand span is a third predicate, not a new concept.** The gate was already the
right shape. Decision #17 added `max_notes` through this same machinery for the
same class of problem — a specification that is legal but not useful — so the
reach bound reuses the resampling, the retry budget and the loud failure rather
than standing up a second mechanism beside them.

Span is measured between the **lowest and highest fretted** notes; open strings
are excluded. That exclusion is a deliberate simplification with a known limit,
recorded as provisional in decision #37 and stated in full under §10.

**A high rejection rate is expected and is not a symptom of anything.** Because
the axes are sampled independently, a draw routinely combines values that
contradict each other, and nothing in the draw coordinates them:
`three_note_per_string` is realizable only when the degree count is exactly
three times the string count, and `traversal` and `string_set` are drawn without
consulting one another. Measured over 20,000 draws per family from a broad pool
on `bass6`, the realizable fraction is 27.8% for `scales`, 34.7% for
`chromatic`, 57.9% for `arpeggios` and 71.3% for `intervals`. A legitimate pool
that pins `three_note_per_string` falls to 11.2%, and a `string_skip` of 2 over
a three-string set is unrealizable outright, at 0%.

None of that is a defect. The gate rejects and redraws, so an unrealizable
combination costs attempts rather than correctness, and no rejected draw ever
reaches the page. It is, however, why the retry bound is 500 rather than a
handful: the bound was sized against this measurement rather than guessed. At
500 attempts a pool half as good as the worst legitimate one measured — one
valid draw in twenty — has roughly a 1-in-10¹¹ chance of failing spuriously,
while a genuinely over-constrained pool is reported in milliseconds instead of
being ground against.

Those fractions were measured before the span bound existed, and a bound that
rejects more draws could have invalidated the sizing. They were therefore
re-measured rather than assumed when it was added, and the retry bound of 500
stands on the lower fractions.

### Determinism

The seed derives from the date plus a hash of the configuration and is written
into `session.json`. Randomness is real but never irreproducible.

#### `[output]` is excluded from the configuration hash

The hash covers `[instrument]`, `[session]` and every `[pool]` section — the
instrument decides which specifications are valid, `[session]` decides how many
and of what, and `[pool]` is the candidate set itself. **`[output]` is
deliberately left out** (decision #42). It selects how a drawn exercise is
engraved, not which exercises are drawn, so folding it into the hash would fold
it into the seed and changing `staves` or `key_signatures` would silently hand
back a different set of exercises.

That exclusion is what makes `--staves` safe to pass on a whim and `--count`
not: one is presentational and the other moves the draw. Candidate lists keep
the order they were written in, because a weighted draw walks them in sequence,
so two pools holding the same values in a different order are two different
configurations and hash differently.

**A seed alone is not sufficient to reproduce a day, and the design accounts for
this.** Selection depends on the session log — the weights above are computed
from how many sessions have passed since each value was last used — and that
history is not a function of the seed. Replaying a seed against today's log
computes different weights than the original run did, because the log has grown
since. The result would be a different sheet, produced silently, with no
indication it had diverged.

`session.json` is therefore **self-sufficient for replay**, and it is so because
it records **every exercise in full** — the family and every drawn parameter.
Alongside them it records the seed, the configuration hash, and the resolved
weight inputs: the `sessions_since` distance per candidate value per axis that
fed that day's draw.

Those two records do different jobs, and conflating them is the error this
section previously made. **The recorded exercises are what `replay` reads.** The
recorded weight inputs are the *account* of why the draw came out as it did —
what makes a past sheet auditable rather than merely regenerable — and they
never re-enter `selection.select`.

| Invocation | Behavior |
|---|---|
| `melete generate` | Normal generation; computes weights from the current log. |
| `melete generate --seed <n>` | Same seed against the **current** history. Not a reproduction, and not described as one. |
| `melete replay <date>` | Re-engraves the exercises recorded for that day. Nothing is drawn: neither the recorded seed nor the recorded weight inputs is fed back into the selector. |

Separating `--seed` from `replay` is what makes the guarantee literally true.
`--seed` fixes the draw against whatever history exists now; `replay` reproduces
a sheet exactly, because the contents of that sheet are on disk rather than
re-derived.

#### Replay is read-back, not re-execution

"Self-sufficient for replay" admits two readings, and the difference matters
enough to state. `melete replay <date>` **re-engraves the exercises recorded in
`session.json`**, running each one back through its family, §8's rhythm
modifier, the emitter and the renderer — `generate`'s pipeline minus the draw.
It does **not** re-run the selector against the recorded seed and distances.

Read-back is the correct reading here. The exercises are recorded in full, so
re-deriving them adds nothing to the reproduction itself, and the injection
point re-execution would need does not exist in `selection.select`.

**What that means replay does not catch, stated plainly:** a change to the
selector that would have drawn differently is invisible to it. The recorded
exercises come back either way, so a replay that succeeds says nothing about
whether today's selector would still produce that day. Re-execution would be the
stronger guarantee — it would turn replay into a regression test on the
weighting function — but that is a different feature wearing the same name, and
building it is a decision for whoever wants that test rather than a gap in this
one.

The ambiguity was found when B14's replay test turned out to pass somewhat
trivially under the read-back reading, which is the kind of thing a test tells
you only if you ask what it would have caught.

Two smaller consequences follow from replay reading the record rather than
re-deriving it. The instrument is checked **by name** and a mismatch is refused,
because engraving one bass's recorded string indices and frets for another
produces convincing tablature for the wrong instrument (§5). The tempo ranges
and the staff mode, being presentational, are read from the configuration as it
is now, and a configuration hash that no longer matches prints a note saying so
rather than refusing.

### Tunability

The weighting function and horizon live in one module behind one entry point.
Replacing linear decay with exponential decay must be a small, local change. The
author intends to tune this empirically through daily use.

## 10. Configuration

A single TOML file. The `[pool.*]` sections are the primary tuning surface.

**The configuration below is complete and working, and is meant to be copied
and run as it stands.** Every axis of every family its `shape` names is
declared, because §9 samples every axis its family reads and an axis the pool
does not declare is an error rather than a default (below). That makes the
example longer than a fragment would be, and the length is the honest picture of
what this tool asks for. An earlier version of this section declared the same
four-family shape over a single `[pool.scales]` section, which is not a shorter
configuration but a broken one. All four families raise on their first draw:
three have no pool section at all, and `scales` has one that never declares
`string_set` or `direction`.

```toml
[instrument]
profile = "bass6"                    # bass4 | bass5 | bass6, or explicit tuning
position_span = 4                    # one hand position, as a span (section 7)

[output]
staves = "both"                      # both | tab | notation
key_signatures = true                # print the key signature (section 10a)

[session]
count = 5
horizon = 14
max_notes = 96                       # per-exercise length bound (section 7)
max_fret_span = 12                   # per-exercise reach bound (section 9)
shape = { chromatic = 1, scales = 2, arpeggios = 1, intervals = 1 }

[pool.chromatic]
permutations = "all"                 # all 24 orderings of the four fingers
start_strings = [0, 1, 2, 3]
start_frets = [1, 3, 5, 7]
directions = "all"
string_traversals = ["adjacent", "skip_1"]
shifts = ["none", "fret_per_cycle"]
spans = [3, 4]
tempo = [60, 84]                     # overrides the family default (section 7)

[pool.scales]
roots = "all"
scale_types = ["ionian", "dorian", "phrygian", "major_pentatonic", "blues"]
traversals = ["positional", "three_note_per_string"]
string_sets = [[0, 1, 2, 3, 4, 5], [0, 1, 2], [1, 2, 3], [2, 3, 4], [3, 4, 5]]
patterns = ["straight", "thirds", "groups_of_3", "groups_of_4"]
octaves = [1, 2]
directions = "all"
tempo = [80, 100]

[pool.arpeggios]
roots = "all"
qualities = ["maj7", "min7", "dom7", "m7b5", "min6"]
inversions = ["root", "first", "second"]
traversals = ["positional", "across_strings"]
string_sets = [[0, 1, 2, 3], [1, 2, 3, 4], [2, 3, 4, 5]]
patterns = ["straight", "broken", "sweep_ordered"]
octaves = [1, 2]
directions = "all"

[pool.intervals]
intervals = [3, 4, 5, 6]             # 2nd through 10th (section 7)
contexts = "all"                     # chromatic | diatonic
roots = "all"
scale_types = ["ionian", "dorian", "aeolian"]
string_skips = [0, 1]
string_sets = [[0, 1, 2, 3], [1, 2, 3, 4], [2, 3, 4, 5], [0, 1, 2, 3, 4, 5]]
directions = ["up", "down"]
patterns = ["ascending_pairs", "descending_pairs", "alternating"]

[pool.rhythm]
subdivisions = ["eighth", "triplet_eighth", "sixteenth"]
time_signatures = ["4_4", "3_4"]
accent_patterns = ["none", "every_3"]
note_value_patterns = ["straight", "long_short"]
```

Three conventions in that file are worth naming.

`"all"` expands an axis to every value it accepts, and is written wherever the
shorthand is honest — `roots`, `permutations`, `directions`, `contexts`.
`string_sets` has no `"all"` and never will: its accepted values are not an
enumeration but a structural rule, since every non-empty subset of six strings
is sixty-three values and an error message listing them is not one anybody could
read. Its candidates are therefore always written out, low string to high.

Where a family realizes only part of an axis's registry, the values are listed
rather than expanded. `traversals` and `patterns` are single axes shared across
families — `across_strings` is an arpeggio traversal and `three_note_per_string`
is a scale one — so `"all"` on either would fill the pool with combinations the
family rejects, spending retries to no purpose.

`tempo` is not a sampled axis (decision #20). It is a per-family default, shown
here overridden for two of the four families and left alone for the other two.
`[pool.rhythm]` is a section of `[pool]` but not a family, so it carries axes
and no tempo.

### The two playability bounds

Two keys bound what an exercise asks of the fretting hand, and both are stated
as a **span**: the distance from the lowest fretted note to the highest.

| Key | Default | What it bounds |
|---|---|---|
| `[instrument] position_span` | 4 | The width of one hand position (§7). |
| `[session] max_fret_span` | 12 | The reach of any one exercise (§9). |

`position_span` is four because four fingers cover four frets, and reaching one
fret beyond them is ordinary technique rather than a stretch a player would
notice. One hand position is therefore five frets, and the span between its
outermost notes is four. It sits on the instrument rather than the session
because fret spacing is what decides the reach: a short-scale instrument puts
more frets under the same hand. It is written beside `profile` rather than
inside an explicit tuning so that a player whose hand disagrees with the default
does not have to write out a whole tuning to say so.

`max_fret_span` is twelve because that is one octave of neck, and it is
deliberately looser than a position: shifting is legitimate practice, not a
defect to be bounded away. `chromatic` with `shift = fret_per_cycle` is
*supposed* to climb — it tops out at nine frets under the example pool above —
and it must keep drawing. What twelve refuses is an exercise that covers more of
the neck than the neck's own repeating unit: every shape recurs an octave
higher, so a span past twelve contains a repetition of itself.

**Open strings are excluded from the span, and that rule is provisional**
(decision #37). Fret 0 sounds while the fretting hand stays where it is, so an
open string neither extends the reach nor pins it to the nut. Without the
exclusion the open-A minor pentatonic box — the open A against frets 3, 5 and 7
— is refused as a seven-fret stretch, when it is one of the most standard shapes
on the instrument.

The rule is blunt, and it is known to be blunt. It cannot distinguish an open
string at the bottom of a low shape, where a three-note-per-string scale
starting on an open string is one position and entirely reasonable, from an open
string interleaved with a hand high on the neck — playable, since the fretting
hand does not move, but a different kind of awkward that the bound is not
measuring. It is accepted as a temporary simplification to get control of the
span problem and produce sheets that read as intuitively correct, on the
explicit understanding that it will be revisited once there is experience of how
the exercises actually play. Tracked as `mnemosys-project/melete#60`.

**Both numbers are a first pass.** A reach bound is the kind of thing that can
only be tuned by generating sheets and looking at them, which is why it is
configuration rather than a constant, and why this section states defaults
rather than settled values.

### An axis the pool does not declare is an error, never a default

An axis a family reads but its `[pool.*]` section does not declare is a **loud
error at the first draw from that pool, naming both the axis and the section**.
Nothing is defaulted, nothing is inferred from the axes that were declared, and
the failure lands before anything is engraved.

This is §13's stance applied to an absent key rather than to a misspelled one,
and the argument is the same one: a candidate pool the configuration did not
write produces a sheet the author did not ask for and cannot account for.

The alternative is superficially attractive, because most axes look like they
have an obvious fallback. `string_sets` is the axis that shows they do not. All
63 non-empty subsets of six strings is nonsense as a practice pool. Restricting
the default to contiguous subsets invents a musical judgment — that string
skipping is exceptional — which the tool has no business making on the author's
behalf. Defaulting to the full string set silently converts every positional
exercise into a different exercise. There is no defensible choice among those,
so the tool declines to make one, and it declines uniformly rather than
defaulting the easy axes and erroring on the hard one.

The error arrives at the **first draw from that pool** rather than at load,
because a pool is only incomplete relative to the family that samples it.

#### The one exception: a conditional axis

`scale_type` on `intervals` is read only when the drawn `context` is
`diatonic` — a chromatic interval sequence never consults it, and two chromatic
specifications differing only in `scale_type` engrave the same sheet. So it is
**not sampled at all** in that branch: it does not enter `params`, it is not
pushed onto §9's history, and the pool is not asked for candidates it will not
use. A pool whose `contexts` is `["chromatic"]` alone may therefore legally omit
`scale_types`. Any pool that can draw `diatonic` must still declare it, and the
omission is caught the first time a diatonic context comes up.

This is the same argument as the rule it excepts, not a softening of it.
Requiring `scale_types` for a chromatic-only pool would demand configuration
that changes nothing on the page, and sampling the axis anyway would credit §9's
coverage accounting with variety that does not exist — pushing a scale type down
the pool for the next slot without a single note of it being played. The rule
and the exception both say the pool describes what will actually be drawn from.

`context` is drawn before `scale_type`, which is what makes the condition
decidable at the moment it is needed. The recorded weight inputs still carry the
axis, because they record what fed the *draw* including the attempts that were
rejected.

### Explicit tunings

`profile` accepts a built-in name or an explicit definition, for a drop tuning
or a non-standard instrument:

```toml
[instrument]
profile = { name = "drop_d", tuning = [26, 33, 38, 43], fret_count = 20 }
```

`tuning` is absolute pitches, **low to high**; index 0 is the lowest string and
every family depends on that ordering (§5). A tuning that is not strictly
ascending is a configuration error, not a re-sortable input — silently sorting it
would move every string index and produce correct-looking tablature for the
wrong instrument. `fret_count` is required, for the reasons in §5.

### Notation conventions

**Key signatures on by default**, and notes spelled correctly for the key. See
§10a for the spelling model.

> **Amended 2026-08-10, superseding decision #9.** This section previously read
> "no key signatures by default; modal exercises are notated in C with explicit
> accidentals throughout." That was not a notation choice — it was a missing
> layer wearing one. See §10a and decision #27.

`key_signatures` remains configurable, and now selects between two settings that
are both correct:

| Setting | Behavior |
|---|---|
| `true` *(default)* | Print the key signature — `\key fis \dorian` — and spell diatonically. Fewest accidentals; matches published practice material. |
| `false` | Print **no** signature, but still spell correctly: F♯ G♯ A B C♯ D♯ E, with an explicit accidental on every altered tone. |

The `false` setting is decision #9's original intent, finally implemented
properly: no asserted tonal center, every altered tone visible to the reader,
and F♯ spelled F♯. What it is *not* is the old behavior, which asserted no tonal
center by spelling the notes wrongly.

The default is `true` because that is what published practice material looks
like, and because the primary reader of the notation staff is an instructor.

**Staff mode is a switch.** `both` (default), `tab`, or `notation`.

The switch itself is renderer-agnostic. Its **consequence is not**, and the
consequence is the part a successor renderer has to re-derive: LilyPond's
`TabStaff` suppresses stems and beams by default, assuming a notation staff
above supplies the rhythm. In `tab` mode the emitter must therefore explicitly
enable rhythm display with `\tabFullNotation` — Guitar Pro-style tablature with
stems — or the exercise is unreadable. In `both` mode plain tablature is
correct. This is a branch in the emitter, not a flag.

What generalizes past LilyPond is the requirement, not the mechanism: **tab-only
output must carry rhythm**, however the renderer expresses that. What does not
generalize is the assumption that it is off by default. See §4, *The renderer
boundary*.

## 10a. Accidental Spelling

### The defect this section exists to fix

`Note.pitch` is a 12-TET integer. In twelve-tone equal temperament F♯ and G♭ are
the same number, so an integer **cannot** carry a spelling. `theory.PITCH_CLASSES`
is an all-flats table, and the emitter derived every note name from it.

F♯ Dorian therefore engraved as **G♭ A♭ B𝄫 C♭ D♭ E𝄫 F♭**. That is not an awkward
rendering of F♯ Dorian; it is a different key, and an absurd one. Tablature was
unaffected — the fret numbers were right — so the two staves disagreed silently,
which is the worst form the failure could take.

The fix is not a better table. It is the layer that was missing: **spelling is a
function of the key, and the key was never represented.**

### Key

```python
@dataclass(frozen=True)
class Key:
    tonic: int        # pitch class, 0-11
    scale_type: str   # identifier from theory.SCALES
```

`Score` gains `key: Key | None`. Families set it — they already hold `root` and
`scale_type`. `None` is a real value, not an omission: it means the exercise has
no key, which is true of everything the `chromatic` family produces.

**The tonic's letter is derived, never stored.** Pitch class 6 is F♯ or G♭
depending on the key: F♯ Dorian is the notes of E major, four sharps; G♭ Dorian
is the notes of F♭ major, eight flats. The rule is to spell the tonic whichever
way yields the signature with **fewer accidentals**, rejecting any spelling that
requires a double accidental in the signature. F♯ Dorian beats G♭ Dorian 4 to 8;
D♭ major beats C♯ major 5 to 7.

Keeping the tonic an integer means `root` stays an integer in the families, the
configuration and the selector. Only one derivation ever asks about letters.

### Three tiers

**Tier 1 — the seven diatonic modes.** Seven degrees, seven letters, each used
exactly once. Fully determined by the tonic letter and the interval pattern; no
judgment involved. Emits `\key <tonic> <mode>`.

**Tier 2 — scales with a parent.** Melodic and harmonic minor and their modes
still have seven degrees, so the letter rule still holds. The pentatonics and
blues are subsets of a seven-note parent and are spelled as that parent spells
them — so the blue note is a ♭5.

The signature is the **parent's**: major for major pentatonic; natural minor for
minor pentatonic, blues, and the melodic and harmonic minor families. Everything
outside the signature prints an accidental, which is exactly how melodic and
harmonic minor are conventionally written — the raised sixth and seventh appear
as accidentals against the natural-minor signature.

**Tier 3 — symmetric and keyless.** Whole-tone, both diminished scales, and
anything from the `chromatic` family. No signature. Spelled **by direction**:
ascending intervals take sharps, descending take flats.

Letters necessarily skip or repeat here, and that is accepted rather than worked
around: six notes cannot occupy seven letters, and eight cannot avoid repeating
one. A symmetric scale has no parent to inherit from, so direction is the only
signal available.

### Arpeggios

Chords are spelled by **function** — root, third, fifth and seventh take the
letters of degrees 1, 3, 5 and 7 — which is its own rule, not the scale rule.

Rather than give `Key` a second form, each chord quality maps to an **implied
parent scale**, and the tiers above do the rest:

| Quality | Implied parent | Tier |
|---|---|---|
| `maj`, `maj6`, `maj7` | ionian | 1 |
| `min`, `min7` | aeolian | 1 |
| `min6` | dorian | 1 |
| `min_maj7` | melodic_minor | 2 |
| `dom7` | mixolydian | 1 |
| `m7b5` | locrian | 1 |
| `dim`, `dim7` | diminished_whole_half | 3 |
| `aug` | whole_tone | 3 |

The parent column names scale identifiers, not prose: these are the keys of the
scale registry, and the table is read as written. "Diminished" on its own would
not be — there are two diminished scales, and tier 3 above refers to both — so
`dim` and `dim7` name `diminished_whole_half` exactly.

Chord tones then fall out as a subset of the parent's spelling, one mechanism
serves both scales and chords, and the arpeggios family sets `Key` exactly like
the others.

**Every quality must map to a parent that contains all of its chord tones.**
That is the property the table has to satisfy, and it is not a nicety of the
mapping but the condition under which the mapping means anything. Subset
spelling names a chord tone by the degree it matches; a tone the parent does
not contain has no degree to take its letter and falls to the out-of-scale
path instead. §14 asserts containment for every quality across all 12 roots.

An earlier version of this table mapped both `min6` and `min_maj7` to aeolian,
and aeolian contains neither chord's characteristic tone: the added sixth is 9
semitones and the major seventh is 11, while aeolian has 8 and 10. Through
aeolian, F♯ min6 spelled `F# A C# Eb` and F♯ min_maj7 spelled `F# A C# F` —
wrong letters in both cases. Dorian contains the natural sixth and melodic
minor contains the major seventh, so each chord now has a parent that names all
four of its tones. This was caught during implementation by checking whether
the model actually held rather than by transcribing the table, which is why the
containment assertion in §14 exists: the property was always the requirement,
and nothing had been asked to enforce it.

### The fully diminished seventh is a known limit

`dim` and `dim7` stay in tier 3 and are therefore spelled by direction rather
than by a parent. C dim7 spells `C D# F# A`, not the functional `C Eb Gb Bbb`.

This cannot be fixed by remapping the parent. A fully diminished seventh needs
a **doubly diminished seventh** above the root, and no seven-note scale
supplies one — there is no parent to point at, so the containment property
above cannot be satisfied for this quality at all. Spelling it functionally
would require degree-aware chord spelling: the speller would have to know that
a given pitch is *the seventh of this chord* and name it accordingly.
`spell(key, pitches)` deliberately does not carry that. It receives bare
pitches and infers function from pitch-class membership in the parent, and that
is exactly what lets one mechanism serve both scales and chords.

So this is a structural limit of the model as designed, not a defect scheduled
for repair. The design chose the narrower contract and the single spelling
path, and this is what that choice costs. It is recorded here, next to the
tiers, so a reader meets it as a stated boundary rather than discovering it in
an engraved sheet.

### The plain diminished triad follows it, and the alternative was declined

`dim` maps to the same parent as `dim7`, so the triad is spelled by direction
too: `A dim` spells `A C D#` rather than the functional `A C Eb`.

Unlike the seventh chord, this one is fixable. Locrian contains both the ♭3 and
the ♭5, so mapping `IMPLIED_PARENT["dim"]` to locrian would spell `A C Eb` and
satisfy the containment property while doing it. The triad is not the seventh:
a parent exists that names all three of its tones.

It was left in tier 3 deliberately. Remapping it moves the quality between
tiers, which splits `dim` from `dim7` — the seventh cannot follow it, for the
reason above — and the judgment was to see how the raised fourth reads on an
engraved sheet before changing the model to avoid it.

So this is an accepted outcome with a named alternative, not a case that was
missed. A reader who meets `A C D#` in a generated sheet is looking at the tier
the quality was left in, and the change, should the review ask for it, is one
entry in `IMPLIED_PARENT`.

### Where spelling lives

`theory` owns it. That module already owns modes, intervals and chord content,
and spelling is the same kind of knowledge.

`theory.spell()` returns a **notation-neutral** `SpelledPitch` of letter,
alteration and octave. `lilypond/emit.py` asks for that and only knows how to
write it down; LilyPond's mode keywords stay in the LilyPond package. §4's
boundary therefore holds — the emitter still never learns what a Dorian mode is,
and it stops carrying a hardcoded pitch-name table, which is an improvement on
what it had.

Tablature never consults any of this. A spelling change must leave the tab staff
byte-identical, and §14 asserts it.

### The tablature staff spells keylessly

The emitter passes `key=None` for the tab staff, in `both` mode and in `tab`
mode alike, so the tablature is spelled by direction while the notation staff
is spelled for the key. In a flat key the two staves therefore name the same
pitch differently in the source: `ges` on the notation staff, `fis` in the tab.

That is the boundary rather than a shortcut. A fret number is a function of
pitch, and F♯ and G♭ are the same pitch, so a change of key cannot move a fret.
Feeding the tab staff the key's spelling would make its source vary with a
change that cannot affect its content — which is precisely what the
byte-identical tablature assertion in §14 forbids. The keyless tab is what
makes that assertion literally true rather than approximately so.

The cost is cosmetic and was accepted. LilyPond derives identical fret numbers
from both spellings, so nothing on the engraved page differs; only a reader of
the raw `.ly` sees the disagreement. To that reader it can look like exactly
the notation-versus-tablature divergence this section exists to fix, and it is
not: the original defect was two staves carrying different music, and this is
two staves carrying the same music under two names.

### The instructor-review list

Tier 1 is fully determined. **Tier 2 is conventional practice and tier 3 is a
defensible convention rather than a rule**, and both will be reviewed by a
reader with formal training once real sheets exist. This list is the agenda for
that conversation: every place the model produces something a trained reader is
likely to call wrong, with the answer the design already has.

- **Blue-note spelling.** The blues scale is spelled as its natural-minor
  parent spells it, so the blue note is a ♭5 and not a ♯4. Tier 2 convention.
- **The fully diminished seventh.** `C dim7` spells `C D# F# A`, not
  `C Eb Gb Bbb`. Unfixable inside the model: no seven-note parent contains a
  doubly diminished seventh.
- **The plain diminished triad.** `A dim` spells `A C D#`, not `A C Eb`.
  Fixable — locrian would spell it functionally — and deliberately not fixed.
- **The symmetric scales.** Whole-tone and both diminished scales are spelled
  by direction, so letters skip or repeat. They have no parent to inherit from.
- **Signatures on modal material.** Whether `\key fis \dorian` is the right
  default at all, or whether modal exercises should print no signature and
  spell every altered tone explicitly.
- **The keyless tablature staff.** `ges` on the notation staff and `fis` in the
  tab for the same pitch. Visible in the `.ly` source only.

`C D# F# A` will read as wrong to an instructor, and the honest answer in every
case above is that it is the cost of a deliberate contract rather than an
oversight — recorded here so the review starts from that answer instead of
rediscovering each item as a bug.

### Why the policy sits behind one entry point

The list above is a list of expected revisions, which is what shapes the code.
So the three tiers live in one module behind one entry point, for the same
reason §9 requires it of the selection weighting: the revision we are expecting
should be a small local change, not a refactor. A policy scattered across four
family modules would not survive its first review.

## 11. Command-Line Interface

```text
melete generate                  # today's session
melete generate --date 2026-08-10
melete generate --seed 12345     # fixed seed against current history
melete generate --dry-run        # print selections, render nothing
melete generate --staves tab     # override staff mode for one run
melete generate --count 6        # only when [session] shape is unset
melete generate --force          # overwrite an existing session directory
melete generate --split          # also emit one PDF per exercise
melete replay 2026-08-09         # re-engrave a past session from its record
melete show 2026-08-09           # summarize a past session
melete families                  # list families and their parameter axes
melete vocabulary                # list every axis and its accepted values
```

`replay` and `--seed` are deliberately distinct; see §9 *Determinism*. `replay`
re-engraves the exercises recorded in that day's `session.json` and is the only
operation that reproduces a sheet exactly. Nothing is re-drawn, and the recorded
seed and weight inputs are the account of the original draw rather than inputs
to this one. `--seed` fixes the draw against whatever history exists now.

Three interactions between the flags are not visible in the list above and
surprise people, so they are stated here.

**`--count` is refused when `[session] shape` is declared.** A shape names one
family per exercise slot and therefore already fixes the count, so the two
contradict each other and guessing which the user meant would silently generate
the wrong session. The error names the count the shape declares. **This includes
§10's worked example**, which declares a shape of five: `--count 6` against that
configuration is a refusal, not an override. Edit the shape, or omit it to
weight the families instead. Unlike `--staves`, `--count` changes the draw,
because `[session]` is inside the configuration hash the seed derives from (§9).

**`--dry-run` is refused for a date that already has a session directory.** The
existence check §13 requires runs *before* the draw, so a dry run against an
already-generated day refuses rather than previewing — even though it would have
written nothing. `--force` previews it. The ordering is deliberate: the check
belongs where it can be made before any work happens, and a second copy of it
after the draw would be a second place to keep the rule.

**The two integer flags are bounded at the flag.** `--seed` must be 0 or
greater and `--count` 1 or greater, both rejected by the parser naming the flag.
A negative seed would otherwise reach the log, and a count of zero would ask the
emitter for a book with no exercises in it.

`vocabulary` prints the canonical registry described in §13 — the same source
that configuration validation and the cover-page renderer read.

## 12. Output and Session Log

```text
sessions/2026-08-09/
  practice.pdf        cover page + exercises, one printable document
  session.json        every parameter of every selection, the seed, and the
                      weight inputs that fed the draw
  src/                generated LilyPond source
    book.ly           the combined document
    exercise-01.ly    one file per exercise, numbered from 01
    exercise-02.ly
    ...
```

`--split` adds one PDF per exercise at the top level of the directory, named for
the same stems as the sources: `exercise-01.pdf`, `exercise-02.pdf`, and so on
beside `practice.pdf`.

The names are part of the contract, not an implementation detail: a reader
re-running the engraver by hand after a failed render (§13) needs to know which
file to run. Sources are written before anything is rendered, and they are
rendered *in* `src/` with the finished PDFs moved up — the combined document is
generated under the stem `book` and its PDF lands at the top as `practice.pdf` —
because a render writes its `.pdf` beside its `.ly` and the printable documents
belong at the top of the session directory. The renderer may leave its own
by-products in `src/` as well.

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

> 1. D Dorian, three-notes-per-string, ascending thirds, strings 2-5, triplet
>    eighths, 80-100 bpm
>    *Keep the plucking hand even through the string crossings.*

Exercise pages themselves carry only a title and minimal annotation. The
`instruction` field is never rendered onto an exercise page.

### The session log

`session.json` records the seed, the configuration hash, the full parameter
dictionary for every exercise, and the **resolved weight inputs** that produced
the draw (§9). It is the history the selector reads back. It is human-readable,
git-committable, and hand-editable.

The **parameter dictionaries** are what make the file sufficient for exact
replay: `replay` re-engraves those exercises, and nothing about the draw is
recomputed (§9). The **weight inputs** are what make the sheet auditable — the
question "why did it pick D Dorian three days running?" is answerable from the
file itself, without re-deriving anything. Recording both means a past day can
be reproduced *and* explained, which are two different questions with two
different answers on disk.

## 13. Error Handling

No swallowed exceptions and no fallbacks that hide errors. A failure in the
generator becomes a wrong exercise on the page, which is worse than no exercise.

| Failure | Behavior |
|---|---|
| Malformed or invalid configuration | Fail at load, naming the exact key and its accepted values. Never fall back to a default for a misspelled key. |
| Axis a family reads that its `[pool.*]` section does not declare | Hard error at the first draw from that pool, naming both the axis and the section to declare it under. Never defaulted — an absent key gets the same treatment as a misspelled one, for the reasons in §10. A conditional axis is exempt while its condition does not hold: see §10, *The one exception*. |
| Explicit tuning not strictly ascending | Fail at load, naming the offending index. Never re-sort — sorting would shift every string index and engrave the wrong instrument convincingly. |
| Pool over-constrained | Hard error naming the axis that could not be satisfied — for example, "no valid `string_set` for `bass4` with `octaves = 3`". |
| Family emits a note outside the fretboard | A bug, not user error. Raise. |
| LilyPond render fails | Surface LilyPond's stderr verbatim and **keep the generated `.ly` on disk** for inspection and manual re-run. Never clean up on failure. |
| LilyPond binary missing | Explicit error stating the resolution, not a stack trace. |
| Session directory exists | **Refuse.** `--force` overwrites and must be asked for explicitly. The check precedes the draw, so `--dry-run` is refused too (§11). |
| No `[pool.<family>]` section declares any axis, and no shape is declared | Hard error: there is nothing to draw from. §9's unshaped draw weights the families a pool section opts in. |
| Corrupt session-history entry | Hard error naming the file. Silently skipping a bad entry would degrade variety invisibly. |

### The vocabulary registry

"Naming the exact key and its accepted values" requires an enumerated set of
accepted values to name. `vocabulary.py` (§4) is that set: for every axis in §7
and §8, the canonical snake_case identifier and its human-readable display name.

```python
SCALE_TYPE = {
    "ionian":            "Ionian",
    "dorian":            "Dorian",
    "major_pentatonic":  "major pentatonic",
    ...
}
TRAVERSAL = {
    "positional":            "positional",
    "three_note_per_string": "three-notes-per-string",
    ...
}
```

One registry, three consumers, no drift:

| Consumer | Use |
|---|---|
| `config.py` | Validate identifiers; on failure, name the key and list its accepted values verbatim from the registry. |
| `families/`, `rhythm.py` | Dispatch on the canonical identifier. |
| `session.py`, cover page (§12) | Render display names into the plain-language summary. |

The display half is required regardless of the error contract: §12's cover page
emits "D Dorian, three-notes-per-string, ascending thirds," which is precisely a
rendering of these identifiers. Without a shared registry, configuration
parsing, family dispatch, and the cover-page renderer each grow a private
vocabulary and drift apart — and §13's promise to name accepted values silently
becomes a promise the code cannot keep.

## 14. Testing Strategy

The architecture was chosen partly for testability: almost nothing requires a
rendered PDF.

| Component | Approach |
|---|---|
| `theory.py`, `instrument.py` | Exhaustive. All 12 roots against all 27 scale types, verified against known interval content. Every pitch maps to valid positions on every profile. |
| Families | Property-based over a wide parameter sweep. |
| `rhythm.py` | **Sounding** durations (§6) of a voice sum to the pattern's cycle length; every written duration is a representable notehead; tuplet ratios well-formed. |
| `vocabulary.py` | Every identifier used in §7, §8, and the §10 example config resolves; every axis value has a display name. |
| `selection.py` | Statistical, deterministic under a fixed seed. |
| `lilypond/emit.py` | Golden-file tests on the emitted `.ly` **text**. No rendering. |
| `lilypond/render.py` | One integration test invoking LilyPond, asserting a multi-page PDF. |
| `cli.py` | One end-to-end smoke test into a temporary directory. |

### The central invariant

Every family test asserts, for every generated note:

```text
note.pitch == instrument.tuning[note.string] + note.fret
```

This single property catches nearly every fretboard-positioning bug. Alongside
it: all frets within range, note count matching the declared pattern, and one
complete cycle emitted.

### The selection test

Simulate 200 sessions under a fixed seed; assert each axis distributes near
uniformly and that no value recurs within the horizon more often than the
weighting permits. This is the test that proves the clumping problem is solved.

Two adjacent assertions guard the weighting function itself:

- **No weight is ever zero.** Every computed weight is `>= FLOOR`, so the
  total-weight-zero draw described in §9 cannot occur.
- **The pathological pool terminates.** `shape = { scales = 3 }` over a
  two-element `scale_types` pool selects successfully rather than dividing by
  zero — the case the earlier three-rule formulation would have crashed on.

### Tests that need the binary

Only two tests invoke LilyPond: the `lilypond/render.py` integration test and the
`cli.py` end-to-end smoke test. Everything else — including `lilypond/emit.py`,
which is golden-file tests on emitted **text** — is pure.

That ratio is what makes the binary prerequisite tolerable: fifteen of the
seventeen implementation tasks can be built and verified without LilyPond present
at all. The two that cannot are exactly the tasks covering the rendering path,
where a skipped test would be a hole rather than an inconvenience. They must
therefore **fail loudly when the binary is absent rather than skip silently**, in
any environment that claims to be running the full suite.

### Coverage

The validation pipeline enforces **100% test coverage**. This is not a target to
approach but a gate that fails the build, and it applies from the first module
onward. Any deliberate exclusion must be an explicit `# pragma: no cover` with a
reason, not an untested branch left to accumulate.

### Spelling

The assertion that matters most is the direct analogue of the central invariant,
and it is checked for every note of every generated exercise alongside it:

```text
spelled pitch class == note.pitch % 12
```

A spelling that does not sound the note it names is the failure mode, and it is
precisely the bug §10a exists to fix.

Alongside it:

- **The letter rule** — tier 1 and the seven-note tier 2 scales use seven
  distinct letters. Tier 3 is explicitly exempt, and the exemption is asserted
  rather than assumed.
- **Chord-tone containment** — every chord quality's tones are a subset of its
  implied parent's pitch classes, asserted for all 12 roots. This is the
  invariant §10a's table has to satisfy, and it is the test that would have
  caught `min6` and `min_maj7` mapped to a parent that does not contain them.
  `dim` and `dim7` are the stated exception, exempt for the reason §10a gives.
- **Exhaustive sweep** — all 12 tonics against all 27 scale types: spelling
  succeeds, pitch classes round-trip, and no key signature contains a double
  accidental.
- **Tonic choice** — F♯ beats G♭ for Dorian; D♭ beats C♯ for major.
- **Known keys** — F♯ Dorian is `F♯ G♯ A B C♯ D♯ E`; plus C blues, C whole-tone,
  C diminished and D♭ major.
- **A named regression test** for the original defect: F♯ Dorian must never
  produce G♭, B𝄫 or C♭.
- **Tablature is byte-identical** before and after the spelling change. This is
  a cheap proof of a boundary the design claims, and the claim is worth
  proving rather than asserting.

None of this needs LilyPond. Spelling is testable as pure data, which is why
`SpelledPitch` is notation-neutral.

### The replay test

Generate a session; append several later sessions to the log; then `melete
replay` the original date and assert the result is **byte-identical** to the
first run. This proves §9's reproducibility guarantee holds against a log that
has moved on: a re-derivation would not survive it, because the weights are
computed from a history that has grown.

**What this test does not prove is worth stating beside it.** Replay reads the
recorded exercises rather than re-drawing them (§9), so the assertion passes
whatever the selector currently does. It is a test of the engraving pipeline's
determinism and of the record's completeness, not of the weighting function. The
selection test above is what covers the draw.

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

**There are no runtime Python dependencies.** LilyPond is required, but as a
**binary on `PATH`**, not as a package installed into the virtual environment.

> **Amended 2026-08-10.** This section previously specified the PyPI `lilypond`
> redistribution as the sole runtime dependency, and claimed that this let the
> tool run inside the standard `dev-python` container with no change to the base
> image and no host-level installation. That is not achievable on this project's
> hardware. The original text and the reasoning that replaced it are recorded in
> decisions #13 and #23 rather than deleted.

#### Why the redistribution cannot be a dependency

The PyPI `lilypond` package publishes **x86_64 wheels only**, across all sixteen
releases from 2.24.1 to 2.25.12:

```text
lilypond-2.25.12-0-py3-none-macosx_10_15_x86_64.whl
lilypond-2.25.12-0-py3-none-manylinux2014_x86_64.whl
lilypond-2.25.12-0-py3-none-win_amd64.whl
```

There has never been an aarch64 build. The dev container is arm64 Debian 13 and
the development host is Apple Silicon, so `uv sync` fails on both — not as a
degraded experience, but outright.

Forking does not rescue this. Upstream publishes `darwin-arm64` from 2.27.0
onward but **no `linux-arm64` binary at any version**, so a fork could repackage
for macOS while the container would still need a source build.

#### Where the binary comes from

| Platform | Source |
|---|---|
| Debian / Ubuntu | `apt-get install lilypond` — trixie ships **2.24.4** for arm64 in `main` |
| macOS | Homebrew, or the upstream `darwin-arm64` tarball (2.27.0+) |
| x86_64, any | the PyPI redistribution, via the opt-in `bundled-lilypond` extra |

A note on versions: LilyPond follows the GNU even/odd convention, so **2.24.4 is
a stable release and 2.25.12 is a development snapshot**. The original pin was
therefore to a development build of an unmaintained repackage — a worse position
than this section previously described, independent of architecture.

#### What this costs, stated plainly

The withdrawn claim was a real benefit, not decoration. Obtaining LilyPond is now
an **environment prerequisite** that a user must satisfy before melete works, and
the dev container needs a system package that the shared `dev-python` image does
not and should not carry.

> **Updated 2026-08-11.** Both follow-ons this paragraph opened have since
> closed, in opposite ways, and the outcome is worth recording because the
> cheaper remedy was the one that landed.
>
> - **The container gap was closed at the toolchain level.**
>   [`vergil-tooling#2718`](https://github.com/vergil-project/vergil-tooling/issues/2718)
>   produced a declarative `[container].system-packages` facility, which melete
>   adopted in `melete#51`. The dev and CI containers now carry Debian's
>   LilyPond 2.24.4 without a bespoke image, so the "Vergil has no mechanism for
>   this" statement above is **no longer true** — it describes the position at
>   the time of the amendment, and is left as written because decision #24 was
>   taken from it.
> - **Publishing our own aarch64 wheels was declined.**
>   [`melete#21`](https://github.com/mnemosys-project/melete/issues/21) is closed
>   won't-do: system-packages removed the need, and the stock Debian binary is a
>   stable release where a self-maintained fork of the PyPI redistribution would
>   have pinned us to a development snapshot we also had to maintain.
>
> Neither changes the user-facing contract for daily use, which is still that
> the binary is a host prerequisite outside the container.

#### Containment held

This amendment changed one line of `pyproject.toml` and no application code,
because `lilypond/render.py` is the only module aware that a binary exists (§4).
Decision #5 bought that isolation on the argument that the renderer might have to
change; the renderer did not change, its *provenance* did, and the boundary
absorbed it anyway. §13's requirement that a missing binary produce an explicit
resolution rather than a stack trace is now the primary user-facing contract for
this dependency.

#### Development toolchain

Python 3.14, uv-managed. Development occurs inside `vrg-container-run` against
`dev-python`.

The dev dependency group is a **contract with `vrg-validate`**, not a preference:
the typecheck stage runs **`ty` and `mypy`**, and the audit stage runs
**`pip-audit` and `pip-licenses`**. Omitting any of them fails the stage with a
`FileNotFoundError` rather than a useful message. With pytest and ruff that is
six tools, not the three this section previously listed.

### Installation for daily use

Development happens in the container; **daily use does not**. The tool's whole
premise is one command each morning, and requiring a container to run it would
put a wrapper between the author and a thirty-second task.

Melete is therefore installed as a standalone tool on the host — `uv tool
install` from the repository — and run directly.

**LilyPond must be installed on the host separately**, since it is a binary
prerequisite rather than a bundled dependency. On Apple Silicon that is Homebrew
or the upstream `darwin-arm64` tarball. This is a genuine regression against the
original design, which intended `uv tool install` to be sufficient on its own;
see the amendment above and `melete#21`.

Because the prerequisite is invisible until something fails, §13's requirement
stands as the safeguard: a missing binary produces an explicit message naming the
resolution, never a stack trace.

Installing and confirming that first successful run is a deployment step
distinct from merging the code, and is tracked as such in the plan.

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
`docs/repository-standards.md`, `.github/workflows/ci.yml`, `.worktrees/`
gitignored, and the parallel-agent worktree section in `CLAUDE.md`.

`ci.yml` composes the shared `vergil-actions` reusable workflows at `@v2.1` —
`ci-audit`, `ci-quality`, `ci-security`, `ci-test` and `ci-version-bump` — and
declares no jobs of its own. What that produces is twelve required status
contexts on `develop`:

```text
quality / common               security / codeql        CodeQL
quality / lint / 3.14          security / semgrep       Semgrep OSS
quality / typecheck / 3.14     security / trivy         Trivy
test / unit / 3.14             version / version-bump
audit / dependencies / 3.14
```

The `matrix` and `evidence` jobs each shared workflow also emits run on every PR
and are **not** required. Melete's own
[`docs/repository-standards.md`](https://github.com/mnemosys-project/melete/blob/develop/docs/repository-standards.md)
carries this list, read from the branch's actual required contexts rather than
inferred from the workflow file, and is the place to look when it changes.

> **Corrected 2026-08-11.** This paragraph previously said `ci.yml` invokes
> `standards-compliance`. **It never has.** No such context appears among the
> required checks, among the seven further contexts that run unrequired, or as a
> job in the workflow. Sibling repositories document gates like `repo-profile`
> and `commit-lint`; melete has never had those either. The sentence was written
> as a description of scaffolding melete would receive and was never checked
> against the repository once it existed. Whether those gates are wanted here is
> a real question and a separate piece of work, filed as
> [`melete#79`](https://github.com/mnemosys-project/melete/issues/79) rather
> than left standing in this section as though it were already true.

Two hard gates are local rather than CI: branch-name validation and
Conventional Commits linting, both installed git hooks enforced through
`vrg-commit`. The agent session gate is `.claude/hooks/guard.sh`, a `PreToolUse`
hook that denies raw `git` and `gh` even when `vergil-tooling` is absent, so the
policy cannot be bypassed by an incomplete environment.

**`vrg-github-repo-config audit` does not evaluate rulesets**, and reported this
repository compliant throughout the bootstrap epic while the ruleset required a
context CI could not emit — the defect behind the `integration-tests` deviation
recorded in the plan. The required-check list above is therefore read from the
ruleset, not from a tool that reports compliance.

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
| 9 | ~~No key signatures by default~~ **Half-right; superseded by #27.** | Original reasoning: modal exercises should not imply a tonal center, and explicit accidentals force the reader to see each altered tone. The reasoning about *signatures* was sound and survives as `key_signatures = false`. What was wrong was the implementation: it conflated whether to print a signature with how to spell a note, and "notate in C with explicit accidentals" spelled F♯ Dorian as G♭ A♭ B𝄫 C♭ D♭ E𝄫 F♭ — a different key, not a neutral one. |
| 10 | Instructional prose on a cover page, not on the exercises | Text overlaid on engraved notation clutters the page and competes with the notes. |
| 11 | One combined `practice.pdf` per day | The printing unit is the day, not the exercise. |
| 12 | Organization bootstrap folded into this epic | The org is empty; `.github` is a hard prerequisite for the epic model. This is a from-scratch bootstrap, and splitting it would add ceremony without clarity. |
| 13 | ~~Accept the unmaintained `lilypond` redistribution; fork it if needed~~ **Superseded by #23.** | Original reasoning: it is the only way to install LilyPond into a virtual environment, and it is open source with a thin packaging layer. This assumed the risk was staleness. The actual defect was that the package has no aarch64 wheel at any version, which forking cannot fix for Linux because upstream publishes no `linux-arm64` binary either. |

### Resolutions from spec review

Decisions 14–19 were recorded during the `paad:pushback` review of this
specification on 2026-08-09, before implementation began. Each closed a gap that
would have reached an implementer as an open question.

| # | Decision | Rationale |
|---|---|---|
| 14 | `session.json` is self-sufficient for replay; `replay` and `--seed` are separate operations | Selection reads the session log, which is not a function of the seed, so a seed alone cannot reproduce a day once the log has grown. Recording the weight inputs makes the guarantee literally true and the sheet auditable. |
| 15 | One weighting expression with the floor applied last | The three-rule formulation contradicted itself at distances 0 and 1 and admitted an all-zero weight vector — an undefined draw on a legal config. One expression is monotonic, never zero, and keeps the policy locally tunable. |
| 16 | `Note.duration` is the written value; `Tuplet.ratio` supplies the scaling | Sounding durations inside a tuplet are not representable as noteheads (a triplet eighth is 1/12). Written durations keep the emitter a pass-through and pin the contract at the `score.py` seam, where `rhythm.py` and `emit.py` would otherwise diverge. |
| 17 | Bound exercise length with `max_notes` through the existing validity gate; accept the short final measure with `\bar "\|."` | Length varied roughly twentyfold across legal draws, making sheet size uncontrolled against a printability criterion. This is a bound, not a volume model — §17's deferral stands. |
| 18 | State `fret_count` for every built-in profile | It determines specification validity, therefore the candidate pool, therefore the draw. Inferring it would make the same seed produce different sheets on different installations. |
| 19 | One vocabulary registry for parameter identifiers and display names | §13's promise to name accepted values needs an enumerated set, and §12's cover page needs display names. Without one registry, config parsing, family dispatch, and the renderer drift apart. |

### Resolutions from alignment review

Decisions 20–22 were recorded during the `paad:alignment` check of this
specification against `plan.md` on 2026-08-09.

| # | Decision | Rationale |
|---|---|---|
| 20 | Tempo is a per-family default, overridable in config; never a sampled axis | `Score.tempo_range` had no producer anywhere — not an axis, not a config key — while §12's cover page prints a tempo per exercise. Sampling it would introduce difficulty variation, which is progressive overload and belongs to v2. |
| 21 | `ExerciseSpec` and `WeightInputs` live in `selection.py`, upstream of the families | §4's pipeline named `ExerciseSpec` but no module owned it. Placing it in `score.py` would blur the family/emitter seam; families keep the `params -> Score` contract of §7 and stay ignorant of the selector. |
| 22 | An explicit tuning that is not strictly ascending is a load error, never re-sorted | Index 0 is the lowest string and every family depends on the ordering. Sorting a malformed tuning would shift every string index and engrave the wrong instrument convincingly — the silent-failure mode §13 exists to prevent. |

### Resolutions from implementation

Decisions 23–26 were forced by contact with the work rather than by review, and
are recorded here so the reversals — and the rules the code adopted beyond what
the specification asked for — are legible.

| # | Decision | Rationale |
|---|---|---|
| 23 | LilyPond is a binary on `PATH`; melete has no runtime Python dependencies. **Supersedes #13.** | The PyPI redistribution has no aarch64 wheel at any version, so `uv sync` fails on both the arm64 container and the Apple Silicon host. Forking cannot fix Linux, since upstream publishes no `linux-arm64` binary. Taking the binary from the system works on both platforms today and cost one line, because `render.py` already isolated it. |
| 24 | Withdraw the "no change to the base image" claim, and treat the container gap as a Vergil-wide design problem rather than a melete workaround | A repo-specific system package is something Vergil's container model has never had to express — every image is generic and repo-agnostic. Solving it privately inside melete would hide a problem that the next such repository will hit. Filed as `vergil-tooling#2718`. |
| 25 | The dev dependency group is a contract with `vrg-validate`, not a style choice | Typecheck runs `ty` **and** `mypy`; audit runs `pip-audit` **and** `pip-licenses`. A missing tool fails the stage with `FileNotFoundError`, so the list is discovered by running the pipeline, not by preference. |
| 26 | `score.py` enforces §6's one-level-nesting rule at runtime: a `Tuplet` rejects any element that is not a `Note`, a `Score` rejects any voice element that is not a `Note` or a `Tuplet`, both raising `TypeError` naming the offending index | §6 states the rule but nothing checked it, and static typing does not close the gap: annotations are erased before any family runs, and families assemble voices dynamically from sampled parameters. A list built by appending in a loop is exactly where a type error slips past mypy. Without the check the failure surfaces as an `AttributeError` inside `emit.py` — far from its cause, in the module §4 keeps deliberately thin. The check is beyond what §6 specifies and was accepted deliberately rather than trimmed back to the letter of the spec. |

### Resolutions from the spelling review

Decisions 27–30 come from the 2026-08-10 brainstorm that reverted decision #9.
The trigger was a bug; the outcome is a model, because the bug turned out to be
a missing layer rather than a wrong setting.

| # | Decision | Rationale |
|---|---|---|
| 27 | Key signatures on by default, and notes spelled for the key. **Supersedes #9.** | A 12-TET integer cannot distinguish F♯ from G♭, so "notate in C with explicit accidentals" was never a neutral choice — it engraved F♯ Dorian as G♭ A♭ B𝄫 C♭ D♭ E𝄫 F♭, a different key, while the tablature stayed correct and disagreed silently. #9 conflated whether to print a signature with how to spell a note; separating the two makes both settings correct. |
| 28 | Three spelling tiers — diatonic, parented, symmetric — behind one entry point in `theory` | The 27 scale types do not admit a single rule: LilyPond has no key signature beyond the modes, and six-note and eight-note scales cannot use each letter exactly once. Tier 1 is determined; tiers 2 and 3 are convention and will be revised after instructor review, so the policy is structured to make that revision a single-function change — the same requirement §9 places on the selection weighting. |
| 29 | The tonic's letter is derived by fewest accidentals, never stored | Keeps `root` an integer in the families, the configuration and the selector, so exactly one derivation ever asks about letters. F♯ Dorian is four sharps and G♭ Dorian is eight flats; choosing the smaller signature gets it right without a spelling parameter that could be set wrong. |
| 30 | Chord qualities map to an implied parent scale rather than `Key` gaining a chord form | Chords are spelled by function, which is its own rule — but mapping `maj7` to ionian, `m7b5` to locrian and so on makes chord tones a subset of the parent's spelling. One mechanism serves scales and chords, and the arpeggios family sets `Key` exactly like the others. |

### Resolutions from the arpeggio table correction

Decision 31 was recorded on 2026-08-10, after S1 found §10a's implied-parent
table mapping `min6` and `min_maj7` to a parent that does not contain them. The
table correction itself is not a decision — the table was simply wrong, and it
is corrected in place — but the limit the correction exposed is one, because it
is a boundary the design accepts rather than a repair it defers.

| # | Decision | Rationale |
|---|---|---|
| 31 | Accept the fully diminished seventh's tier-3 spelling as a structural limit; do not add degree-aware chord spelling to correct it | `dim` and `dim7` are spelled by direction, so C dim7 spells `C D# F# A` rather than the functional `C Eb Gb Bbb`. Remapping cannot fix it: the chord needs a doubly diminished seventh and no seven-note scale supplies one, so no parent contains it. The functional spelling requires the speller to know that a pitch is *the seventh of this chord*, and `spell(key, pitches)` takes bare pitches and infers function from pitch-class membership — the narrow contract that lets one mechanism serve both scales and chords. Flagged for instructor review alongside the blue note. |

### Resolutions from the spelling-outcome review

Decisions 32 and 33 were recorded on 2026-08-10, from a review of what the §10a
model actually produces rather than of what it says. Neither is a repair
deferred to later: both are outcomes that look like defects to a reader who has
not seen the reasoning, and both were accepted so that the first instructor
review starts from the reasoning instead of rediscovering them.

| # | Decision | Rationale |
|---|---|---|
| 32 | Leave `dim` in tier 3 alongside `dim7`; do not remap `IMPLIED_PARENT["dim"]` to locrian | `A dim` therefore spells `A C D#` rather than the functional `A C Eb`. Unlike `dim7` this one is fixable — locrian contains both the ♭3 and the ♭5, names all three chord tones, and would satisfy the containment property — so the alternative is recorded as declined rather than absent. Remapping moves the quality between tiers and splits `dim` from `dim7`, which cannot follow it, and the judgment was to see how the raised fourth reads on an engraved sheet before changing the model to avoid it. Flagged for instructor review. |
| 33 | The tablature staff spells keylessly in both staff modes; accept the two staves naming one pitch differently in the source | `emit.py` passes `key=None` for the tab staff, so in a flat key the notation staff writes `ges` where the tab writes `fis`. A fret is a function of pitch and F♯ and G♭ are the same pitch, so a change of key cannot move a fret; feeding the tab the key's spelling would make its source vary with a change that cannot affect its content, which is what §14's byte-identical tablature assertion forbids. LilyPond derives identical frets from both spellings, so the cost is confined to a reader of the raw `.ly` — accepted rather than paid for with a coupling the boundary rules out. |

### Resolutions from the selector meeting the configuration

Decision 34 was recorded on 2026-08-10, when the implemented selector was run
against §10's own example configuration and raised on the first draw. The
example's incompleteness is corrected in place rather than recorded — it was
simply wrong — but the rule that exposed it is a decision, because the
alternative is superficially attractive and was rejected on its merits.

| # | Decision | Rationale |
|---|---|---|
| 34 | An axis a family reads but its `[pool.*]` section does not declare is a hard error naming the axis and the section; it is never defaulted | §13 already forbids falling back to a default for a *misspelled* key, and an *absent* one differs only in being easier to miss. `string_sets` is the axis that shows the fallback has no principled form: all 63 non-empty subsets of six strings is nonsense as a practice pool; contiguous-only invents a musical judgment the tool has no business making for the author; the full string set silently converts every positional exercise into a different one. Defaulting would fail in the mode this design consistently rejects — a plausible sheet the author did not ask for and cannot explain — and defaulting only the axes with an obvious guess would make the rule unpredictable. §10's example is corrected to declare every axis for the same reason: the example is what a reader copies. |

### Resolutions from the first printed sheet

Decisions 35–37 were recorded on 2026-08-10, after the first practice sheet was
generated, rendered and read. Three of its five exercises asked for a reach no
hand has, and one of those was labelled `positional`. Decision 37 is recorded as
**provisional**: it is a simplification accepted knowingly, with a named limit
and a tracked revisit, rather than a rule anyone believes is complete.

| # | Decision | Rationale |
|---|---|---|
| 35 | `positional` raises when the content cannot fit one position; bound the width of a position with `[instrument] position_span`, default 4 | The layout minimizes total fret travel and an argmin has no floor, so two octaves of a pentatonic across three strings returned the least bad answer and the family called it success — printing "G♭ major pentatonic, positional, ascending groups of 4" over a fourteen-fret reach. A mislabelled exercise is worse than an unplayable one, because the label is the part a student trusts. Four is the span between the outermost notes of one position: four fingers cover four frets and reaching one beyond is ordinary technique, so a position is five frets wide. It lives on the profile because fret spacing decides the reach, and because a `params -> Score` family is handed nothing else that could carry it. |
| 36 | Bound the reach of every traversal with `[session] max_fret_span`, default 12, through §9's existing validity gate | Nothing checked that a hand could get to a note the fretboard contained, and the first sheet held spans of 15 and 17 frets that nothing had decided were acceptable. Decision #17 added `max_notes` through the same machinery for the same class of problem — a specification that is legal but not useful — so this is a third predicate rather than a new concept. Twelve is one octave of neck, looser than a position on purpose: deliberate shifting is legitimate, and chromatic with `shift = fret_per_cycle` spans nine under §10's pool and must keep drawing. Past twelve an exercise covers more than the neck's own repeating unit and contains a repetition of itself. |
| 37 | **Provisional.** Span is measured between the lowest and highest *fretted* notes; open strings are excluded. Revisit tracked as `mnemosys-project/melete#60` | The justification is real: fret 0 sounds while the fretting hand stays put, and without the exclusion the open-A minor pentatonic box — open A against frets 3, 5 and 7 — is refused as a seven-fret stretch, when it is one of the most standard shapes on the instrument. But the rule is blunt and known to be blunt. It cannot distinguish an open string at the bottom of a low shape, where a three-note-per-string scale starting open is one position and entirely reasonable, from an open string interleaved with a hand high on the neck — playable, but a different kind of awkward the bound is not measuring. Accepted as a temporary simplification to get control of the span problem, to be revisited once there is experience of how the exercises play. Recorded so a future reader can tell this was a knowing simplification with a named limit, not a rule believed to be complete. |

### Resolutions from the scope-boundary review

Decision 38 was recorded on 2026-08-11, after §3's non-goals list was found to
be deciding an open design question by accident. It is recorded as a decision
rather than as prose in §3 because the specific correction is only half of it:
the general rule about boundaries stated more firmly than they were reasoned
has nowhere else to live, and a reader hitting the next such boundary needs the
rule, not the instance.

| # | Decision | Rationale |
|---|---|---|
| 38 | Split §3's non-goals into permanent exclusions and deferrals, and move audio, MIDI, playback and performance scoring from the first into the second | The two were one list ending in the sentence that made the database, server, API and web UI *permanently* out of scope, so audio inherited a permanence nobody had ever argued for. The refusal of the service infrastructure is the point of the project and is left firm. Audio is a different case: it is the input side of the retention thesis, which is the one claim the spec makes that nothing in v1 tests — decay is self-reported, and "played at 140" says nothing about how well. The correction was forced by a live decision rather than by tidiness. `mnemosys-project/melete#69` chooses between emitting written pitches under a plain clef and sounding pitches under an octavated clef, and sounding pitches are what an audio or MIDI comparison would need; under §3 as written that argument counted for nothing, because the capability was banned. A scope boundary stated more strongly than its reasoning supports does not sit inert — it quietly makes downstream decisions on the strength of an adjective, and this one already had. |

### Resolutions from the closing sweep

Decisions 39–41 were recorded on 2026-08-11 by the documentation review that
closes this epic. None of them changes what shipped. Each records something the
epic decided in code, or in an issue, without writing it into the specification
— which is precisely what a closing documentation bookend exists to catch.

| # | Decision | Rationale |
|---|---|---|
| 39 | An octave transposition has **exactly one owner**, and melete is it: `emit.py` writes the *printed* pitch, an octave above the IR's sounding pitch, under a plain `\clef "bass"` or `\clef "treble"` | Bass guitar sounds an octave below written, and both conventions for expressing that are correct — the application transposes under a plain clef, or the renderer transposes under an octavated one. What is not correct is applying it twice, which is what melete did for the whole of Phase B: it added 12 to every note *and* wrote `\clef "bass_8"`, which **performs** a transposition rather than describing one. **Every exercise engraved two octaves above its sound.** The tablature was right throughout, 2,700 tests at 100% branch coverage passed, and the defect was found only by rendering a page and looking at it (`melete#58`, fixed in `melete#66`). The IR stays in sounding pitch; the +12 lives inside `emit.py` alongside the matching transposition of the `stringTunings` chord, and the two must move together because LilyPond derives frets from pitch against the declared tuning. `melete#69` proposed reversing this to sounding pitch under an octavated clef and is closed unbuilt, since the module is being replaced — but the question arrives again with the next renderer, and the general rule is the part that carries: **when both the application and the renderer can apply an octave and neither states that it does, the failure is silent and looks plausible.** |
| 40 | `replay` is read-back, not re-execution, and the limit is stated rather than left to be discovered | Decision #14 called `session.json` "self-sufficient for replay" without saying which of the two operations it meant. Read-back is what shipped and what is correct here — the exercises are recorded in full, so re-deriving them adds nothing to the reproduction, and the injection point re-execution needs does not exist in `selection.select`. The cost is that a selector change which would have drawn differently is invisible to replay. That is a real gap in what the test proves, and stating it is the difference between a known boundary and a guarantee quietly weaker than its name. See §9, *Replay is read-back, not re-execution*. |
| 41 | Mark the renderer boundary in the specification rather than leaving it implicit in the module layout, and delete none of the LilyPond material | The renderer is being replaced (`melete#71`) and nearly everything else in this design outlives it — but that was legible only to a reader who already knew which modules imported which. §4's *The renderer boundary* makes it explicit in both directions, and it corrects the "blast door" claim, which is true of a change of *distribution* and false of a change of *renderer*: `render.py` isolates the binary while `emit.py` isolates the syntax, and the second is the larger. The LilyPond-specific material is kept and labelled rather than removed, because it is the record of what was learned — the seven constructs, the string-numbering inversion, the `TabStaff` defaults, the octave trap in #39, and the finding that golden-file tests pin generated text rather than its correctness. A migration that has to rediscover all of that pays for this epic twice. |

### Resolution from the reference-documentation sweep

Decision 42 was recorded on 2026-08-11, after melete's CLI and configuration
references were written **from the source** and found seven places where this
specification described behaviour the code does not have. All but one were
corrections to prose — a stale sentence, an example that does not run, a rule
stated more strongly than the code enforces it — and are recorded in the
sections they belong to rather than here. One was a real design choice that had
been made in code and written down nowhere, which is what the decision table is
for.

| # | Decision | Rationale |
|---|---|---|
| 42 | `[output]` is excluded from the configuration hash the seed derives from | The hash exists so that a day re-generated after the pool was edited is a different draw rather than the same one against a pool that no longer means the same thing. `[output]` does not move the draw — it chooses how a drawn exercise is engraved — so folding it in would make `staves = "tab"` hand back a *different set of exercises*, silently, from a key that a reader would reasonably expect to be presentational. That is the same class of failure as the octave in #39 and the spelling in #27: an output-side setting reaching back into correctness with nothing declaring that it does. It is also what makes §11's two overrides behave differently and defensibly — `--staves` cannot change what you practise and `--count` can, because `[session]` *is* in the hash. The choice was load-bearing and was recorded only as a comment in `session._fingerprint`, so a future edit adding "just one more field" to the fingerprint had nothing to read. |

## 17. Deferred to v2

The following are explicitly planned but out of scope, and the v1 design leaves
room for each:

- **Practice logging and note-taking.** How a session is recorded after it is
  played: what was practiced, at what tempo, how it went. The author expects
  this to be the v2 brainstorm.
- **Progression and progressive overload.** Tempo and complexity advancing over
  time. Requires the logging layer first; the session log is its substrate.
- **Measurement dimensions.** Mastery estimation, rolling volume, fatigue
  budgeting — the full MNEMOSYS state model. Every one of these needs a measure
  of how well an exercise was played, and the logging layer above supplies only
  what the author reports.
- **Performance capture.** Audio and MIDI input, playback, and scoring a
  recorded performance against the exercise that produced it. This is the input
  side of the measurement dimensions: "how well did you play it" is the question
  a mastery estimate is an answer to, and an honour-system log is a weak source
  for it. Recorded as a direction rather than a plan — the point is that the v1
  architecture should not be built in a way that forecloses it, which is why §3
  lists it as deferred rather than excluded.
- **Additional families.** Voice-led arpeggios (A3), grouping-based patterns
  (P3), legato mechanics (T5), and others from the canonical library.
- **`aoede`** — repertoire management, descended from the MNEMOSYS RPM design.

---

**Status:** Design approved 2026-08-09. Filed as epic
[`mnemosys-project/.github#1`](https://github.com/mnemosys-project/.github/issues/1)
on 2026-08-09. **Delivered**: melete v1 shipped, was deployed to the author's
host and validated on printed paper (`melete#19`, `melete#20`). Reviewed and
amended by this epic's closing documentation sweep on 2026-08-11, and corrected
again the same day against melete's source when its CLI and configuration
references were written from the code (`melete#75`, `melete#76`; corrections in
`.github#44`).

The renderer this specification describes is being replaced; §4, *The renderer
boundary*, says what that reaches and what it does not. Everything outside that
boundary stands as written.

The authoritative naming convention referenced in §2 lives at
[`NAMING.md`](../../NAMING.md) in this repository.
[`spec-as-approved.md`](./spec-as-approved.md) is the frozen 2026-08-09 snapshot
and is never amended; this document is the living one.
