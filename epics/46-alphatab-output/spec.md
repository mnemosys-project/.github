# Melete — alphaTab/`.gp` Output Pipeline (port from LilyPond)

**Epic:** [`mnemosys-project/.github#46`](https://github.com/mnemosys-project/.github/issues/46).
**Predecessor:** epic #1 (`epics/1-org-bootstrap-melete-v1/`), the v1 bootstrap
that built the generator and engraved through LilyPond. This epic is its planned
successor — the renderer migration epic §4 of epic #1 anticipated.
**Date:** 2026-08-11.

## Table of Contents

- [1. Overview](#1-overview)
- [2. Scope and non-goals](#2-scope-and-non-goals)
- [3. Strategic decisions](#3-strategic-decisions)
- [4. The renderer boundary (building on epic #1 §4)](#4-the-renderer-boundary-building-on-epic-1-4)
- [5. Architecture](#5-architecture)
- [6. IR evolution and the barring pass](#6-ir-evolution-and-the-barring-pass)
- [7. The renderer (`melete-render`)](#7-the-renderer-melete-render)
- [8. Removing LilyPond](#8-removing-lilypond)
- [9. Testing strategy](#9-testing-strategy)
- [10. Task breakdown and bookends](#10-task-breakdown-and-bookends)
- [11. Risks, assumptions, open questions](#11-risks-assumptions-open-questions)

## 1. Overview

Melete generates daily practice sheets. v1 (epic #1) engraves through LilyPond
and is proven. But LilyPond has no semantic support for the advanced guitar
techniques melete exists to serve — two-handed tapping foremost, and the wider
set of harmonics, slides, bends, and right-hand fingering that Guitar Pro
represents natively. The evaluation that decided the renderer must be replaced is
recorded in `melete#71`; the successor survey is in melete's `docs/reports/`
(melete#65/#67/#68).

The anchoring insight, established in epic #1 §4: **v1's value is the
generation/management logic, not the LilyPond output.** Nearly the whole design
is renderer-agnostic; LilyPond is confined to two modules and their golden files.
This epic replaces that renderer with a pipeline that emits **alphaTex** text and
runs **alphaTab** to produce **Guitar Pro `.gp` files**.

**Deliverable and its acceptance.** A `generate` run produces a day's practice
as loadable `.gp` file(s). Because the generation core is unchanged, a given seed
and config produce the **identical `Score` IR** as v1 *by construction* — so the
deliverable is not a fuzzy v1-vs-v2 comparison. The only thing this epic can get
wrong is the **emit + render**, so acceptance lives there: the alphaTex golden
tests assert the content, and the `Gp7Exporter` smoke test confirms the `.gp`
faithfully carries it. An informal side-by-side with a v1 sheet is a sanity
check, not the gate.

## 2. Scope and non-goals

**In scope — this epic is a port.**

- Redo current v1 functionality (the `scales`, `arpeggios`, `intervals`,
  `chromatic` families on the current bass instrument) through the new pipeline.
- Emit alphaTex; render `.gp` via `melete-render` (Node/alphaTab).
- Evolve the Score IR only as the `.gp` target forces (measures; nested tuplets).
- Remove LilyPond **as the final task**, once the new pipeline is green (§8).
- Sync `CLAUDE.md` and `docs/design.md` to the new reality.

**Non-goals (deliberately deferred).**

- **No tapping family and no guitar instrument** — those are the next epic,
  seeded here as a follow-on brainstorm (melete#82). This epic ports what exists.
- **No image (SVG/PNG) or PDF output** — output is `.gp` only. Sequencing is
  `.gp` now → images later → PDF maybe-later-and-cheap.
- **No native-TypeScript rewrite** — kept as the probable long-term move,
  decided empirically (§3).

## 3. Strategic decisions

- **Runtime = A (Python core + a thin `melete-render` tool).** Keep the
  well-tested Python core; add a Node/alphaTab renderer. A native-TypeScript
  rewrite (B) is the probable long-term move; C (hand-writing `.gp` in Python) is
  rejected — it discards alphaTab's rendering, half the reason to adopt it.
- **The interface is alphaTex text.** Python emits an alphaTex string; the
  renderer consumes it and returns `.gp`. The narrowest possible boundary — text
  over a subprocess, the exact shape of today's LilyPond blast door. Python tests
  assert the alphaTex we emit; the renderer is a black box (fits the current
  absence of TypeScript testing infrastructure).
- **The B-signal.** How hard the emitter strains against "just text" is the
  standing metric that decides the TypeScript rewrite. It is reviewed at each
  checkpoint. The first real stress test is the tapping family (next epic).
- **Output = `.gp` only** for this epic; the alphaTex source is kept on disk
  (mirroring the kept `.ly`); the session log is unchanged.
- **The LilyPond-suspect principle.** Any core decision that exists to serve
  LilyPond is re-evaluated (§4, §6), not inherited.

## 4. The renderer boundary (building on epic #1 §4)

Epic #1 §4, *The renderer boundary*, is the technical asset this epic starts
from. It classifies the design into what survives a renderer change and what is
LilyPond-specific. That classification stands, with **two corrections** this
migration surfaces — both instances of a LilyPond-imposed constraint having been
recorded as if it were renderer-agnostic.

### Corrections to epic #1 §4

1. **The one-level tuplet-nesting rule does NOT survive.** Epic #1 §4 lists it
   under *what survives the renderer change*, but `score.py` justifies it as
   *"a tuplet inside a tuplet has no LilyPond construct to map onto"* — a pure
   LilyPond limitation. Guitar Pro and alphaTab represent nested tuplets, and the
   maintainer has a real exercise that needs them. **The migration lifts it**
   (§6); that exercise is the acceptance case.
2. **"Measures are not modelled" does NOT survive.** The IR is measure-less
   because *"LilyPond inserts barlines"* (§6 of epic #1). alphaTex and `.gp` are
   bar-oriented. **The migration adds a measure model and a barring pass** (§6).

Everything else epic #1 §4 lists as surviving does survive: the 12-TET theory
and spelling model (`SpelledPitch` is notation-neutral by construction), the
instrument/fretboard model, the written-duration contract, all four families and
their axes, the rhythm modifier, coverage-aware selection and determinism, the
config, the session log and replay, the CLI, and error handling.

### What is renderer-specific (and, this epic, removed)

Per epic #1 §4, the LilyPond-specific surface is confined to: `lilypond/emit.py`
(syntax), `lilypond/render.py` (the binary), the golden `.ly` files, and the
LilyPond-shaped notation choices (`\tabFullNotation`, clef selection, mode
keywords, `\bar`, `\accidentalStyle`, `\tuplet`, the cover `bookpart`, the binary
prerequisite). Epic #1 §4 measured this at ~210 statements across the two
modules plus the golden files, in a codebase otherwise renderer-agnostic. This
epic replaces that surface and, in its final task, removes it (§8).

**`score.py` remains the renderer-agnostic seam.** Families produce a `Score`;
the emitter consumes one; neither imports the other. No alphaTab or `.gp`
knowledge leaks past it — the same law that kept LilyPond out of the core.

## 5. Architecture

The generation core is preserved; only the output half changes.

```text
families ─▶ Score IR ─▶ bar() ─▶ measures ─▶ emit ─▶ alphaTex ─▶ [ melete-render ] ─▶ .gp
 (kept)    (evolved)  (new,pure)  (new)     (new)     text          Node/alphaTab
                                                             THE INTERFACE (thin, text, black box)
```

- **Kept, untouched:** `theory`, `selection`, `rhythm`, `families/*`, `session`,
  `config`, `instrument`, `vocabulary`, `cli`.
- **Evolved:** `score.py` — add a `Measure` model; lift the tuplet-nesting rule.
- **Added:**
  - a pure **barring pass** `bar(voice, time_signature) -> list[Measure]`;
  - an **alphaTex emitter** (`Score`/measures → alphaTex text) — the replacement
    for `lilypond/emit.py` and the locus of interface complexity;
  - the **Python blast door** shelling out to the renderer (replaces
    `lilypond/render.py`);
  - the vendored **`melete-render/`** Node tool (§7).
- **Removed (final task, §8):** the `lilypond/` package and its golden tests, the
  LilyPond container capability, and LilyPond framing in the docs.

The session/directory model (§12 of epic #1) is preserved in spirit: a run keeps
the **alphaTex source** on disk (mirroring the kept `.ly`) and emits the **`.gp`**
(mirroring the `.pdf`), plus the unchanged session log.

## 6. IR evolution and the barring pass

"Measures are not modelled" was LilyPond doing bar-splitting **and**
cross-barline ties for free. alphaTex is bar-delimited (`|`) and will not; and v1
exercises can cross barlines (odd groupings against the meter). So bar-splitting
plus tie-across-barline logic must live somewhere.

**Decision (of three approaches considered): a barring pass between families and
emitter.** Families keep producing a flat `Voice` (**core untouched**). A single
pure, heavily-tested function `bar(voice, time_signature) -> list[Measure]` owns
all split-and-tie logic, including nested tuplets and the short final measure.
The emitter consumes tidy measures.

Rationale: it protects the robust core (no family changes), models measures where
the bar-oriented target needs them, and quarantines the one genuinely tricky
piece in a pure function with its own test suite — the seam style this codebase
favors — reusable by every future emitter.

Rejected alternatives: (1) emit-time barring inside the text generator — tangles
the hardest logic into text emission and is not reusable; (2) measures modelled
in the IR with families producing them — touches every family, the core we
protect, for a port.

**Nested tuplets** are lifted in the same IR change: the `Tuplet`-cannot-hold-
`Tuplet` check and the single-level-nesting invariant in `score.py` are removed,
with the maintainer's nested-triplet exercise as the acceptance case.

**A tie representation is part of this IR change, not an afterthought.**
Split-and-tie has no output vocabulary today: `Note` carries no tie field, so a
note the barring pass splits across a barline would emit as two independent notes
and the `.gp` would *re-articulate* it (a half note across a barline becomes two
attacks — wrong rhythm and sound). The measure model therefore carries a `tied`
marker on the split fragment ("tied to the following note"), which the emitter
maps to alphaTex's tie token. In v1 LilyPond invented these ties from durations;
now that melete bars, melete must record them. The exact form is confirmed by the
spike (below), but the IR evolution must add it.

**The IR change is additive, so the still-present LilyPond emitter stays green.**
§8 keeps LilyPond working until the final task, and `lilypond/emit.py` consumes
`score.py`. The evolution is therefore constrained to be backward-compatible:
`Measure` is a new type consumed only by the new emitter; new `Note` fields
(`tied`) are default-valued; families keep producing the flat `Voice`; and the
nested-tuplet lift is *permissive* (allowed by the IR, not produced by the port's
families — verified: no family emits nested tuplets today). So `lilypond/emit.py`
and its golden tests stay green untouched until removal (§8). A restructuring
change that reshaped `Voice`/`Tuplet` or made families emit `Measure`s would break
LilyPond mid-epic and is out of bounds.

**A spike precedes and shapes this IR/barring design.** Whether alphaTab
auto-bars a duration stream or requires exact bar layout, and how it expresses a
tie across a barline, are unresolved (§11) — and they determine how much of
`bar()` and the tie representation is even needed. The first implementation task
(§10) is a spike: hand-write alphaTex for a representative exercise (bass tuning,
a tuplet, a note crossing a barline), run it through `Gp7Exporter`, and open the
`.gp` in Guitar Pro 8. Its findings fix the shape of the `Measure` model, the tie
marker, and `bar()` — possibly shrinking them — before any of that is built.

## 7. The renderer (`melete-render`)

**Acknowledged, tracked technical debt.** A vendored, untested Node/alphaTab
snippet inside a Python repo is debt. It is designed to be small, isolated, and
extraction-ready, and its extraction is a scheduled item — not a silent
liability.

- **Contract (stable, minimal):** `alphaTex` on stdin → `.gp` bytes on stdout;
  non-zero exit + stderr on failure. This is the exact contract a future
  standalone package would honor unchanged.
- **Dumb-as-possible:** read alphaTex → alphaTab importer → `Gp7Exporter` → write
  bytes. **No musical logic, no decisions, no config.** All intelligence lives in
  the tested Python emitter and barring pass. Target: a few dozen lines. The
  dumber it is, the less its untested-ness costs — the "keep it as stupid as a
  shell script" bar, applied honestly to a more capable language.
- **Designed to leave:** a self-contained `melete-render/` directory with its own
  `package.json` pinning alphaTab, and a README declaring it extraction-bound
  debt. Python reaches it only through the process boundary — never imports,
  never patches internals — so extraction later is a one-line invocation change.
  The **extraction disposition** — defer, expand into a follow-on epic, or drop —
  is owned by the follow-on brainstorm bookend (melete#82), decided before the
  retrospective; cross-org linking to the Vergil TypeScript initiative is out of
  scope, so that dependency is recorded in prose.
- **Name:** `melete-render` / "the renderer" is a working name, to be revisited.
  It pairs with the Python "emitter" (emit → render).
- **Errors (blast-door contract, as `render.py` today):** on failure, keep the
  alphaTex on disk, surface alphaTab's stderr verbatim, and give a missing-Node
  error naming the resolution (mirroring the missing-`lilypond` message). No
  silent fallback.

## 8. Removing LilyPond

Epic #1 §4 framed the LilyPond code as *"not deleted... the input the migration
starts from."* **This epic consciously supersedes that stance and removes it**,
for minimal complexity: the maintainer has decided LilyPond's output is not
adequate for advanced guitar and that the codebase should not carry a renderer it
will not use. Two safeguards preserve everything of value without keeping the
code in the working tree:

1. **Removal is the final task**, done only after the new alphaTab pipeline is
   green — `generate` is never without a working renderer. This is only
   achievable because the IR evolution is **additive** (§6): `lilypond/emit.py`
   and its golden tests stay green, untouched, through every task until this one.
2. **The comparison baseline is preserved as data and history:** one committed
   sample of v1 output (a rendered `.pdf` and its `.ly`) is retained, and the
   pre-removal commit is **git-tagged**, so the A/B comparison and the code
   itself remain recoverable. "Revive if we want" stays true.

Removal deletes `lilypond/emit.py`, `lilypond/render.py`, `tests/lilypond/`
(golden files), the LilyPond container capability in `vergil.toml`, and updates
`docs/design.md` and `CLAUDE.md` (melete#83) to the new reality.

## 9. Testing strategy

- **Python (where the real logic is):** unit-test the **emitter**
  (`Score`/measures → alphaTex, golden/structural), the **barring pass**
  (split-and-tie across barlines, nested tuplets, short final measure), and IR
  validation (as today). The **100% coverage bar holds** for Python. This suite
  is also the instrument that measures interface complexity.
- **Renderer (black box):** one integration test, Node-gated (the `skipif`
  discipline used for `lilypond`), feeding known alphaTex through the renderer
  and asserting a **valid `.gp`** (parses, expected track/bar counts).
  `integration-tests` stays `false` until it exists.
- **Fidelity smoke test (manual, one-off):** confirm alphaTab's `Gp7Exporter`
  round-trips our features by opening an exported `.gp` in Guitar Pro 8 — the
  docs guarantee full fidelity explicitly only for the alphaTex *exporter*, not
  `Gp7Exporter`.

## 10. Task breakdown and bookends

Implementation tasks (filed from the plan; each ≈ one PR), in dependency order:

1. **Spike — validate alphaTex bar/tie semantics and `Gp7Exporter` fidelity.**
   Hand-write alphaTex for a representative exercise (bass tuning, a tuplet, a
   note crossing a barline), run it through `Gp7Exporter`, open the `.gp` in
   Guitar Pro 8. Resolve, before any IR change: does alphaTab auto-bar or require
   exact layout; how is a tie across a barline expressed; do the tuning and
   fingering round-trip. Its findings fix the shape of tasks 3–5 (and may shrink
   `bar()`). This is the interface-complexity learning, done first.
2. **Container capability: Node + alphaTab** — add the Node runtime and the
   vendored `melete-render/` package to the dev/CI container. (Adds alongside;
   the LilyPond capability is removed only in the final task.)
3. **The renderer (`melete-render/`)** — the dumb Node tool: alphaTex → `.gp` via
   `Gp7Exporter`; self-contained; README declaring extraction-bound debt.
4. **IR evolution (additive)** — add the `Measure` model and a `tied` marker
   (§6); lift the tuplet-nesting rule (permissive); nested-triplet test. Shaped
   by the spike; must not touch `lilypond/emit.py`'s inputs.
5. **The barring pass** — pure `bar(voice, time_signature) -> list[Measure]`,
   including split-and-tie and the short final measure.
6. **The alphaTex emitter** — `Score`/measures → alphaTex text.
7. **The Python blast door** — shell out to the renderer; blast-door error
   contract.
8. **Wire CLI/session** — `generate` produces a day's `.gp` + kept alphaTex +
   session log.
9. **Black-box integration test** — alphaTex → renderer → valid `.gp`.
10. **Remove LilyPond (final)** — delete the LilyPond surface, remove its
    container capability, preserve the sample + git tag, sync `docs/design.md` and
    `CLAUDE.md` (melete#83).

**Bookend tasks (seeded at epic creation):**

- Documentation (this spec + plan) — `.github#47`.
- Documentation-review (closing bookend) — melete#81.
- Retrospective (terminal bookend) — `.github#48`.
- Fix `CLAUDE.md` de-pollution — melete#83.
- Follow-on brainstorm (tapping family + `melete-render` extraction disposition;
  the tapping work becomes the *next* epic, minted when that brainstorm runs) —
  melete#82.

**`melete-render` extraction disposition:** owned by the follow-on brainstorm
bookend (melete#82) — defer / expand into a follow-on epic / drop — decided
before the retrospective, dependent on the Vergil TypeScript-support initiative.

**Cold-rebuild validation:** a `validation` task is filed (blocked-by Tasks 2 and
8) to prove the new container capability on a fresh image end-to-end — Node +
alphaTab present and `generate` produces a valid `.gp` — catching image-build
regressions CI's cached layers can hide.

## 11. Risks, assumptions, open questions

The first three are **resolved by the task-1 spike** (§10) before any IR or
barring code is written — that is the whole reason the spike is task 1:

- **alphaTex expressiveness for the port's features** — bass tuning, string/fret,
  left-hand fingering, tempo, key/accidentals, tuplets (incl. nested), the short
  final measure. *Resolved by the spike.*
- **alphaTab `Gp7Exporter` fidelity** — round-trip our features into a `.gp` that
  opens cleanly in Guitar Pro 8. *Resolved by the spike.*
- **alphaTex bar/tie semantics** — exact syntax for bars (`|`) and ties across
  barlines, and whether alphaTab auto-bars or requires exact layout. *Resolved by
  the spike; it fixes the shape of the `Measure` model, the `tied` marker, and
  `bar()`.*
- **Node in the container** — version, alphaTab pinning, headless `.gp` export
  (no rendering engine needed for `Gp7Exporter`). *Confirm.*
- **Interface-complexity metric** — the standing A-vs-B signal, reviewed each
  checkpoint.
- **Re-scout alphaTab for disqualifiers** — the best option today, not the only
  one; keep looking while building.
