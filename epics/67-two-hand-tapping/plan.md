# Two-handed tapping Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> `superpowers:subagent-driven-development` (recommended) or
> `superpowers:executing-plans` to implement this plan task-by-task. Steps use
> checkbox (`- [ ]`) syntax for tracking.

**Epic:** [`mnemosys-project/.github#67`](https://github.com/mnemosys-project/.github/issues/67)
**Spec:** [`spec.md`](./spec.md) (v2.0 rebase)

**Goal:** Add two-handed tapping to melete as a **tabulated triad tap-shape
vocabulary** for the `arpeggios` family, so `arpeggios` exercises can be realized
as two-hand tapped triads walked up and back across the neck.

**Architecture:** A curated tap-shape table (one two-hand choreography per triad ×
inversion) drives a tapped-journey path inside the `arpeggios` family. Placement
happens at family generation (where the inversion is known), realized through
`box`'s reserved two-anchor path in `_shared`; articulation (`hand`, `attack`)
and per-hand fingering are stamped there, with legato derived from same-string
adjacency. Selection lists tapped triads as `arpeggios` quality candidates and
derives `hands` from the drawn quality — no cross-axis selector machinery. The
phases go data-model + renderer feasibility first, then the pure vocabulary and
placement core (verifiable without a renderer), then selection/rhythm, then
rendering, with an instructor validation gate on the musical shape data.

**Tech Stack:** Python 3.13+, `uv`, `pytest`; the vendored `melete-render`
(Node + `@coderline/alphatab`) for rendering; validation via
`vrg-container-run -- vrg-validate`.

## Global Constraints

Every task's requirements implicitly include these, copied from the spec:

- **Hand count is bounded at two, never more** (spec §1, §11 decision 2).
- **`PLUCKED` and `LEFT` are the defaults; every existing family and golden file
  is unchanged** (spec §2, §4).
- **The central invariant** — `pitch == tuning[string] + fret` for every emitted
  note, and the emitted pitch multiset equals the triad's own `theory.chord_pitches`
  tiled across the register (spec §10). v1 **preserves pitch** (one octave range).
- **In a two-hand shape every note is `TAPPED` or `SLURRED`, never `PLUCKED`**
  (spec §11 decision 12 in v1.0 lineage; §4/§5 here).
- **The hand partition is data in each tap shape, not a derived rule; no global
  fret-ordering invariant holds** (spec §5, §11 decision 6).
- **No layout is clamped to fit; an unrealizable two-hand position raises and §9
  resamples** (spec §9, §6).
- **v1 is `arpeggios` + the four triads (`maj`/`min`/`dim`/`aug`) only**; scales,
  sevenths, and stretched/re-voiced shapes are deferred (spec §2, decisions
  9–11).
- **`hands` is derived from the drawn quality, never sampled independently**
  (spec §7, decision 8). The single-hand `box` path stays byte-for-byte identical
  (spec §6, §10).

## Placement Law

Every task below lands its PR in `mnemosys-project/melete`, so every task issue is
filed there. The epic issue and these `spec.md`/`plan.md` documents live in
`mnemosys-project/.github`.

## Human-Gated Preconditions

| Gate | Why |
|---|---|
| PR submission and merge | Standing policy: agents report ready, humans submit. |
| Tap-shape choreography capture (Task B0) | The shapes are the instructor's taught technique and are not derivable (spec §5); capturing them needs an instructor working session before B1 can encode anything. |
| Tap-shape data confirmation (Task E1) | The tap shapes are musical judgement; the instructor confirms them before generated sheets are trusted, exactly as the one-hand seed shapes are gated on `melete#152`. |

No repository creation or release is required by this epic.

## The REFACTOR Step

- [ ] **REFACTOR (standing step for every implementation task)**
  - Extract duplicated logic — on the second occurrence, not in anticipation.
  - Move hard-coded values to configuration or a shared registry.
  - Consolidate with existing patterns rather than inventing a parallel one.
  - Improve names, then re-run the task's tests to confirm they still pass.

A task is not complete until this step has been performed and its tests are green
afterwards.

---

## Phase A — Data model and renderer feasibility

### Task A1: Renderer feasibility spike (go/no-go)

**Repo:** `mnemosys-project/melete`
**Tracks issue:** `melete#140`
**Blocked-by:** —

A spike, not a TDD task: it produces a short findings note, committed to the epic,
that the emitter task (D1) consumes. It is first because a hard failure re-scopes
the epic (spec §8).

**Files:**

- Create: `docs/reports/alphatex-tapping-effects.md`

**Interfaces:**

- Produces: a table of the exact alphaTex note-effect tokens for **right-hand
  tap**, **left-hand tap**, **hammer-on**, **pull-off**, and **right-hand
  fingering** (or the finding that a token does not exist), plus a verdict:
  `expressible in alphaTex` / `needs lower-level channel` / `no workaround`.

- [ ] **Step 1: Emit probe alphaTex and render it in-container.** Write a
  throwaway alphaTex string exercising a tapped note, a left-hand-tapped note, a
  hammer/pull pair, and a right-hand-fingered note; render with the vendored tool
  and unzip `Content/score.gpif` to confirm the `Tapped` / `LeftHandTapped` /
  `HopoOrigin`/`HopoDestination` and fingering properties, exactly as the R&D
  survey did.
- [ ] **Step 2: Cross-check against the vendored alphaTab source** for the
  effect keywords the parser accepts (the container has the `@coderline/alphatab`
  dist; the host does not).
- [ ] **Step 3: Record findings and verdict** in the report, including the
  **right-hand-fingering** finding, which D1 needs. If the verdict is not
  "expressible", stop and raise it with the human — D1 and the end-to-end
  deliverable are re-scoped per spec §8.
- [ ] **Step 4: Commit** —
  `vrg-commit --type docs --scope reports --message "record alphaTex tapping-effect feasibility (#67)"`

### Task A2: `Hand` and `Attack` on `Note`

**Repo:** `mnemosys-project/melete`
**Tracks issue:** `melete#141`
**Blocked-by:** —

**Files:**

- Modify: `src/melete/score.py`
- Test: `tests/test_score.py`

**Interfaces:**

- Produces: `class Hand(Enum)` (`LEFT`, `RIGHT`); `class Attack(Enum)` (`TAPPED`,
  `PLUCKED`, `SLURRED`); two new `Note` fields `hand: Hand = Hand.LEFT` and
  `attack: Attack = Attack.PLUCKED`, appended after the last existing field. Both
  exported from `melete.score`.

- [ ] **Step 1: Write the failing test** — a default-constructed `Note` is
  `(LEFT, PLUCKED)`; a note can carry `hand=RIGHT, attack=TAPPED`.
- [ ] **Step 2: Run it and confirm it fails** (`ImportError: cannot import name
  'Hand'`).
- [ ] **Step 3: Implement the minimum** — the two enums and two defaulted fields.
- [ ] **Step 4: Run the full suite** (`vrg-container-run -- uv run pytest tests/ -q`)
  — the defaults must leave every family and golden unchanged.
- [ ] **Step 5: REFACTOR**, then commit
  `vrg-commit --type feat --scope score --message "add Hand and Attack to Note (#67)"`

---

## Phase B — The tap-shape vocabulary and two-hand placement (the pure core)

The placement code (B2–B3) is verifiable without a renderer. The vocabulary
(B0→B1) is human knowledge first, code second.

### Task B0: Capture the triad tap-shape choreography from the instructor

**Repo:** `mnemosys-project/melete`
**Kind:** elicitation / data capture (human-collaboration; not TDD) — new sub-issue
**Blocked-by:** —

The tap-shape choreography is **not derivable** (spec §5) — it is the
instructor's taught technique. This task captures it so B1 has a source of truth
to encode. Without it, B1 would encode guesses (most dangerously for `dim`/`aug`).

**Files:**

- Create: `docs/reports/triad-tap-shapes-capture.md` (the captured source table)

**Interfaces:**

- Produces: for each of the four triads (`maj`, `min`, `dim`, `aug`) and each
  inversion (root, first, second), the played choreography on the target
  instrument — per chord tone: which hand, which finger, and its string/fret
  relative to the shape anchor — plus notes on how consecutive shapes leapfrog as
  the journey climbs (feeding spec §6's chaining rule).

- [ ] **Step 1:** Working session with the instructor: record each (triad ×
  inversion) pattern, on the target instrument (the 6-string bass of the survey),
  in one octave range (no stretched/re-voiced shapes — spec §2).
- [ ] **Step 2:** Write the captured table to
  `docs/reports/triad-tap-shapes-capture.md`, marking any shape the instructor
  could not confirm in the session as open.
- [ ] **Step 3:** Commit —
  `vrg-commit --type docs --scope reports --message "capture the triad tap-shape choreography (#67)"`

The formal per-shape confirmation still happens in Task E1; B0 is the elicitation
that makes B1 encodable, E1 is the sign-off on the encoded result.

### Task B1: The triad tap-shape vocabulary (data module)

**Repo:** `mnemosys-project/melete`
**Tracks issue:** reshapes `melete#147` (was "the tapping.reach modifier")
**Blocked-by:** A2, B0

The curated data at the heart of the epic (spec §5). It ships as **PROVISIONAL**
and is confirmed by Task E1.

**Files:**

- Create: `src/melete/families/arpeggio_tap_shapes.py`
- Test: `tests/families/test_arpeggio_tap_shapes.py`

**Interfaces:**

- Produces: `TAP_SHAPES: dict[tuple[str, str], TapShape]` keyed by
  `(quality, inversion)` for `quality ∈ {maj, min, dim, aug}` and
  `inversion ∈ {root, first, second}`. A `TapShape` is an ordered sequence of
  chord-tone placements, each carrying `hand: Hand`, `finger: int` (1–4), and
  `string_offset`/`fret_offset` relative to the shape anchor. A lookup helper
  `tap_shape(quality, inversion) -> TapShape` raises a `ValueError` naming a
  missing entry (spec §9).

- [ ] **Step 1: Write the failing test** — every `(triad × inversion)` has a
  shape; each shape's chord-tone set matches `theory.chord_pitches` for that
  quality/inversion; every placement carries a `Hand` and a 1–4 `finger`; a
  missing lookup raises.
- [ ] **Step 2: Run and confirm failure** (`ModuleNotFoundError`).
- [ ] **Step 3: Implement** the `TapShape` structure, the `TAP_SHAPES` table
  (marked **PROVISIONAL — instructor-gated, Task E1**), and the lookup helper.
  Encode the shapes **from B0's captured table** (`docs/reports/triad-tap-shapes-capture.md`),
  not from a guess; any entry B0 left open stays out until captured, rather than
  fabricated.
- [ ] **Step 4: Run the vocabulary tests.**
- [ ] **Step 5: REFACTOR**, then commit
  `vrg-commit --type feat --scope families --message "add the provisional triad tap-shape vocabulary (#67)"`

### Task B2: Realize `box`'s two-anchor path

**Repo:** `mnemosys-project/melete`
**Tracks issue:** reshapes `melete#143` (was "two_hand_boxed in _shared")
**Blocked-by:** A2

**Files:**

- Modify: `src/melete/families/_shared.py`
- Test: `tests/families/test_shared_two_anchor.py`

**Interfaces:**

- Consumes: `melete.instrument.positions`/`hand_span`; `_shared._reachable`;
  `melete.score.Hand`.
- Produces: `box(profile, pitches, strings, anchors, family, axes)` with
  `len(anchors) == 2` now realized. Given the two anchors (left lower, right
  higher) **and a per-pitch hand assignment supplied by the caller** (the tap
  shape owns the partition — spec §6), it places each hand's tones near that
  hand's anchor, requires each hand's fretted span ≤ `position_span`, requires
  both hands non-empty, preserves pitch, and returns each note's
  `(string, fret, hand)`. Raises `ValueError` (naming pitches, profile, axes)
  when a two-hand box cannot be laid out. **The single-anchor path is byte-for-byte
  unchanged.**

- [ ] **Step 1: Write the failing test** — a two-anchor call with a supplied
  partition returns each hand within `position_span`, both hands non-empty,
  pitch preserved on every note; an unrealizable two-anchor spec raises; **assert
  no global fret ordering** (the partition, not fret order, decides hands).
- [ ] **Step 2: Run and confirm failure** (currently `NotImplementedError`
  naming #67 for `len(anchors) != 1`).
- [ ] **Step 3: Implement** the two-anchor branch; leave the one-anchor branch
  untouched.
- [ ] **Step 4: Run `tests/families/` in full** — the existing `box` tests prove
  the one-hand path is unchanged (the central guard, spec §10).
- [ ] **Step 5: REFACTOR**, then commit
  `vrg-commit --type feat --scope families --message "realize box's two-anchor path (#67)"`

### Task B3: The arpeggios tapped-journey driver

**Repo:** `mnemosys-project/melete`
**Tracks issue:** reshapes `melete#148` (was "wire tapping.reach into pipeline.realize")
**Blocked-by:** B1, B2

**Files:**

- Modify: `src/melete/families/arpeggios.py` (the tapped-journey path)
- Modify: `src/melete/families/_shared.py` (the shared legato pass) and
  `src/melete/pipeline.py` (wire legato after the fitter)
- Test: `tests/families/test_arpeggios_tapping.py`, `tests/test_pipeline.py`

**Interfaces:**

- Consumes: `TAP_SHAPES`/`tap_shape` (B1); `box`'s two-anchor path (B2);
  `theory.chord_pitches`; `melete.score.Hand`, `Attack`.
- Produces: (a) a tapped-journey path in `arpeggios.generate` that, for a triad
  quality, walks the inversions up and back across the neck (spec §6's v1
  chaining rule), realizes each inversion's tap shape via `box`'s two-anchor
  path, stamps `hand`/`finger`, marks every note `TAPPED`, and preserves the
  pitch multiset; and (b) a **shared legato pass** run in `pipeline.realize`
  **after the fitter** (`layout.plan_voice`) and before `rhythm.restamp`, which
  converts each hand's same-string-run followers to `SLURRED` (string/hand change
  forces a fresh `TAPPED`). Legato runs post-fitter so a lever-repeated or
  -dropped note gets correct first-attack-per-run articulation (spec §3, §6). A
  seventh quality continues to use the existing one-hand `shape_places` journey.
  The driver is quality-agnostic — `dim`/`aug` are data lookups, not code
  branches.

- [ ] **Step 1: Write the failing tests** — a triad generate produces a journey
  whose pitch multiset equals `theory.chord_pitches` tiled across the register;
  both hands used; no note `PLUCKED`; the legato pass applied to a tiled voice
  yields `TAPPED`-first-per-run; **a voice a lever repeated/dropped still gets a
  correct first-`TAPPED`-per-run (no stranded slur)**; a seventh generate is
  unchanged.
- [ ] **Step 2: Run and confirm failure.**
- [ ] **Step 3: Implement** the tapped-journey path (all `TAPPED`) and the shared
  post-fitter legato pass; wire the legato pass into `pipeline.realize` after
  `plan_voice`; route by quality class (triad ⇒ tapped, seventh ⇒ one-hand).
- [ ] **Step 4: Run the full family + pipeline suite** — one-hand arpeggios and
  the untapped pipeline unchanged.
- [ ] **Step 5: REFACTOR**, then commit
  `vrg-commit --type feat --scope arpeggios --message "walk the two-hand tapped triad journey with post-fitter legato (#67)"`

---

## Phase C — Selection, configuration, and the rhythm interaction

### Task C1: Tapped triads as quality candidates; derive and record `hands`

**Repo:** `mnemosys-project/melete`
**Tracks issue:** reshapes `melete#142` + `melete#144` (was the `hands` config axis + drawing it)
**Blocked-by:** B3

**Files:**

- Modify: `src/melete/families/arpeggios.py` (or the selection/params seam) and
  `src/melete/config.py` as needed for candidate validation
- Test: `tests/test_selection.py`, `tests/test_config.py`

**Interfaces:**

- Produces: the `arpeggios` pool accepts the four triads as `quality` candidates
  alongside the sevenths; the drawn quality determines `hands` (triad ⇒ `2`,
  seventh ⇒ `1`), which is written into `params` and recorded in the session log
  for coverage and replay (spec §7, decision 8). No independent `hands` axis is
  added and no cross-axis selector machinery is introduced. The default pool
  lists no triads, so existing configurations are untapped and unchanged.

- [ ] **Step 1: Write the failing tests** — an `arpeggios` pool listing a triad
  quality draws that triad and records `hands == 2`; a seventh records
  `hands == 1`; the session log round-trips `hands`; a triad `quality` key under
  a non-arpeggios family is a loud config error (existing axis validation).
- [ ] **Step 2: Run and confirm failure.**
- [ ] **Step 3: Implement** the derivation and recording; wire triad candidates
  through config validation.
- [ ] **Step 4: Run the full selection + config suite** — untapped configs
  unchanged; `hands` replays deterministically.
- [ ] **Step 5: REFACTOR**, then commit
  `vrg-commit --type feat --scope selection --message "route tapped triads and derive the hands value (#67)"`

### Task C2: `restamp` never accents a slurred note

**Repo:** `mnemosys-project/melete`
**Tracks issue:** `melete#145`
**Blocked-by:** A2

**Files:**

- Modify: `src/melete/rhythm.py`
- Test: `tests/test_rhythm.py`

**Interfaces:**

- Consumes: `melete.score.Attack` (A2).
- Produces: `restamp` clears any accent it would otherwise stamp on a `SLURRED`
  note — an accent marks an attack, a slur has none (spec §9).

- [ ] **Step 1: Write the failing test** — a slurred note is never accented, for
  an accent pattern that would otherwise land on it.
- [ ] **Step 2: Run and confirm failure.**
- [ ] **Step 3: Implement** — mask the accent off for any `SLURRED` note before
  the accent is applied.
- [ ] **Step 4: Run the full rhythm suite.**
- [ ] **Step 5: REFACTOR**, then commit
  `vrg-commit --type feat --scope rhythm --message "never accent a slurred note (#67)"`

---

## Phase D — Rendering and integration

### Task D1: Emit `hand`/`attack` and per-hand fingering as alphaTex effects

**Repo:** `mnemosys-project/melete`
**Tracks issue:** `melete#146`
**Blocked-by:** A1 (tokens), A2 (fields)

**Files:**

- Modify: `src/melete/alphatab/emit.py`
- Test: `tests/alphatab/test_emit_tapping.py`

**Interfaces:**

- Consumes: the confirmed token table from A1; `melete.score.Hand`, `Attack`.
- Produces: `_note_token` appends, into its `effects` list, the tap effect for a
  `TAPPED` note (right- vs left-hand per `note.hand`), the hammer/pull effect for
  a `SLURRED` note, and **right-hand fingering** for a right-hand note (in place
  of `lf`, which is left-hand only). Token strings are module constants set from
  A1's findings. If A1's verdict was "needs lower-level channel," implement that
  channel here instead — the deliverable is the articulation and fingering
  reaching the `.gp`.

- [ ] **Step 1: Write the failing test** — a right-hand tapped note carries the
  RH-tap token and a right-hand-fingering token; a **left-hand tapped note
  carries the LH-tap token and keeps `lf` fingering**; a slur carries the
  hammer/pull token; a default `(LEFT, PLUCKED)` note emits exactly as before.
- [ ] **Step 2: Run and confirm failure.**
- [ ] **Step 3: Implement** the effect appends and the per-hand fingering branch.
- [ ] **Step 4: Run the full emit suite** — the golden alphaTex tests guard the
  default path.
- [ ] **Step 5: REFACTOR**, then commit
  `vrg-commit --type feat --scope alphatab --message "emit hand/attack and per-hand fingering (#67)"`

### Task D2: End-to-end tapped-sheet integration test

**Repo:** `mnemosys-project/melete`
**Tracks issue:** `melete#149`
**Blocked-by:** B3, C1, C2, D1

**Files:**

- Test: `tests/test_tapping_end_to_end.py`

**Interfaces:**

- Produces: a black-box test that an `arpeggios` config listing a tapped triad
  generates and renders to a valid `.gp` whose `Content/score.gpif` carries the
  `Tapped` property — the spec's success criterion, checked the way the R&D
  survey detected tapping.

- [ ] **Step 1: Write the test** following the existing real-alphaTex-to-`.gp`
  integration test; configure `[pool.arpeggios] qualities` with a triad so every
  draw taps.
- [ ] **Step 2: Run and confirm** it is red until the chain lands.
- [ ] **Step 3: Make it pass** — no new production code beyond Phases B–D should
  be needed.
- [ ] **Step 4: Run the whole suite** — `vrg-container-run -- vrg-validate`.
- [ ] **Step 5: REFACTOR**, then commit
  `vrg-commit --type test --scope tapping --message "end-to-end tapped .gp integration test (#67)"`

---

## Phase E — Instructor validation of the tap-shape data

### Task E1: Confirm the triad tap-shape vocabulary

**Repo:** `mnemosys-project/melete`
**Kind:** `validation` (run via `issue-validate`) — new sub-issue
**Blocked-by:** B1 (data authored); may run in parallel with C–D

A live check, not a code change (like `melete#152` for the one-hand seed shapes).
The instructor confirms each `(triad × inversion)` tap shape against how the
material is actually played; confirmed shapes drop the `# PROVISIONAL` marker.

- [ ] **Step 1:** Render a reference sheet per triad × inversion from `TAP_SHAPES`.
- [ ] **Step 2:** Review each with the instructor; record confirmations and
  corrections in the epic.
- [ ] **Step 3:** Apply confirmed corrections to `arpeggio_tap_shapes.py` (a
  same-repo PR) and remove the provisional markers for confirmed entries.
- [ ] **Step 4:** Record the validation outcome as an issue comment.

---

## Task Summary

| # | Task | Repo | Kind | Blocked-by | Tracks issue |
|---|---|---|---|---|---|
| A1 | Renderer feasibility spike (go/no-go) | `melete` | spike | — | `#140` |
| A2 | `Hand`/`Attack` on `Note` | `melete` | code | — | `#141` |
| B0 | Capture triad tap-shape choreography | `melete` | capture | — | new |
| B1 | Triad tap-shape vocabulary (data) | `melete` | code | A2, B0 | `#147` (reshaped) |
| B2 | Realize `box`'s two-anchor path | `melete` | code | A2 | `#143` (reshaped) |
| B3 | Arpeggios tapped-journey driver + post-fitter legato | `melete` | code | B1, B2 | `#148` (reshaped) |
| C1 | Tapped triads as candidates; derive `hands` | `melete` | code | B3 | `#142` + `#144` (merged) |
| C2 | `restamp` skips accents on slurs | `melete` | code | A2 | `#145` |
| D1 | Emit hand/attack + per-hand fingering | `melete` | code | A1, A2 | `#146` |
| D2 | End-to-end tapped-sheet test | `melete` | code | B3, C1, C2, D1 | `#149` |
| E1 | Instructor validation of tap shapes | `melete` | validation | B1 | new |

**Genuinely parallel:** A1, A2, and B0 have no code blockers (B0 gates on
instructor availability, not code). B2 needs only A2; B1 needs A2 + B0's captured
data; B3 joins B1 and B2. C2 needs only A2. D1 needs A1+A2. E1 signs off the
encoded shapes once B1 lands and runs alongside C–D. D2 is the join of
everything. B0 and E1 both depend on instructor time and are the epic's two human
gates beyond PR submission.

## Sub-issue reconciliation (for the human at the gate)

The rebase changes the task shape, so the epic's sub-issues need reconciling.
Proposed — to be actioned on approval, not before:

- **Keep as-is:** `#140` (A1), `#141` (A2), `#145` (C2), `#146` (D1), `#149` (D2).
- **Reshape (retitle + rebody):** `#143` → B2 (box two-anchor path, not a new
  sibling); `#147` → B1 (tap-shape data, not `tapping.reach`); `#148` → B3 (the
  arpeggios driver, not pipeline wiring); `#142` → C1 (candidates + derived
  `hands`).
- **Close as merged into C1:** `#144` (drawing the `hands` axis) — its work is
  absorbed by C1; there is no independent `hands` axis to draw.
- **Create:** `B0` — the tap-shape choreography capture task; `E1` — the
  tap-shape instructor-validation task.
- **Bookends unchanged:** `#133` (docs review), `#69` (retrospective).

## Spec Coverage

| Spec section | Task |
|---|---|
| §1 Overview | B1, B3 (the vocabulary and its journey) |
| §2 Scope (in-scope deliverables) | A2, B1, B2, B3, C1, C2, D1, E1 |
| §3 Architecture (placement at generation) | B3 |
| §4 Data model (`hand`, `attack`, per-hand `finger`) | A2; fingering emitted in D1 |
| §5 The tap-shape vocabulary | B0 (capture), B1 (data), E1 (validation) |
| §6 Placement (two-hand box, tapped journey, chaining rule, post-fitter legato) | B2, B3 |
| §7 Selection and configuration | C1 |
| §8 Rendering / spike | A1, D1 |
| §9 Error Handling | B2 (unboxable raises), B1 (missing shape), C1 (derived `hands`), C2 (slur accents), A1/D1 (renderer) |
| §10 Testing Strategy | every task's tests; central invariant in B3 |
| §11 Recorded Decisions | enforced across tasks via Global Constraints |
| §12 Deferred | not implemented by design (multi-octave climb raises in B2/B3) |

## Evolution during execution

**Required. Append as the epic runs, not at the end.** One entry per deviation —
what changed and, above all, *why*.

<!-- (no entries yet — implementation has not started; this plan is the v2.0
rebase of the pre-#72 plan) -->
