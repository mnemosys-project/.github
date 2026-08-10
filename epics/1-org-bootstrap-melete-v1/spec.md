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

```
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

```
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

### Validity

Validity is a hard gate, not a weight. Each sampled specification is checked
against the active instrument profile **and against `max_notes`** (§7). Invalid
specifications are resampled up to a bounded retry count; exhausting retries is a
loud error naming the over-constrained axis, never a silent fallback.

Both checks run through the same gate. A cycle that is too long and a string set
the profile cannot supply are the same kind of failure — a specification the
instrument or the session cannot accommodate — and neither is ever quietly
adjusted into something renderable.

### Determinism

The seed derives from the date plus a hash of the configuration and is written
into `session.json`. Randomness is real but never irreproducible.

**A seed alone is not sufficient to reproduce a day, and the design accounts for
this.** Selection depends on the session log — the weights above are computed
from how many sessions have passed since each value was last used — and that
history is not a function of the seed. Replaying a seed against today's log
computes different weights than the original run did, because the log has grown
since. The result would be a different sheet, produced silently, with no
indication it had diverged.

`session.json` is therefore **self-sufficient for replay**. Alongside the seed
and the configuration hash it records the resolved weight inputs — the
`sessions_since` distance per candidate value per axis — that fed that day's
draw. Replay reads those recorded inputs rather than recomputing from the live
log:

| Invocation | Behavior |
|---|---|
| `melete generate` | Normal generation; computes weights from the current log. |
| `melete generate --seed <n>` | Same seed against the **current** history. Not a reproduction, and not described as one. |
| `melete replay <date>` | Exact reproduction from that session's recorded seed and weight inputs. |

Separating the two is what makes the guarantee literally true. `--seed` remains
useful for exploring a draw; `replay` is the operation that reproduces a sheet.
Recording the weight inputs is a small extension of what §12 already commits
`session.json` to holding, and it makes any past sheet auditable — not merely
regenerable — because the inputs that produced it are on disk next to the output.

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
key_signatures = true                # print the key signature (section 10a)

[session]
count = 5
horizon = 14
max_notes = 96                       # per-exercise length bound (section 7)
shape = { chromatic = 1, scales = 2, arpeggios = 1, intervals = 1 }

[pool.scales]
roots = "all"
scale_types = ["ionian", "dorian", "phrygian", "major_pentatonic", "blues"]
patterns = ["straight", "thirds", "groups_of_3", "groups_of_4"]
traversals = ["positional", "three_note_per_string"]
octaves = [1, 2]
tempo = [80, 100]                    # overrides the family default (section 7)

[pool.rhythm]
subdivisions = ["eighth", "triplet_eighth", "sixteenth"]
accent_patterns = ["none", "every_3"]
note_value_patterns = ["straight", "long_short"]
```

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

This carries a genuine emitter consequence: LilyPond's `TabStaff` suppresses
stems and beams by default, assuming a notation staff above supplies the rhythm.
In `tab` mode the emitter must therefore **explicitly enable rhythm display** —
Guitar Pro-style tablature with stems — or the exercise is unreadable. In `both`
mode plain tablature is correct. This is a branch in the emitter, not a flag.

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
| `min`, `min6`, `min7`, `min_maj7` | aeolian | 1 |
| `dom7` | mixolydian | 1 |
| `m7b5` | locrian | 1 |
| `dim`, `dim7` | diminished | 3 |
| `aug` | whole-tone | 3 |

Chord tones then fall out as a subset of the parent's spelling, one mechanism
serves both scales and chords, and the arpeggios family sets `Key` exactly like
the others.

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

### Why the policy sits behind one entry point

Tier 1 is fully determined. **Tier 2 is conventional practice and tier 3 is a
defensible convention rather than a rule** — and both will be reviewed by a
reader with formal training once real sheets exist. Blue-note spelling, the
diminished scales, and whether modal material should carry a signature at all
are exactly the questions that will come back with corrections.

So the three tiers live in one module behind one entry point, for the same
reason §9 requires it of the selection weighting: the revision we are expecting
should be a small local change, not a refactor. A policy scattered across four
family modules would not survive its first review.

## 11. Command-Line Interface

```
melete generate                  # today's session
melete generate --date 2026-08-10
melete generate --seed 12345     # fixed seed against current history
melete generate --dry-run        # print selections, render nothing
melete generate --staves tab     # override staff mode for one run
melete generate --count 6
melete generate --force          # overwrite an existing session directory
melete generate --split          # also emit one PDF per exercise
melete replay 2026-08-09         # reproduce a past session exactly
melete show 2026-08-09           # summarize a past session
melete families                  # list families and their parameter axes
melete vocabulary                # list every axis and its accepted values
```

`replay` and `--seed` are deliberately distinct; see §9 *Determinism*. `replay`
reconstructs a session from its recorded seed and weight inputs and is the only
operation that reproduces a sheet exactly. `--seed` fixes the draw against
whatever history exists now.

`vocabulary` prints the canonical registry described in §13 — the same source
that configuration validation and the cover-page renderer read.

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

`session.json` records the seed, the configuration hash, the full parameter
dictionary for every exercise, and the **resolved weight inputs** that produced
the draw (§9). It is the history the selector reads back. It is human-readable,
git-committable, and hand-editable.

The weight inputs are what make the file sufficient for exact replay rather than
merely descriptive of the result. They also make a sheet auditable: the question
"why did it pick D Dorian three days running?" is answerable from the file
itself, without re-deriving anything.

## 13. Error Handling

No swallowed exceptions and no fallbacks that hide errors. A failure in the
generator becomes a wrong exercise on the page, which is worse than no exercise.

| Failure | Behavior |
|---|---|
| Malformed or invalid configuration | Fail at load, naming the exact key and its accepted values. Never fall back to a default for a misspelled key. |
| Explicit tuning not strictly ascending | Fail at load, naming the offending index. Never re-sort — sorting would shift every string index and engrave the wrong instrument convincingly. |
| Pool over-constrained | Hard error naming the axis that could not be satisfied — for example, "no valid `string_set` for `bass4` with `octaves = 3`". |
| Family emits a note outside the fretboard | A bug, not user error. Raise. |
| LilyPond render fails | Surface LilyPond's stderr verbatim and **keep the generated `.ly` on disk** for inspection and manual re-run. Never clean up on failure. |
| LilyPond binary missing | Explicit error stating the resolution, not a stack trace. |
| Session directory exists | **Refuse.** `--force` overwrites and must be asked for explicitly. |
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
| `theory.py`, `instrument.py` | Exhaustive. All 12 roots against all ~28 scale types, verified against known interval content. Every pitch maps to valid positions on every profile. |
| Families | Property-based over a wide parameter sweep. |
| `rhythm.py` | **Sounding** durations (§6) of a voice sum to the pattern's cycle length; every written duration is a representable notehead; tuplet ratios well-formed. |
| `vocabulary.py` | Every identifier used in §7, §8, and the §10 example config resolves; every axis value has a display name. |
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

```
spelled pitch class == note.pitch % 12
```

A spelling that does not sound the note it names is the failure mode, and it is
precisely the bug §10a exists to fix.

Alongside it:

- **The letter rule** — tier 1 and the seven-note tier 2 scales use seven
  distinct letters. Tier 3 is explicitly exempt, and the exemption is asserted
  rather than assumed.
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
first run. This is the test that proves §9's reproducibility guarantee holds
against a log that has moved on, and it is the reason the weight inputs are
recorded rather than recomputed.

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
not and should not carry. Vergil has no mechanism for repo-specific system
dependencies today; the design problem is filed as
[`vergil-project/vergil-tooling#2718`](https://github.com/vergil-project/vergil-tooling/issues/2718),
and publishing our own aarch64 wheels — which would restore the original
property — is [`mnemosys-project/melete#21`](https://github.com/mnemosys-project/melete/issues/21).

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
