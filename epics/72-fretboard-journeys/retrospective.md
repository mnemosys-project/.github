# Retrospective — Coherent fretboard journeys and the layout substrate

**Epic:** [`mnemosys-project/.github#72`](https://github.com/mnemosys-project/.github/issues/72)
**Partners:** [`spec.md`](./spec.md) → [`plan.md`](./plan.md) → this record
**Opened:** 2026-08-14 · **Closed:** 2026-08-15

## §0 At a glance

We set out to replace melete's incoherent, independently-sampled exercise
*geometry* with **computed coherent journeys** — and to extract the one-hand half
of a placement substrate that two-handed tapping (`#67`) will rebase onto. We
shipped exactly that, plus a body of engraving work the redesign *exposed*: once
the exercises became long full-neck up-and-down journeys, the rigid one-system
layout and the odd 2/4 meter choice became visibly wrong, and we folded the fix
into the same epic as fair-game fallout.

**What shipped:** every family (chromatic, scales, arpeggios, intervals) now emits
a single up-and-down journey anchored at the root on the lowest string,
travelling outer-string to opposite-outer-string with octaves emergent; a
generalized `box` primitive and a `journey.py` builder compute placement;
fingering styles are first-class (scales boxed / three-note-per-string; arpeggios
by seed shape); the `direction` / `string_set` / `range_octaves` axes (and
arpeggio `traversal`) are gone; the fitter no longer picks 2/4; and each exercise
wraps into even ~4-bar systems. The five acceptance exercises were regenerated,
signed off, and re-frozen.

### Work delivered

| PR | Task | What it did |
|---|---|---|
| melete#163 | A1 | generalized `box` placement primitive (N=1; N≥2 is the #67 seam) |
| melete#164 | C1 | arpeggio seed shapes + derivation |
| melete#165 | D1 | chromatic: outer-to-outer up-and-down traversal |
| melete#166 | — | quarantine the byte-for-byte acceptance goldens (`xfail`) |
| melete#167 | E1 | anchor the root on the lowest instrument string |
| melete#168 | B1 | the one-hand journey builder (`journey.py`) |
| melete#169 | B2 | scales compute the outer-to-outer journey |
| melete#170 | D2 | intervals compute the journey (via `boxed_span`) |
| melete#171 | C2 | arpeggios compute the seed-shape journey |
| melete#172 | E2 | remove the geometry axes; migrate the config; un-relax the meta-test |
| melete#176 | (layout) | demote 2/4 to a strict last resort |
| melete#177 | (layout) | wrap each exercise into ~4-bar systems |
| melete#179 | (layout) | make 2/4 a *true* last resort (a `_quality` penalty) |
| melete#180 | (fix) | `boxed_span` names the unreachable anchor |
| melete#182 | (layout) | distribute systems evenly — never a lonely 1-bar line |
| melete#184 | F1 | re-freeze the acceptance goldens; lift the quarantine |
| melete#185 | doc-review | sweep melete's docs for the journey + layout model |
| melete#187 | (fix) | sweep removed-axis names from user-visible strings |
| .github#75 | docs | publish `spec.md` + `plan.md` |
| .github#77 | docs | the plan's Evolution log + config-path correction |

Plus the operational **F1 validation** (`melete#152`, closed by SUCCESS comment —
human sign-off that the regenerated exercises are correct and playable).

- **Repos touched:** `mnemosys-project/melete` (code + tests + docs),
  `mnemosys-project/.github` (spec / plan / evolution / this retrospective).
- **Children:** 22 (21 closed + this retrospective). ~18 code/docs PRs + 1
  validation task.
- **Releases cut:** none (no release was required).
- **Span:** opened 2026-08-14, closed 2026-08-15 — roughly a day and a half.

## §1 How the plan evolved

The plan's task graph held up well — the family rewrites, the substrate, and the
axis removal landed close to as written. The interesting deltas were all *emergent*:

- **The acceptance goldens were a moving target the plan under-modelled.** Every
  output-changing task drifts the byte-for-byte goldens, so mid-flight we
  introduced a **quarantine** (`xfail` the snapshots up front, re-freeze once at
  the end) rather than re-freezing per task, which would have bypassed the
  instructor sign-off. The quarantine was itself under-scoped — it missed the
  render-layout golden, which drifts the moment any exercise's bar count changes —
  and had to be extended once that surfaced.
- **Trimming family `AXES` before removing the config axes broke a shared
  invariant.** The pool-vs-family meta-test failed the instant a family dropped an
  axis its pool still sampled. We bridged it with a *transitional relaxation*
  (family axes ⊆ pool, surplus must be a retiring axis) in the first family PR and
  **un-relaxed it back to strict equality** in the consolidated removal (E2). This
  self-healing pattern kept `develop` green across the parallel family rewrites.
- **Some failures only existed when parallel work combined.** With all three
  family rewrites merged, axes read by *no* family orphaned two `_shared` helpers
  and shifted a CLI vocabulary test — a "each PR green alone, red together" class
  the last-to-merge PR had to reconcile.
- **The single largest delta was unplanned engraving work.** The content fix
  *revealed* that the one-system-per-exercise layout (a prior fix for short
  exercises) and the meter selection were wrong for long journeys. Rather than a
  separate epic, four small tasks were folded in — skip 2/4, 2/4 as a true last
  resort, wrap into ~4-bar systems, distribute systems evenly — extending the
  spec's renderer-boundary scope by explicit agreement.

## §2 Lessons learned

- **Fixing content exposes the problems the broken behaviour was hiding.** The
  cramming and 2/4 were invisible while exercises ran one direction and stopped
  short; correct, full-neck journeys made them obvious. Budget for downstream
  fallout when a redesign lands — it is a sign of success, not scope creep.
- **A redesign that moves snapshot output needs an explicit quarantine strategy in
  the plan, not invented mid-flight.** Deciding up front *which* goldens drift, how
  they're suppressed, and when they're re-frozen (once, with sign-off) would have
  saved two reconcile passes. Snapshot tests are a moving target during a refactor
  by nature.
- **Parallel tasks that each trim a shared invariant produce combine-time
  failures.** A transitional relaxation that self-heals at the consolidation step
  is a clean way to keep the trunk green; the alternative (each PR editing the
  shared guard) causes churn and conflicts.
- **Adversarial grounding beats plan literals.** The plan's hand-computed test
  pitches were wrong (a low-string `Bb` is fret 11 = pitch 34, not 46); the
  implementing agents caught it by computing from the real tuning. Structural
  assertions (`tuning[string] + fret == pitch`) are more robust than frozen
  numbers.
- **Designing epic N with epic N+1's requirements in hand pays off.** #72 was
  shaped by reading #67's (tapping) plan first, so the `box` primitive is N-anchor
  from day one and coverage is a strategy property — the seam tapping needs is
  already there, and #67 rebases onto it instead of colliding.

## §3 Compromises & tradeoffs

- **Arpeggio seed shapes are provisional.** v1 ships one canonical one-tone-per-
  string diagonal shape per quality, instructor-validated. It spans the neck wide
  (a swept A7 lands ~frets 11–19), and some low roots genuinely raise (the 5th/7th
  fall off the neck) and are resampled. A tighter boxed fingering is a **data-only**
  change to `SEED_SHAPES`, deliberately deferred pending the instructor's call.
- **The renderer-boundary scope was widened.** The spec explicitly excluded the
  emitter/fitter; the engraving fallout pulled `layout.py` and `emit.py` in. A
  justified expansion, but it means the shipped epic is broader than its spec —
  recorded here and in the plan's Evolution log rather than pretended away.
- **Chromatic still starts mid-neck.** `start_string` is still sampled, so a
  skip-string chromatic can begin on an inner string rather than an outer one. Made
  *coherent* (up-and-down, full traversal) but not *anchored* — a small content
  follow-up (see §4).
- **#67 supersession is reconciled by hand.** The tapping epic's prior plan is
  superseded by this substrate; rather than mint tracker-blocking links, the author
  owns re-brainstorming #67 on top of #72. Correct for a single-owner project;
  worth noting as an untracked dependency.
- **One trivial dead constant** (`STRING_SET` in `selection.py`) was left behind —
  logged, not swept, to avoid an extra PR at the finish line.

## §4 New problems & opportunities

- **Engraving heuristics became real work** — what looked like a follow-on epic
  collapsed into four in-epic tasks (2/4, systems). Logged and *done*.
- **Chromatic complementary-descent** — when a skip-string exercise ascends on
  strings 1-3-5, it could descend on 6-4-2 to cover the neck fully. Raised, judged
  minor, **not yet filed** — a content follow-up to the chromatic family.
- **Arpeggio seed-shape refinement** — pending the instructor's review of the
  provisional shapes; a `SEED_SHAPES` data change if he prefers boxed fingerings.
- **Dead `STRING_SET` constant** in `selection.py` — trivial cleanup, logged.
- **The two-hand placement substrate now exists.** `box` accepts N anchors and
  coverage is a strategy property, so the seam for tapping is in place and
  exercised only at N=1 today.

## §5 What's next

- **Two-handed tapping (`#67`) — re-brainstormed and rebased on this substrate.**
  This was the explicit enabling chain: #72 built the one-hand half of the
  placement engine *so that* #67 becomes the two-hand strategy on top of it,
  instead of the colliding parallel mechanism its original plan assumed. The author
  owns kicking that off as its own `epic-create` run.
- **Deferred (named in the spec §13), unchanged:** single-string and two-string
  modes; horizontal / diagonal continuations; upper-neck root anchoring; the
  hand-written bespoke exercise library.
- **Small follow-ups accrued here:** chromatic complementary-descent; arpeggio
  seed-shape refinement with the instructor; the `STRING_SET` dead-constant sweep.

---

**Status:** Terminal. Merging this retrospective's docs PR closes epic
[`#72`](https://github.com/mnemosys-project/.github/issues/72).
