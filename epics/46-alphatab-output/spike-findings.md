# Spike findings — alphaTex bar/tie semantics & `Gp7Exporter` fidelity

> **Task 1 of epic `mnemosys-project/.github#46`** (tracked by `mnemosys-project/melete#84`).
> Every answer below was determined **by experiment** against a real
> `@coderline/alphatab` install, not by assumption.
>
> **Gate: PASSED — no disqualifier.** A human opened the exported `.gp` in
> Guitar Pro 8 across two passes and confirmed fidelity. The alphaTab/`.gp`
> architecture carries every element the port needs; Tasks 2–10 proceed on the
> syntax recorded here.

## Environment

- **alphaTab package:** `@coderline/alphatab` — **exact installed version `1.8.4`**
  (npm `latest` at spike time). This is the version to pin in Task 2's
  `melete-render/package.json`.
- Node.js v22, ESM (`.mjs`).
- Scratch prototype: `render.mjs` (stdin alphaTex → stdout `.gp`), plus
  `inspect.mjs` / `pitches-gp.mjs` / `diag.mjs` verification helpers.

## Confirmed alphaTab Node API

The plan's guess (`ScoreLoader.loadScoreFromBytes` for alphaTex,
`Gp7Exporter().export(score, null)`) was **partly wrong**. The verified API:

```js
import * as alphaTab from '@coderline/alphatab';

// PARSE alphaTex -> Score. Do NOT use ScoreLoader for alphaTex text; that is for
// detecting/loading binary score files. Use AlphaTexImporter directly:
const settings = new alphaTab.Settings();
const importer = new alphaTab.importer.AlphaTexImporter();
importer.initFromString(alphaTexString, settings);   // (string, Settings)
const score = importer.readScore();                  // throws on parse error

// EXPORT Score -> Guitar Pro 7/8 (.gp) bytes:
const bytes = new alphaTab.exporter.Gp7Exporter().export(score, settings); // Uint8Array
```

- Import paths: `alphaTab.importer.AlphaTexImporter`,
  `alphaTab.exporter.Gp7Exporter`, `alphaTab.Settings`.
- `export(score, settings)` returns a `Uint8Array`; `settings` may be `null` but
  passing the same `Settings` is fine. Write it with
  `Buffer.from(data.buffer, data.byteOffset, data.byteLength)`.
- **Parse errors do not throw with the detail attached to `.message`.** Diagnostics
  live on `importer.lexerDiagnostics`, `importer.parserDiagnostics`,
  `importer.semanticDiagnostics`, each an `AlphaTexDiagnosticBag` whose `.items`
  (or `.errors`) array holds `{code, message, start:{line,col}}`. The renderer
  **must** dump these to stderr on failure (no silent failure) — `render.mjs` does.
- `ScoreLoader.loadScoreFromBytes(Uint8Array, settings)` **is** the correct call
  for reading a `.gp` back (used by the round-trip check below).
- `.gp` output is a ZIP container (`PK\x03\x04` magic; entries `VERSION`,
  `Content/…`), i.e. the GP7/GP8 format. Guitar Pro 8 reads GP7 `.gp` files.

## Q1 — Does alphaTab auto-bar a duration stream? **NO. Manual `|` required.**

Feeding **8 quarter notes in 4/4 with no `|`** produced **one bar** containing all
eight beats (an overfull bar), not two bars:

```
:4 3.3 3.3 3.3 3.3 3.3 3.3 3.3 3.3      -> bars = 1  (8 beats in one bar)
3.3.4 3.3.4 3.3.4 3.3.4 | 3.3.4 3.3.4 3.3.4 3.3.4  -> bars = 2
```

alphaTab does **not** distribute beats into bars by time signature; the `|`
barline token is the only thing that starts a new bar.

**Implication for plan Task 5:** the barring pass is **real work, not a no-op.**
The emitter must place `|` itself, which means melete must compute bar boundaries
(the `bar()` pass) and — because there is no auto-tie either (Q2) — split
bar-crossing notes and emit explicit ties. Task 5 stands as written.

## Q2 — Tie across a barline: token, placement, and auto-tie? **Explicit `-` fret; NOT automatic.**

There is no auto-tie: since bars are manual (Q1), a note that would overflow a bar
simply sits in an overfull bar; alphaTab never splits or ties it for you.

A tie is expressed by writing a note whose **fret position is `-`** on the **same
string**, with its own duration, as the first beat of the next bar:

```
... 3.3.4 |            <- last beat of bar N (becomes the tie ORIGIN)
-.3.4 3.3.4 3.3.4 3.3.4  <- '-' on string 3 = tie DESTINATION, continues previous
```

Round-trip confirms: the origin note reports `isTieOrigin`, the `-` note reports
`isTieDestination`, same string, pitch carried over. The general beat shape is
**`-.<string>.<duration>`** (a beat with a single tie note).

- The note-effect forms `{t}` / `3.3.4{t}` do **not** parse in 1.8.4 (property
  error). Use the `-` fret-value form; it is the canonical alphaTex tie and is the
  natural output of a split-and-tie barring pass.
- For a note longer than one writable value, split into writable pieces and chain
  them: first piece is the sounded note, each subsequent piece is `-.<string>.<dur>`.

## Q3 — Exact alphaTex for each vocabulary element

The **single-note beat grammar** (verified) is:

```
<fret>.<string>{ note-effects }.<duration>{ beat-effects }
```

- **Note effects** (`lf`, `ac`, tie via `-` value) go in ONE brace block attached
  directly after `<fret>.<string>`, **before** the duration, space-separated:
  `4.6{lf 2 ac}.4`. Multiple separate `{}{}` blocks FAIL; combine into one brace.
- **Beat effects** (`tu`) go in a brace block **after** the duration:
  `5.3.8{tu 3}`.
- Putting a note effect after an inline duration (`5.3.4{lf 2}`) FAILS — the
  post-duration brace is a *beat* block and `lf`/`ac` are unknown there.
- Durations may instead be set globally with a `:N` duration-change token that
  persists until changed (`:8 5.3 5.3` = two eighths); inline `.N` is per-beat.

| Vocabulary element | Confirmed alphaTex | Notes |
|---|---|---|
| 6-string bass tuning (B0 E1 A1 D2 G2 C3) | `\tuning(C3 G2 D2 A1 E1 B0)` | **Listed HIGH string first** (C3) → low (B0). Round-trips to tuning MIDI `[48,43,38,33,28,23]`, `stringCount=6`. |
| string.fret note | `<fret>.<string>` e.g. `4.6` | **String is 1-based, 1 = HIGHEST string, 6 = lowest.** So melete IR string index → alphaTex string = `string_count - index` (same reversal as `lily_string_number`). |
| written duration | `.4` `.8` `.16` (inline) or `:4` (global) | `4`=quarter, `8`=eighth, `16`=sixteenth, `2`=half, `1`=whole; dotted via `{d}` beat effect. |
| tuplet (3:2) | `<beat>{tu 3}` | `{tu N}` uses default denominator (3→2, 5→4, 6→4, 7→4, 9→8…); `{tu N D}` for an explicit ratio. Per-beat property. |
| **nested tuplet** | *no nesting syntax* — **flatten** to per-beat cumulative ratio, e.g. inner triplet-of-16ths inside an outer triplet = `7.3.16{tu 9 4}` | See Q4 / caveat below. Sound is exact; there is **no** hierarchical bracket. |
| tempo | `\tempo(96)` | Round-trips to `score.tempo = 96`. |
| left-hand fingering | `<fret>.<string>{lf N}...` e.g. `4.6{lf 2}` | **`lf` numbering: 1=thumb, 2=index, 3=middle, 4=ring, 5=little** (letters `t i m a c` also accepted). melete's fret-hand finger (1=index …) must be **remapped** (e.g. finger→`lf finger+1`, or emit letters `i m a c`). |
| accent | `<fret>.<string>{ac}...` | `{ac}` = normal accent (`AccentuationType.Normal`); `{hac}` = heavy/marcato. |
| tie across barline | `-.<string>.<duration>` | See Q2. Explicit; not automatic. |
| clef (bass) | `\clef bass` (per-bar, in the bar stream) | See Q3b. Propagates to later bars; round-trips as `clef=F4`. Octave handling below. |

The metadata **parenthesized form** (`\tempo(96)`, `\tuning(C3 …)`) with no trailing
`.` is diagnostic-clean in 1.8.4; the older space form (`\tempo 96` + `.`) still
works but emits style warnings (codes 301/400). Prefer the parenthesized form in
the emitter.

## Q3b — Clef and octave: `\clef bass` (plain), sounding pitch shown directly

alphaTex exposes clef and octave control as **two independent per-bar directives**,
placed **in the bar content stream** (not the metadata header):

- **`\clef <value>`** — accepted names (verified against the installed enum map):
  `bass` (F4 clef), `treble` (G2), `alto` (C3), `tenor` (C4), `neutral`/`n`; the
  underlying enum names `c3 c4 f4 g2` also work, as do MIDI-line numbers.
- **`\ottava <value>`** — `8vb`, `8va`, `15ma`, `15mb`, `regular`. Sets the clef's
  octave transposition (`bar.clefOttava`), independently of the clef itself.

**The first candidate had no clef directive**, and Guitar Pro 8 defaulted to a
**treble** clef, putting every bass note far below the staff — so an explicit
**bass clef** is required.

**Chosen convention: `\clef bass` with NO ottava — the notation shows sounding
pitch directly.** This was decided by reviewing the actual Guitar Pro 8 rendering:
with `\ottava 8vb` the notes sat too high on the staff relative to their (low)
fret positions; a plain bass clef reads truer for this material. The emitter
supplies **sounding-pitch** tuning (`C3 G2 D2 A1 E1 B0`), so playback is correct
either way; dropping the ottava simply keeps written position equal to sounding
position.

**This is a deliberate re-evaluation of v1's LilyPond convention, not an
oversight.** melete's LilyPond emitter uses `\clef "bass_8"` (write 8va / read
8vb) — an octave-displacing bass clef. The spec's **LilyPond-suspect principle**
(spec §3) calls for re-examining core/notation decisions that existed to serve
LilyPond; the bass-octave displacement was flagged as debatable in the code
itself (`lilypond/emit.py`, decisions #58/#69; melete#69 raised the trade-off and
was closed unbuilt). The port takes the opposite, equally-defensible reading:
**emit sounding pitch on a plain bass clef.** The port's `.gp` output therefore
diverges from the v1 `.pdf` on octave *display* only — playback pitch is
identical. This is an input decision for the emitter (Task 6), recorded here so it
is not re-litigated as a bug during the port.

```
\clef bass 4.6{lf 2}.4 5.5.4 3.4{ac}.4 2.3.4 | ...
```

- The directive belongs at the **start of the first bar**; alphaTab **propagates**
  the clef to subsequent bars automatically (only re-emit on a change).
- **Round-trip confirmed:** the exported `.gp` re-imports with `clef = F4` and
  `clefOttava = Regular` on **all four bars**, note frets/pitches unchanged.
- **Emitter note (Task 6):** emit `\clef bass` once at the head of the first
  measure. `\ottava 8vb` remains available if a future exercise wants the
  octave-displaced reading; clef and ottava are independent tokens, so either
  convention (or a per-exercise choice, as the LilyPond `_clef` logic does) maps
  directly. The clef-selection *logic* is renderer-agnostic; only the token changes.

## Q4 — Nested tuplet: flattening strategy (representable in sound, not as a bracket)

alphaTab/Guitar Pro model a tuplet as a **flat per-beat** `(tupletNumerator,
tupletDenominator)` — there is **no bracket-grouping / nesting syntax** in alphaTex.
A nested tuplet therefore cannot be emitted as a nested structure. It **can** be
carried faithfully in *sound* by flattening each leaf note to its **cumulative**
ratio:

- Outer triplet 3:2 of eighths, whose first member is itself a triplet 3:2 of
  sixteenths. The inner sixteenths get cumulative ratio `9:4`
  (`(1/16)·(4/9) = 1/36`, three of them = 1/12 = one outer slot); the two plain
  outer eighths stay `{tu 3}` (=3:2). The whole group occupies exactly one quarter.
- Emitted: `7.3.16{tu 9 4} 7.3.16{tu 9 4} 7.3.16{tu 9 4} 5.3.8{tu 3} 5.3.8{tu 3}`.
- Round-trip confirms the exported `.gp` preserves `tuplet 9:4` on the sixteenths
  and `tuplet 3:2` on the eighths, and total bar duration is correct.

**This is a caveat, not a disqualifier, and not a port requirement.** The
durations/playback are exact, but Guitar Pro shows flattened tuplet numbers (a `9`
bracket over the sixteenths, a `3` bracket over the eighths) rather than a true
3-inside-3 nested bracket. The human reviewed this in Guitar Pro 8 and **dropped a
true nested bracket as a non-requirement** ("a nice-to-have; often a manual fix
anyway"). It is moot in practice: the port's families never emit nested tuplets.

### Is there ANY nested-tuplet-bracket control in alphaTex 1.8.4? — DEFINITIVELY NO

Searched the installed importer/model source (`dist/alphaTab.core.mjs`) and tested
candidates. Evidence:

- **A beat carries exactly one tuplet.** The model is a flat pair on the beat —
  `beat.tupletNumerator` / `beat.tupletDenominator` (defaults `-1/-1`). There is no
  second/nested level and no field for one.
- **Tuplet brackets are auto-derived, not authored.** `TupletGroup.check(beat)`
  extends a bracket over consecutive beats **only while**
  `beat.tupletNumerator === group[0].tupletNumerator && beat.tupletDenominator ===
  group[0].tupletDenominator`. Grouping is one flat level keyed on equal ratios;
  differing ratios start a new, sibling (not nested) group. Nesting is structurally
  unreachable.
- **No alphaTex token for grouping/brackets.** The full beat-property keyword set
  is `f fo vs v vw s p tt d dd su sd cre dec spd sph spu spe slashed ds glpf glpt
  waho wahc legatoorigin timer tu txt lyrics tb tbe bu bd au ad ch gr dy tempo hide
  volume balance tp default barre full rasg ot instrument percussion bank fermata
  beam` — `tu` (tuplet) is the only tuplet control and takes just `N` or `N D`;
  nothing expresses a bracket, a group, or a nesting level.
- **`beam` controls beams only, not tuplet brackets.** The `beam` beat property maps
  to `BeatBeamingMode` (`Auto` / `ForceSplitToNext` / `ForceMergeWithNext` /
  `ForceSplitOnSecondaryToNext`) plus `bu`/`bd` for beam direction — it changes
  how noteheads are beamed, never the tuplet bracket.

**Conclusion:** flatten-to-cumulative-ratio is the **only** faithful option in
1.8.4; a genuine nested tuplet bracket cannot be expressed. This never occurs in
ported output — the port's families do not emit nested tuplets (plan Task 4/5) — so
it matters **only** against Task 4's permissive lifted-nesting invariant, where the
emitter should flatten (rhythm exact, display flat) and a nested bracket, if ever
wanted, is a manual touch-up in Guitar Pro.

## Q5 — Round-trip structural check

Exported `spike-candidate.gp`, then re-imported it with
`ScoreLoader.loadScoreFromBytes` and asserted structure:

- **tracks = 1** (expected 1). ✓
- **bars = 4** (expected 4 — one per `|`-delimited measure, last measure needs no
  trailing `|`). ✓
- tempo = 96 ✓, title preserved ✓, tuning `[48,43,38,33,28,23]` / stringCount 6 ✓.
- **clef = F4 (bass), ottava = Regular on all 4 bars** ✓ (see Q3b — sounding-pitch
  display, no octave displacement).
- **Sounding pitch preserved**: bar 1 notes re-import as MIDI 27/33/36/40, exactly
  `B0+4 / E1+5 / A1+3 / D2+2`. Frets and strings intact; tuplet ratios intact;
  tie origin/destination intact.

This is the strongest verification available **without** Guitar Pro 8: real
alphaTex → real `Gp7Exporter` → valid `.gp` → re-parsed to the expected structure.
It is the pattern plan Task 9 should use for the black-box integration test
(`n.string` renumbers internally on GP re-import, so assert **fret + realValue**,
not the raw `string` index).

## Guitar Pro 8 fidelity — CONFIRMED (human, two passes)

A human opened the exported `.gp` in Guitar Pro 8:

- **Pass 1:** tuning, string/fret, durations, tuplets, fingering, accent, and the
  cross-barline tie all rendered correctly. Two refinements surfaced: the staff
  needed an explicit **bass clef** (GP8 had defaulted to treble), and the nested
  tuplet renders flattened.
- **Pass 2 (final):** re-opened the regenerated **plain bass-clef** `.gp` and
  confirmed the notes sit correctly on the bass staff with everything else intact
  ("Looks great"). The octave convention (plain bass clef, sounding pitch direct)
  and the flattened nested tuplet (non-requirement) were both accepted.

## Disqualifiers found: NONE

Every required vocabulary element is representable in alphaTex 1.8.4, survives the
`Gp7Exporter` round-trip structurally and in sound, and renders correctly in Guitar
Pro 8. **No hard disqualifier — the architecture assumption holds; Tasks 2–10
proceed.**

Two decisions recorded for the emitter (Task 6), neither a blocker:

1. **Octave convention:** plain `\clef bass`, sounding pitch shown directly — a
   deliberate re-evaluation of v1's `bass_8` displacement (Q3b). `.gp` diverges
   from the v1 `.pdf` on octave *display* only; playback pitch is identical.
2. **Nested tuplets** have no native bracket and are flattened to cumulative
   per-beat ratios (Q4) — faithful in sound, visually flat, dropped as a
   non-requirement. Moot in practice (families never emit nested tuplets); recorded
   against Task 4's lifted-nesting invariant.

## Notes for whoever opens the `.gp` in Guitar Pro 8

- File: `spike-candidate.gp` (final, **plain bass-clef** version); its source:
  `spike.atex`. A copy is at `mnemosys-project/spike-46/spike-candidate.gp`.
- Expect **one track, 6-string bass, 4 bars**, 4/4, on a **plain bass clef**
  (sounding pitch, no octave displacement).
- Bar 1: four quarters (fingering on beat 1, accent on beat 3).
- Bar 2: a 3:2 eighth triplet, then the flattened nested tuplet (three 9:4
  sixteenths + two 3:2 eighths), then two quarters — the nested tuplet shows as
  flat `9` and `3` brackets, not a nested bracket (confirmed unavoidable, Q4).
- Bars 3→4: the last note of bar 3 ties across the barline into bar 4's first note.
