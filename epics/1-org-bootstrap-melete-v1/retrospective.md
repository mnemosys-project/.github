# Org Bootstrap and Melete v1 — Retrospective

**Epic:** [`mnemosys-project/.github#1`][epic]
**Spec:** [`spec.md`](./spec.md) · **Plan:** [`plan.md`](./plan.md)
**As approved:** [`spec-as-approved.md`](./spec-as-approved.md)
**Date:** 2026-08-11

[epic]: https://github.com/mnemosys-project/.github/issues/1

## §0 At a glance

We set out to bootstrap an organization from nothing and deliver `melete` v1 — a
command that generates a daily bass practice sheet good enough to hand to an
instructor. The organization exists and is complete. Melete is built, runs end
to end, and has produced real printed sheets. It is **not** in daily use,
because rendering the sheets exposed enough about LilyPond that the renderer is
being replaced, and the epic is closing at that milestone rather than deploying
a tool that is about to change underneath its user.

The v1 success criterion — *run one command each morning and get a printable
sheet* — is **met mechanically and withdrawn deliberately**. That distinction is
the honest summary of this epic.

### Work delivered

| | |
|---|---|
| **Repositories created** | `mnemosys-project/.github`, `mnemosys-project/docs`, `mnemosys-project/melete` |
| **Tracked children** | 54 (53 closed, this retrospective the 54th) |
| **Pull requests merged** | **56** — `.github` 18, `melete` 36, `docs` 2 |
| **Releases cut** | **none** — melete is at `0.1.0`, unreleased, by design |
| **Opened → closed** | 2026-08-09 20:32 UTC → 2026-08-11 (≈ 2 days) |
| **Cross-org issues filed** | 4 in `vergil-project/vergil-tooling` (#2717, #2718, #2720, #2721) |
| **Melete at close** | 19 modules, 2,711 tests, 100% coverage including branches |

The full PR enumeration is in **Appendix B**; a fifty-six-row table is not a
landing view. Grouped by what it delivered:

| Group | PRs | What it did |
|---|---|---|
| Org bootstrap | 6 | Naming convention, org metadata and health files, epic document formats, banner image and its design record, org site |
| Melete scaffolding | 3 | Python project, container LilyPond, `examples/` |
| Core model | 5 | `theory`, `instrument`, `vocabulary`, `score` IR, `config` |
| Exercise families | 5 | chromatic, scales, arpeggios, intervals, shared helpers |
| Pipeline | 5 | `rhythm`, `selection`, `session`, `emit`, `render` |
| CLI | 2 | `generate` and its seven flags; `replay`, `show`, `families`, `vocabulary` |
| Spelling model (Phase S) | 5 | Key signatures and accidental spelling, spec §10a, decisions #27–30 |
| Defects found on paper | 2 | Hand-span bound; the double-octave clef fix |
| Documentation | 12 | Reference docs, renderer boundary, spec corrections, evaluation reports |
| Corrections to the epic's own documents | 11 | Spec and plan amendments as the design moved |

**Eleven of fifty-six pull requests existed to correct the epic's own
specification and plan.** That number is not a failure; it is the measurement
this retrospective exists to take.

## §1 How the plan evolved

The plan's Evolution log carries 23 entries. They fall into three groups, and
the groups say different things.

**The plan's own details aged, and the tests caught it.** Six entries record the
plan being wrong in specific, checkable ways: task A1's template paths pointed
at `.github/ISSUE_TEMPLATE/` when an organization's templates live at the
repository root; its planned `task.yml` mirrored a label that does not exist in
this org's registry; its discovery command used `vrg-gh api`, which is denied to
the agent identity that was supposed to run it. Task B3's sketch listed 11 scale
types where `theory` defines 27 — a sketch that would have failed its own
drift-guard test. B13's sketch asserted an error message that a decision had
already superseded. B5's asserted a tempo the spec no longer carried.

None of these reached the code. Each was caught by an implementer checking the
plan against the source rather than transcribing it, which is the behaviour the
plan's "no placeholders" rule was written to make possible. **A plan detailed
enough to be wrong is more useful than one vague enough to be unfalsifiable.**

**Two reversals changed the shape of the system.** Decision #13 — accept the
unmaintained PyPI LilyPond redistribution and fork it if inadequate — assumed
the risk was staleness. The actual defect was that no aarch64 wheel exists at
any version, and upstream publishes no `linux-arm64` binary either, so forking
could not fix the container. LilyPond became a system binary, which the shared
container model had no way to express, which became a change to the Vergil
toolchain itself.

Decision #9 — notate everything in C with explicit accidentals — was not a
notation choice at all. It papered over a missing layer: a 12-TET integer cannot
distinguish F♯ from G♭, so "no key signature" silently meant "wrong key." The
reversal produced §10a, three spelling tiers, and four new decisions. Both
reversals are recorded as *half-right* rather than wrong, because in each case
the reasoning survived and only the implementation failed.

**The printed page found what the tests could not.** Two entries record defects
that appeared the first time a human looked at a rendered sheet: three of five
exercises were unplayable, one of them labelled `positional` while demanding a
fourteen-fret reach; and every exercise had been engraved **two octaves above
its sound** for the whole of Phase B, because the emitter transposed and an
octavated clef transposed again. The tablature was correct throughout. 2,711
tests at 100% branch coverage passed throughout.

## §2 Lessons learned

**Reference documentation written from the source is a spec audit in disguise.**
Writing melete's CLI and configuration references meant reading `cli.py` and
`config.py` line by line. That surfaced seven places where the spec described a
tool we had not built — including `replay`, whose true semantics contradicted
the spec in four separate sections. The documentation-review bookend caught the
documents that mirror *decisions*; only prose written against the code caught
the document that mirrors *behaviour*.

**Golden-file tests pin text, not correctness.** Both defects that mattered were
invisible to a full suite at 100% branch coverage, because the tests asserted
that the emitter produced the text we intended, and the text we intended was
wrong. For any generated-output target, the only real gate is rendering the
output and having a person look at it.

**Therefore: do not schedule the paper gate last.** C2 existed precisely to
catch what unit tests cannot, and it sat at the end of the plan as a formality.
Moving it to the first moment the pipeline produced a document would have caught
the octave bug days earlier and at a fraction of the cost. This is the single
change most worth making to how the next epic is planned.

**A closed issue is not a fixed issue.** `vergil-tooling#2720` was closed, and
on that basis we nearly re-enabled a CI flag that would have made the repository
unmergeable again. Reading *how* it closed showed it had added a guard that
*detects* the mismatch without making the context emit. Checking `--json state`
alone would have shipped the bug twice.

**Drift guards earn their keep.** Tests that assert two sources agree — the
vocabulary registry against `theory.SCALES`, `_AXES_BY_FAMILY` against each
family's declared `AXES`, the family tempo table against spec §7 — caught real
divergences repeatedly, including one that made the selector unable to produce a
valid `intervals` specification at all. Every one of them was cheap.

**Extract on the second occurrence, not in anticipation.** The plan proposed a
shared `assign_positions` helper. Having written the second family, the
implementer declined it: `chromatic` derives position from finger and shift and
reads pitch out of it, the opposite direction from `scales`, so one signature
would have had a dead half per caller. It was extracted later, when a third
family made the duplication real.

**A scope boundary stated too strongly makes decisions for you.** §3 listed
audio and MIDI among things "permanently out of scope," which silently removed a
real argument from a rendering decision. Distinguishing *refused* from *not yet*
is not pedantry; it is the difference between a boundary and an accident.

## §3 Compromises and tradeoffs

**The tool was never actually used.** C1 (deploy) and C2 (validate five days of
printed sheets) were closed will-not-implement. They were deliberate scope
decisions rather than skipped gates — deploying a tool whose renderer is being
replaced means building habit around output about to change — but the honest
consequence is that **melete has never been played from**. One sheet was read.
Five days of real practice would have found things one sheet did not.

**Integration tests do not run in CI.** They exist, they pass, and they run in
the ordinary pytest invocation — but `integration-tests = false` in
`vergil.toml`, because setting it true adds a required status check the
generated workflow never emits, which blocks every pull request with all checks
green. The upstream fix is `vergil-tooling#2721`, still open. The coverage is
real; the CI guarantee is not.

**The spelling policy is convention, unreviewed by an expert.** Tier 1 is fully
determined. Tiers 2 and 3 are defensible convention chosen by an agent and a
hobbyist, and they are what an instructor with a theory degree is most likely to
correct. Known and accepted: the blue note spells ♭5 rather than ♯4; the plain
`dim` triad spells its fifth as a raised fourth when locrian would spell it
correctly; the fully diminished seventh cannot be spelled functionally at all
under this model; symmetric scales are spelled by direction, so letters skip and
repeat.

**The hand-span rule excludes open strings, and that is blunt.** It is
physically justified — fret 0 sounds while the fretting hand stays put — and
without it the open-A pentatonic box is refused as a seven-fret stretch. But it
cannot distinguish an open string at the bottom of a low shape from one
interleaved with a hand high on the neck. Accepted as a temporary
simplification, tracked as `melete#60`, and recorded as provisional in decision
#37 so a later reader knows it was a knowing choice rather than a rule someone
believed was complete.

**The tablature staff spells keylessly**, so in a flat key the two staves carry
the same pitch under different names — `ges` above, `fis` below. Required by the
boundary claim that spelling must not reach tablature, and cosmetically odd in
the source.

**The epic's own documents ran behind its code, repeatedly.** Eleven of
fifty-six PRs were corrections to the spec and plan. Each individually was
justified; collectively they show a spec that stated intentions in the present
tense with nothing checking them. Decision #38 and #42 record the pattern.

**Almost nothing under `docs/` is linted.** `vrg-validate`'s markdown discovery
finds `docs/site/**` plus `README.md`. Neither `.github` nor `melete` has a
`docs/site/`, so the naming convention, all four health files, both 2,000-line
epic documents and every reference document written this epic are checked by
nothing — including the document that *declares* the 80-column standard. Filed
as `.github#42`.

## §4 New problems and opportunities

| Surfaced | Where it went |
|---|---|
| LilyPond imposes costs disproportionate to its benefit here: no aarch64 distribution, semantics traps, unverifiable output | `melete#71` — closed; the founding input to the renderer migration |
| Renderer alternatives evaluated | `melete#70`, `#72`, `#73` — annotation gap analysis, generation feasibility, notation display targets |
| Open strings and hand span need a less blunt rule | `melete#60` — ad-hoc, deliberately not blocking v1 |
| Melete has none of the CI gates its sibling repos carry, contrary to spec §15 | `melete#79` — ad-hoc, with the constraint that adding a required context must add its emitting job in the same change |
| `vrg-github-repo-init` clones into `$CWD` and resumes another repo's wizard state | `vergil-tooling#2717` |
| Vergil has no way to express a repo-specific system dependency | `vergil-tooling#2718` — **fixed**, produced `[container] system-packages` |
| Required status checks the generated workflow never emits | `vergil-tooling#2720` — **fixed** for `publish-docs`; guarded, not fixed, for integration tests |
| Integration tests are not a first-class Vergil feature | `vergil-tooling#2721` — **open**; blocks melete's CI integration coverage |
| Markdown outside `docs/site/` is unlinted | `.github#42` — ad-hoc |
| The org site describes a project with no code | `docs#5`, `docs#6` — moved to ad-hoc; deferred until the architecture settles |
| Audio and MIDI were wrongly excluded permanently, foreclosing objective measurement | Spec §3 corrected; §17 gained *Performance capture* |

The last one is the most consequential opportunity. The project's thesis is that
the domain is retention and the hard problem is decay — and decay is currently
self-reported. Recording a performance and scoring it is what would make that
claim testable rather than asserted.

## §5 What's next

**The renderer migration epic.** Not yet created; it should start with its own
brainstorm rather than inherit assumptions from this one. Its inputs are
`melete#71` and the three evaluation reports in melete's `docs/reports/`. The
requirements list in `#71` was deliberately written before the candidate tools
were known, and is worth more for it.

**The measurement layer**, per §17 — practice logging, progression, mastery and
fatigue, and now performance capture. This is where the tool stops being a
generator and starts being an instrument for the thing it was named after.

**`aoede`** remains reserved for repertoire management. Unstarted, unchanged.

**Not yet decided:** whether melete deploys on the current renderer for interim
use, or waits. It is closed as will-not-implement for now, and reinstating it is
a scope decision, not a gate to reopen.

## Appendix B — Extended metrics

### Merged pull requests

**`mnemosys-project/.github` (18)** — #7 spec and plan published, #9 banner
integrated, #10 org metadata, #11 epic document formats, #14 LilyPond as binary
prerequisite, #16 profile README rewritten, #19 tempo and nesting decisions, #20
accidental spelling and key signatures, #22 Phase S plan, #25 banner statement
corrected, #30 arpeggio parent table, #32 min6/min_maj7 divergence recorded, #34
diminished-triad and keyless-tab spellings, #37 §10 example config made working,
#39 hand-span bounds, #41 scope exclusions separated from deferrals,
#43 renderer boundary marked, #45 seven spec corrections.

**`mnemosys-project/melete` (36)** — #22 project scaffolding, #24 integration
tests off, #25 theory, #26 instrument, #27 render adapter, #28 vocabulary, #29
config, #30 Score IR, #31 rhythm, #33 emit, #34 chromatic and registry, #36
design pointer, #37 scales, #42 spelling model, #43 key signatures default, #44
shared family helpers, #45 score and rhythm carry the key, #46 arpeggios, #47
key signatures printed, #48 intervals, #49 tempo and axes from the registry, #50
selection, #53 session and replay, #54 CLI generate, #55 container LilyPond, #59
hand-span bound, #62 examples move, #63 CLI query commands, #64
integration-tests comment, #66 clef double-transposition fix, #70/#72/#73
evaluation reports, #77 CLI and configuration references, #78 repository
standards, #80 renderer boundary.

**`mnemosys-project/docs` (2)** — #2 site content, #4 `docs/docs` context
emitted on pull request.

### Notes on the counts

- PR numbers share a sequence with issues, so they are not contiguous.
- `melete#70`, `#72` and `#73` are evaluation reports authored outside the agent
  workflow; they are counted as merged PRs but produced no code.
- 54 tracked children excludes issues later re-parented to ad-hoc epics
  (`docs#5`, `docs#6`) and issues filed ad-hoc from the outset (`melete#21`,
  `#60`, `#79`).
- Wall-clock span is approximately two days. Human input was roughly eight
  hours.
