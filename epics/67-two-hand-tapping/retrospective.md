# Two-handed tapping — retrospective

**Epic:** [`mnemosys-project/.github#67`](https://github.com/mnemosys-project/.github/issues/67)
**Partners:** [`spec.md`](./spec.md) (v2.1) · [`plan.md`](./plan.md)
**Opened:** 2026-08-13 · **Closed:** 2026-08-18 · **Span:** ~5 days
**Repos touched:** `mnemosys-project/.github` (planning docs), `mnemosys-project/melete` (code + docs)

## §0 At a glance

We set out to add **two-handed tapped triad arpeggios** to melete as a small,
tabulated v1. What shipped is much larger and better: a **rule-derived two-hand
tapping system** spanning **triads, seventh chords, and 3-notes-per-string
scales** — all config-driven, generated end-to-end to a real Guitar Pro `.gp`,
with fingering and articulation faithful to how the player actually plays. The
turn came from a single decision to stop tabulating and start **deriving from a
growing corpus of the player's own example files**; that corpus (rules **R1–R14**,
`docs/reports/tapping-fingering-rules.md`) went from *recording* the technique to
*predicting* it — the seventh fingering fell out of the existing rules with no new
data at all.

**Delta from plan: very large — and that is the story.** The plan (v2.1) scoped
*triads only, one tabulated box*. The epic instead produced a derivation and two
whole extra families. This retrospective is mostly about *why* that delta happened
and what it teaches.

### Work delivered

**29 child tasks, ~23 PRs, 2 repos.** Grouped by phase:

| Phase | PRs | What |
|---|---|---|
| **Planning** (`.github`) | #70, #80, #82 | spec+plan v1.0 → **conceptual rebase** onto #72 (v2.0) → **revise to the universal box** (v2.1) |
| **v1 build** (`melete`) | #190 A1, #191 A2, #195 B0, #196 B1, #192 B2, #197 B3, #198 C1, #194 D1, #199 D2 | renderer spike; `Note.hand`/`attack`; capture + encode the tap box; two-anchor placement; tapped journey + post-fitter legato; selection; emit; end-to-end tapped `.gp` |
| **Faithfulness** | #203 F1, #204 F2, #206 F3 | symmetric descent (R7); quality-aware root finger (R2); apex-doubled journey + turnaround re-tap (R12) |
| **Rules corpus** | #207, #221 | R1–R12; then R13–R14 |
| **Sevenths** | #209 G1, #211 G2, #213 G3 | seventh two-string grid (R11) + journey + `tapped_qualities` selection |
| **Scales** | #215 H1, #217 H2 | tapped 3nps scale (per-string, cross-hand cascade) + `tapped_scale_types` selection |
| **Fingering solve** | #222 I1, #223 I2 | octave-dependent third (R4); descending re-fingering (R5) |
| **Close-out** | #224, E1 (#189) | docs sweep; validation (player sign-off) |

**Releases cut:** none (feature is default-off; no version bump was in scope).
**Validation:** the suite grew from ~2150 to **2472 tests**, 100% coverage held throughout.

## §1 How the plan evolved

The plan was written once and then *out-evolved by reality* three times, each a
sharper cut than the last.

**1. The conceptual rebase (v1.0 → v2.0).** Before implementation began, epic #72
(the coherent up-and-down journey) landed and rewrote the exact seams tapping
plugged into: `_shared.boxed` became `box(..., anchors)` with the two-anchor case
*already reserved for #67*; the `direction`/`string_set`/`range_octaves` axes were
retired; families now owned placement. Rather than patch a stale plan, we re-ran
the whole planning loop — brainstorm → pushback → alignment — against the new
code. Pushback and alignment each caught real defects the rebase introduced (the
"constrain the quality universe" selection claim wasn't expressible in the
independent sampler; the invariant test referenced a one-hand triad journey that
can't exist; a capture task for the shape data was missing). *Lesson embedded:
when the ground moves under a plan, re-derive the plan, don't edit it.*

**2. The box collapse (v2.0 → v2.1).** v2.0 still assumed the choreography was
**12 hand-authored `(quality × inversion)` shapes**. The player's first example
`.gp` refuted that in one read: the four triads share **one universal box** whose
frets *derive* from the chord intervals. B1 shrank from "author 12 shapes" to
"encode one box + derive." The spec was revised to v2.1 to match.

**3. The corpus turn (unplanned, and the biggest delta).** The plan ended at
triads. But each example the player produced taught a rule, and the rules started
to *compose*: the symmetric-descent bug (R7) exposed that the fitter's `DROP_ONE`
lever stripped the closing root, which the apex-doubled re-tap (R12) fixed *and*
matched how the player actually plays; the seventh grid (R11) needed **zero** new
fingering data because R2/R3 predicted it (R13); scales brought the legato
machinery — built dormant for arpeggios — to life, and surfaced that some
articulation is context-dependent, not local (R14). None of this was in the plan;
all of it was captured in the living rules doc (`melete#200`) as it happened.

## §2 Lessons learned

- **Derive beats tabulate — once you have enough instances to see the rule.** The
  single most valuable move was refusing to hand-author shapes and instead
  building a corpus + rules. It made sevenths and scales cheap and made the
  fingering *predictable*. But note the ordering: tabulation (the box) came first
  and *accumulated the instances* the derivation needed. You cannot derive a rule
  you have not yet seen enough of.
- **Test at the layer the claim lives.** Twice a fix was verified at the family
  layer and silently undone downstream: F1's symmetric descent was correct in the
  family but the layout fitter dropped the closing note; only the E1 *reference
  sheet* (real `.gp`) caught it. The durable fix was to assert **post-fitter** via
  `pipeline.realize`. Generating the real artifact is a load-bearing check, not a
  nicety.
- **A domain expert in the loop changes the economics.** The player authored
  precise `.gp` examples and confirmed every derivation. That tight loop is what
  let us turn *implicit* playing decisions (the turnaround re-tap; the descending
  cross-hand cascade) into deterministic rules rather than guesses.
- **Re-derive plans on a moving base.** The #72 rebase would have produced subtly
  wrong code if we had patched the v1.0 plan; the full re-run paid for itself.

## §3 Compromises & tradeoffs

- **Fingering is a *solve* we only partially built.** R8 (rolling-window
  re-handing / "the fold") is real but applies only to the groups-of-three
  pattern, which melete does not generate — so it is deferred, honestly, rather
  than half-implemented. R4/R5 (octave/direction-dependent fingering) *were* built
  for the shapes that ship.
- **The descending scale cascade couples two modules.** The cross-hand pull-off is
  *locally indistinguishable* from an arpeggio's hand-leapfrog re-tap, so the
  scale family **stamps** it and `derive_legato` **preserves** it — a deliberate
  coupling the local legato rule could not resolve alone (R14). Correct, but worth
  a reviewer's eye.
- **`# PROVISIONAL` markers remain in `arpeggio_tap_shapes.py`.** E1 passed, so
  they can come off; that trivial cleanup was logged (E1 close comment) rather than
  done, to not gate the retrospective.
- **Validation is the player, not a separate instructor.** E1 was framed as
  "instructor validation"; in practice the player (whose corpus this is) is the
  authoritative reviewer. Recorded as passed on that basis.
- **No `Co-Authored-By` trailer on feature commits.** `vrg-commit` composes its own
  body and offers no trailer flag; amend is denied. A tooling gap, noted on the PRs.

## §4 New problems & opportunities

- **The fingering solve as a general algorithm.** R8/R14 point at the real prize
  the player named: given a passage, *solve* for the two-hand fingering by walking
  it with context — a small state machine, not a lookup. The corpus (R1–R14) is
  the constraint set that solve would consume. *Logged in the rules doc's open
  questions; not yet a task.*
- **Symmetry predicts alternate fingerings (R6).** Augmented's uniform-diagonal
  admits a second fingering; the open hypothesis is that diminished-seventh and
  whole-tone structures do too. *Logged.*
- **More patterns & voicings deferred, named:** the rolling groups-of-three
  pattern, the alternate four-string seventh voicing, positional (non-3nps) tapped
  scales, octave-displaced "stretched" voicings, and the pre-fretted-cascade
  **technique tag** (R10) — a phrase-level annotation that renders identically to
  tap+pull but records the player's intent. All named in spec §12 / the rules doc.
- **Publishing the derivation.** The player's stated long-game: once the corpus is
  rich enough, the mathematical structure underneath these rules may be a
  publishable contribution. The rules doc is the seed.
- **Tooling gap** — `vrg-commit` cannot carry a `Co-Authored-By` trailer. Worth
  filing against the tooling.

## §5 What's next

No follow-on epic is minted yet. The forward-looking work lives, referenced, in:

- `docs/reports/tapping-fingering-rules.md` — **§ Open questions** (the fingering
  *solve*; symmetry→alternates; per-root confirmation) and the changelog, which is
  the live intake for the next corpus additions.
- **spec §12 (Deferred)** — the named non-goals (scales-beyond-3nps, stretched
  voicings, the demand-driven exercise-specification mechanism that would supersede
  the config toggles, external corpora such as Berthoud).

The natural next step, when the player wants it, is a **forward brainstorm on the
fingering solve** — turning the corpus's constraints into a passage → hand-fitting
algorithm. That is where "record the technique" becomes "generate any tapped
passage."

---

*The delta between this and the plan is enormous, and it is the good kind: the plan
was honest about a small v1, and the work, guided by a domain expert and a growing
corpus, discovered that the technique it was capturing had a derivable structure
underneath. We stopped tabulating and started deriving — and the rules began to
predict.*
