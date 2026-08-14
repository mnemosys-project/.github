# Retrospective — Even-measure exercise layout

**Epic:** [`mnemosys-project/.github#57`](https://github.com/mnemosys-project/.github/issues/57).
**Partners:** `spec.md` (what we set out to do) and `plan.md` (how we planned it);
this is the third of the trio — honestly how it went.
**Span:** opened 2026-08-13, closed 2026-08-14.

## 0. At a glance

We set out to fix the *engraving* of generated exercises: the first practical
Guitar Pro output (the Aug 12 practice session) was musically fine but every
exercise ended in a **partial measure** that Guitar Pro paints with red staff
lines, because the meter and subdivision were drawn at random from config pools.
What shipped: a **layout fitter** that derives a legible engraving from the
pattern — every exercise now tiles into **whole, complete measures, wrapped in
repeat barlines, in a simple low-denominator meter** — validated in Guitar Pro
against the regenerated day. The random *sampling* of meter and subdivision was
replaced by *derivation*; the combined practice book now places one exercise per
system so section titles no longer overprint.

### Work delivered

| PR | Task | What it did |
|---|---|---|
| [`.github#60`](https://github.com/mnemosys-project/.github/pull/60) | #58 | Published the spec + plan |
| [`melete#123`](https://github.com/mnemosys-project/melete/pull/123) | #111 | `LayoutHints` contract + cell-aware apex-lever realization |
| [`melete#124`](https://github.com/mnemosys-project/melete/pull/124) | #112 | Repeat barlines (`Score.repeat` + alphaTex emitter) |
| [`melete#125`](https://github.com/mnemosys-project/melete/pull/125) | #120 | `MEMORY.md` + the `build/` development output discipline |
| [`melete#126`](https://github.com/mnemosys-project/melete/pull/126) | #113 | Fitter tiling + the meter priority ladder |
| [`melete#127`](https://github.com/mnemosys-project/melete/pull/127) | #115 | Every family returns `(Score, LayoutHints)` |
| [`melete#128`](https://github.com/mnemosys-project/melete/pull/128) | #114 | Note-count lever search + odd-but-sane fallback |
| [`melete#129`](https://github.com/mnemosys-project/melete/pull/129) | #116 | Chromatic all-strings cycle + apex lever + hints |
| [`melete#130`](https://github.com/mnemosys-project/melete/pull/130) | #117 | Scales two-octave-from-low + `up_down` default + positional fallback |
| [`melete#134`](https://github.com/mnemosys-project/melete/pull/134) | #132 | **Cell-granular levers + cell-aligned turnaround** (the mid-epic fix) |
| [`melete#135`](https://github.com/mnemosys-project/melete/pull/135) | #118 | Pipeline: the fitter drives meter/subdivision (`generate → fit → restamp`) |
| [`melete#136`](https://github.com/mnemosys-project/melete/pull/136) | #119 | Retire the sampled meter/subdivision config axes |
| [`melete#137`](https://github.com/mnemosys-project/melete/pull/137) | #121 | Freeze the five 2026-08-13 exercises as acceptance goldens |
| [`melete#139`](https://github.com/mnemosys-project/melete/pull/139) | #138 | One system per exercise in the book (fix title overprint) |
| [`melete#150`](https://github.com/mnemosys-project/melete/pull/150) | #110 | `docs/design.md` reflects the fitter/pipeline |

- **Repos touched:** 2 — `melete` (code + docs), `.github` (spec/plan/retrospective).
- **Tasks:** 17 children — 13 implementation, 1 documentation, 1 acceptance golden, 1 validation (no PR), + this retrospective.
- **PRs merged:** 15 (14 in `melete`, 1 in `.github`), plus one unplanned model-fix PR (#134) and one unplanned book-layout PR (#139).
- **Operational tasks:** 1 validation (#122) — recorded SUCCESS in Guitar Pro, no code PR.
- **Releases cut:** none — `melete` is a generator, not a released artifact.

## 1. How the plan evolved

The plan was an 11-task TDD breakdown; execution ran to 14 code PRs plus two the
plan did not foresee. Three deviations mattered.

**The integration task exposed a model defect the plan had baked in.** The plan
assumed the layout fitter, the per-family `LayoutHints`, and the note-count
levers (built in #113/#114/#116/#117) were correct, and that #118 merely wired
them into the pipeline. When #118 actually routed draw-validation through the
fitter, it surfaced that **whole classes of `(family, direction)` could not tile
into whole bars at all** — the windowed families (scales, arpeggios) turned
around at the *note* level (`2L−1` notes ≡ `cell−1 mod cell`, never a whole
number of cells for `cell>1`), and the `ADD_ONE`/`DROP_ONE` levers moved by a
single *note*, which broke cell-divisibility. The implementing agent stopped and
reported rather than masking the skew, and the fix became a new blocking task
(**#132**): make every lever **cell-granular** — so each moves the beat count by
exactly one, and any odd `B` reaches an even one — and make the windowed families turn
around at the **cell** level (as chromatic already did over its string walk).
That single model change restored the §14 draw uniformity and unblocked #118,
which then went green on rebase with no further integration edits.

**Two plan examples did not survive contact with arithmetic.** The plan's Task 3/4
lever tests used a `B=14` "raises" case that actually tiles as `2/4 × 7`
(`b=2` divides 14), and specified "all strings" as an `"all"` sentinel on the
integer `span` axis. Both were corrected during execution — the `B=14` cases were
replaced with genuinely un-tileable counts, and "all strings" was implemented as
`span` = the profile's string count (no sentinel, no vocabulary surgery).

**A book-layout bug surfaced only under a real viewer.** After everything
validated green, opening the *combined* practice book in Guitar Pro showed
section titles overprinting — the book packed 3 bars per system by default so
consecutive 2-bar exercises shared a line. Reverse-engineered from a hand-fixed
file (the same edit-and-diff trick that cracked the repeat tokens), the fix (#138)
emits a track-level `systemsLayout` in alphaTex so each exercise gets its own
system. Pure alphaTex; `melete-render` stayed untouched.

Pushback and alignment also moved a design decision before any code: apex-repeat
became a **fitter lever** the family declares rather than a cycle the family bakes
in — which is why the 44-note chromatic base cycle reaches 48 through the fitter,
and why the lever machinery is exercised on the exact case it exists for.

## 2. Lessons learned

- **Front-loading the fitter design paid off precisely because it let the defect
  surface at integration, not in the field.** The cell-alignment flaw would
  otherwise have shipped as red measures scattered across most scale/arpeggio
  exercises; instead it appeared as a measurable draw-distribution skew at one
  gate and was fixed as one clean model change. Design-heavy epics earn this.
- **Cell-alignment is the load-bearing invariant.** Everything tiles because the
  note count is always a whole number of cells and every lever moves it by whole
  cells. Chromatic got this right by turning around at the string (= cell) level
  from the start; the windowed families had to be brought to the same footing.
- **Engraving aesthetics need real-output review, not spec reasoning.** The
  ladder's "prefer fewer bars / larger beats-per-bar" tie-break, chosen during
  alignment for compactness, turns out to read *worse* than four bars in many
  cases (owner feedback on the rendered book). A spec argument is not a
  substitute for looking at the page.
- **Agents that stop and report beat agents that force green.** #118's refusal to
  mask the skew is what turned a latent model defect into a tracked, fixed one.
- **Deterministic byte-reproduction is a cheap, strong acceptance test.** The day
  regenerates identically from date+config, so the goldens catch any drift.

## 3. Compromises & tradeoffs

- **Section-title horizontal stagger — accepted, not fought.** Titles align to
  each measure's start, but measures begin at different x-positions because the
  exercises are engraved in their **natural keys** (key signatures differ in
  width). This is a true consequence of the natural-key decision, not a bug; the
  only fixes (force a common key, or embed a full Guitar Pro stylesheet) cost
  more than the annoyance. Left as-is.
- **The fitter is a tunable heuristic engine, by design.** The ladder tie-breaks,
  the sane-beats whitelist, and the uniform-pulse fallback for unequal positional
  groups are explicit, extensible surfaces expected to accrete corner cases — not
  a closed formula.
- **Note-count levers act on whole cells, generalizing the spec's "add/drop one
  note."** For `cell>1` families a lever adds or drops a whole group (a string's
  four notes, a pair); this is what tiling requires and is a deliberate
  generalization of the spec wording.
- **`melete-render` remains extraction-bound tech debt** (`melete#82`),
  untouched — every renderer fix in this epic (repeats, book layout) was made in
  pure alphaTex, keeping the Node tool dumb.

## 4. New problems & opportunities

- **Note-content layout quality (the large follow-on).** Owner review of the
  rendered book found that, *within* the now-correct even measures, the way notes
  are chosen and placed needs a rethink: scales are not reliably spanning **two
  octaves**; many exercises would read better as **four measures, not two** (the
  fewer-bars tie-break above); and the **starting position and pattern shape are
  inconsistent** across exercises and need a principled, pedagogically-sound
  placement model. → Logged as the seed for the next epic (see §5); not yet acted
  on.
- **The fewer-bars tie-break likely needs to flip or become context-aware** — a
  concrete, isolable piece of the above.
- **Cell-alignment corner cases will keep appearing** as the heuristic engine
  meets new families/axes; the legibility trace and the fit-sweep test are the
  tools for catching them.
- **Book layout for long exercises** — one-system-per-exercise is right for 2-bar
  exercises but would make a long exercise one very wide system; a future
  refinement could split long exercises while still starting each fresh.

## 5. What's next

The forward axis is a **follow-on epic on note-content layout** — reliable
two-octave spans, the two-versus-four-measure (and tie-break) rethink, and a
consistent start-position/shape model — brainstormed via `epic-create` when the
owner is ready. It builds directly on this epic's deliverable: the even-measure
engraving framework is the stable base those note-placement decisions now sit
inside. This epic is deliberately **called a success on its own scope** (the
engraving problem) with the note-content problem cleanly separated forward, rather
than blurred into it.
