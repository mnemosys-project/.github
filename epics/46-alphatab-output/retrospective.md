# Retrospective — alphaTab/`.gp` output pipeline (port from LilyPond)

> Reads with its siblings: **[spec](spec.md) → [plan](plan.md) → retrospective**.
> What we set out to do, how we planned it, and honestly how it went.

## §0 At a glance

We set out to replace melete's LilyPond/PDF output with a pipeline that emits
**alphaTex** text and runs **alphaTab** to produce Guitar Pro **`.gp`** files —
porting v1's four families (chromatic, scales, arpeggios, intervals) with **no
change to the generation core**, and removing LilyPond entirely. That is exactly
what shipped: melete now emits alphaTex through a vendored `melete-render` Node
tool, renders a day's session to `.gp`, and carries LilyPond nowhere. The port
was cold-rebuild-validated end to end and a real sample opened cleanly in
Guitar Pro.

The bulk of what was done:

| PR | What it did |
|---|---|
| `.github#50` | Spec + plan for the epic |
| `.github#52` | **Spike** findings — confirmed alphaTex tokens, the alphaTab Node API, and Guitar Pro 8 fidelity |
| `melete#95` | **Container capability** — bake alphaTab into the image via `[container].build-command` |
| `melete#96` | `melete-render` — the dumb alphaTex→`.gp` Node renderer |
| `melete#97` | **IR evolution** — `Note.tied`, `Measure`, lift the tuplet-nesting prohibition (additive) |
| `melete#98` | The **barring pass** — split-and-tie a voice into `Measure`s |
| `melete#99` | The **alphaTex emitter** — the content-correctness heart |
| `melete#100` | The Python **blast door** (`alphatab/render.py`) |
| `melete#101` | Wire the `generate` CLI to the new pipeline (→ `.gp`) |
| `melete#102` | Black-box integration test — real alphaTex → valid `.gp` |
| `melete#103` | **Remove LilyPond** — delete `lilypond/`, drop the container package |
| `melete#105` | Gitignore the `vrg-validate` quality reports (hygiene) |
| `melete#106` | Docs review — reflect the port across melete's docs |
| `melete#107` | De-pollute `CLAUDE.md`; drop the inert `[output]` config + `--staves` |
| `docs#9` | Docs-site: reflect melete's alphaTab/`.gp` output |
| `.github#<this>` | This retrospective (closes the epic) |

- **Repos touched:** `mnemosys-project/melete` (12 PRs), `mnemosys-project/.github`
  (spec/plan, spike, this retro), `mnemosys-project/docs` (1 PR).
- **Spawned upstream:** a whole vergil-tooling capability — the `[container]`
  build-command hook — that the port forced into existence (see §4).
- **Tasks:** the 10 port tasks + a cold-rebuild validation + 5 bookends.
- **Span:** opened **2026-08-11**, closed **2026-08-13** (~2 days).
- **Releases cut:** none for melete (still pre-release); the port drove a
  vergil-tooling release (`2.1.191`) for its container capability.

## §1 How the plan evolved

The plan held remarkably well at the level of *tasks* — all ten ran in the
planned order and shape — but three things it could not have known reshaped the
execution.

**The container capability did not exist, and the port had to build it first.**
The plan's Task 2 assumed melete's `[container]` mechanism could bake the
alphaTab npm dependency the way it bakes LilyPond's apt package. It could not:
`[container]` was apt-only, with no hook to run `npm install` at image build.
Rather than hand-roll a workaround, we filed a vergil-tooling triage that became
its own epic (`vergil-project/.github#291`) — a `[container].build-command` hook
plus a `NODE_PATH` fix — and **blocked the whole port on it** until it shipped in
vergil-tooling `2.1.191`. The single largest deviation, and the right call: a
reusable capability instead of a melete-local hack.

**The renderer had to be CommonJS, not the plan's ESM.** That same capability
resolves the baked dependency over `NODE_PATH`, which only CommonJS `require`
honours — ESM `import` ignores it. So `melete-render/render.js` shipped as
CommonJS, not the plan's `render.mjs`. alphaTab being a dual package made this
free (`require` yields the full API).

**The octave convention was re-evaluated, not ported.** The spike's Guitar Pro 8
check showed the notes sitting oddly under an `8vb` clef; the owner chose a plain
`\clef bass` at sounding pitch — a deliberate departure from v1's `bass_8`, and
exactly the "LilyPond-suspect principle" the spec invited. Spelling fidelity was
*kept* (the emitter reproduces `theory.spell`'s exact enharmonics via `\ks` +
forced accidentals).

Smaller course-corrections: the nesting lift was applied at the **runtime check
only** (static type stayed `list[Note]`) to avoid cascading type errors into the
soon-deleted LilyPond emitter; the barring pass stayed **real work** (the spike
confirmed alphaTab neither auto-bars nor auto-ties, so the plan's "collapse to a
no-op" contingency never fired); and Task 10 was **relaxed** by the owner —
recoverability guards (a git tag + a committed v1 sample) dropped as
disproportionate for a one-day prototype nobody will check out and rebuild.

## §2 Lessons learned

- **The thinnest-boundary bet paid off.** Keeping the Python↔alphaTab interface
  as *text over a subprocess* (alphaTex) meant a one-day spike could de-risk the
  entire architecture: it pinned every token and the Node API by experiment, and
  every downstream task consumed confirmed syntax rather than guessing. The
  emitter's goldens are `theory.spell`'s exact output, so content correctness was
  provable without opening Guitar Pro.
- **Some gates are irreducibly human — design the automation up to them.** "Does
  this `.gp` render correctly in Guitar Pro 8" cannot be automated. But producing
  the *candidate* `.gp` can, which turned a multi-hour manual spike into a
  five-minute eyeball and made the human gate cheap to hit repeatedly.
- **Verify container/tooling capabilities before planning tasks on them.** The
  plan assumed `[container]` could bake an npm dep; it couldn't. Confirming that
  during planning (as the spike did for alphaTex) would have surfaced the
  vergil-tooling dependency a step earlier.
- **A renderer swap is 90% not the renderer.** Nearly everything — theory, the
  spelling model, the families, `score.py` — survived untouched; the seam held
  exactly as epic #1 §4 promised.

## §3 Compromises & tradeoffs

- **Nested tuplets flatten.** alphaTab has no nested-tuplet-bracket syntax, so a
  nested tuplet is emitted as a flattened cumulative ratio — exact in sound,
  visually flat. Moot in practice (no family emits one), recorded against the
  lifted nesting invariant.
- **The `key_signatures` "no signature" mode was retired.** The alphaTab emitter
  always writes `\ks`; v1's option to omit the signature was already an inert
  no-op under alphaTab and its config key was removed in `melete#107`. A genuine
  capability drop, intended.
- **The `.gp` cover is thinner than v1's.** A day's book carries `\section`
  titles + a date/instrument subtitle, not LilyPond's full axis-listing cover
  page. Acceptable for a `.gp`-parity port; noted here rather than silently lost.
- **`melete-render` is acknowledged, vendored tech debt.** A dumb Node tool
  living inside melete's renderer boundary, deliberately extraction-ready; the
  extract/defer/drop decision was *not* pre-committed — it is `melete#109`.
- **The nesting lift is slightly type-dishonest by design.** `Tuplet.notes` stays
  `list[Note]` statically while the runtime accepts a nested `Tuplet`; documented,
  and the honest-union alternative (which would have touched the deleted emitter)
  was deliberately not taken.

## §4 New problems & opportunities

- **A reusable container capability, born here.** The port surfaced that
  `[container]` couldn't bake non-apt deps and drove the fix into vergil-tooling:
  triage `vergil-project/vergil-tooling#2751` → epic
  `vergil-project/.github#291` (the `build-command` hook + `NODE_PATH`). Every
  org repo now has it. **Acted on — shipped.**
- **CI resolution parity for alphaTab** is tracked separately as
  `vergil-project/vergil-actions#824`; it only bites once a CI test imports
  alphaTab (the Node-gated integration test), so it did not block the port.
  *Logged.*
- **`docs#5` staleness.** Mid-flight, a separate docs-site PR (`docs#8`) landed a
  status page written from the *pre-port* perspective — "the renderer is being
  replaced," "migrating is the next epic," and a "why it is not installed yet"
  rationale premised on the renderer about to change. `docs#9` corrected the
  clear-cut output-format drift; the broader narrative needs a product-informed
  rewrite (melete's actual install/release status). **Logged, not yet acted on.**
- **Tooling-artifact hygiene.** `vrg-validate` emits `quality-mypy.xml` /
  `quality-ruff.json`, which weren't gitignored and snagged worktree cleanup;
  fixed for melete in `melete#105`. Likely worth a template-level fix org-wide.
  *Acted on locally.*

## §5 What's next

The port existed to unlock advanced guitar techniques LilyPond could not express.
Both forward items from the follow-on brainstorm (`melete#82`) were **deferred to
the ad-hoc epic** (`mnemosys-project/.github#12`) to be triaged deliberately
rather than rushed:

- **Tapping family (`melete#108`)** — the next epic. Generate two-handed tapping
  exercises (specifying left/right tapping fingers, over arpeggio/scale patterns)
  to `.gp`. One design fork to resolve at epic-create: a single hand-annotated
  line vs. genuine two-voice polyphony (the "measures → voices" IR evolution).
- **`melete-render` extraction (`melete#109`)** — defer / extract to its own
  TypeScript repo (now feasible on the Vergil TS-support initiative) / drop.

A forward-looking brainstorm on real-world feedback (from bringing the `.gp`
output to a bassist) is expected to open the next iteration in a fresh session.

## Appendix A — Operational notes

- **Cold-rebuild validation (`melete#94`).** Acceptance was proven by evicting the
  cached dev image and rebuilding from scratch: the `build-command` ran
  (`npm install` of alphaTab), Node + alphaTab resolved at
  `/usr/lib/node_modules`, and a `generate` run produced a valid, re-importable
  `.gp` (book + split exercises). Recorded `Outcome: SUCCESS`.
- **Release ordering.** The port could not proceed past Task 2 until
  vergil-tooling `2.1.191` (the `build-command` + `NODE_PATH` fix) was installed;
  CI parity (`vergil-actions#824`) remains a later, non-blocking release.
