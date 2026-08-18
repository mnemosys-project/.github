# Retrospective — Roll out centralized .gitignore baseline + self-policing ops audit

**Epic:** [`mnemosys-project/.github#65`](https://github.com/mnemosys-project/.github/issues/65)
**Partners:** *(no `spec.md` / `plan.md` — see §1)* → this record
**Opened:** 2026-08-13 · **Closed:** 2026-08-18

## §0 At a glance

We set out to apply the centralized-`.gitignore`-baseline and
self-policing-`ops.yml` rollout — designed and shipped from `vergil-tooling`
under **vergil-project/.github#311** — to this org's managed repos. **No new
design**: the baseline, the audit checks, and the reusable workflow all arrive
from #311, and this epic was purely the operational arm. We shipped exactly
that, across all three managed repos, plus one scope correction the charter's
own rule implied but the decomposition had missed (§1).

**What shipped:** every managed repo in `mnemosys-project` now carries a
`.gitignore` that is a verbatim superset of the released baseline and a
`.github/workflows/ops.yml` calling
`vergil-actions/.github/workflows/ops-github-config.yml@v2.1` on a
deterministic, per-repo **staggered** cron minute. `vrg-github-repo-config
audit` reports `local: compliant` in all three. Baseline drift is now a nightly
red-X in each repo, feeding the GitHub-failure-email operational discipline.

### Work delivered

| PR | Task | What it did |
|---|---|---|
| melete#225 | #131 | reconcile `.gitignore` to the baseline; add `ops.yml` (cron `23 6 * * *`) |
| docs#11 | #10 | adopt the baseline; add `ops.yml` (cron `55 6 * * *`) |
| .github#85 | #84 | adopt the baseline; add `ops.yml` (cron `27 6 * * *`) |

- **Repos touched:** `mnemosys-project/melete`, `mnemosys-project/docs`,
  `mnemosys-project/.github` — all three managed members, i.e. the whole org.
- **Children:** 4 (3 rollout tasks + this retrospective). 3 PRs, all merged.
- **Findings cleared:** 83 `local.*` audit findings → **0** (melete 19, docs 32,
  `.github` 32).
- **Releases cut:** none — no release was required.
- **Span:** opened 2026-08-13, closed 2026-08-18. Nearly all of that was
  *waiting on the #311 precondition*; once the tooling was live, execution ran
  in a single sitting — first task filed 19:47Z, last PR merged 19:55Z.

## §1 How the plan evolved

**This epic had no `spec.md` and no `plan.md`**, and therefore no "Evolution
during execution" log to synthesize from. That was deliberate and correct: the
charter opens with *"No new design"* — the design lives in #311, and this epic
carried only a per-repo step list in its issue body. It is recorded here rather
than papered over, because it has a consequence: the plan-vs-actual delta that
is this artifact's deeper purpose cannot be measured for this epic. What follows
is the one substantive deviation, reconstructed from the execution record.

**The epic was under-decomposed at creation.** It was opened with a single
rollout task (`melete#131`) against a charter whose scope clause read *"for each
managed repo in this org"*, qualified by an exclusion rule: non-managed repos
(bare `.github`, `docs`, `template`) are out of scope **"unless they carry
`vergil.toml`."** Checking that rule against the actual repos showed all three —
`melete`, `docs`, **and** `.github` — carry `vergil.toml`. So `docs` and
`.github` were in scope by the charter's own test, but had no tasks.

This was caught at the start of `epic-implement`'s first frontier pass, raised
with the human, and resolved by filing `docs#10` and `.github#84` as siblings of
`#131` before any work began. Had it not been caught, the epic would have closed
having delivered one-third of its stated goal, and its closing claim — *"once
every repo carries `ops.yml`, this org self-polices"* — would have been false
while reading as satisfied.

Everything else went as the charter described. The three tasks were
non-interdependent same-repo changes, ran in parallel, and merged within ~45
seconds of each other.

## §2 Lessons learned

- **A charter that states a membership *rule* should be decomposed by executing
  that rule, not by hand.** The exclusion clause was written correctly and still
  produced a wrong task list, because decomposition enumerated repos from
  memory instead of testing each against `vergil.toml`. Where a charter encodes
  a predicate, run the predicate.
- **Derive mechanical values from the tooling source; never let agents invent
  them.** The cron stagger is
  `sha256("<org>/<name>").digest()[0] % 60` (`repo_init._ops_cron_minute`), and
  `ops.yml` has a canonical renderer (`repo_init.render_ops_workflow`). Reading
  those and handing all three agents the exact expected bytes meant the three
  files came out byte-identical to what `vrg-github-repo-init` generates. Three
  agents each reasoning independently about "a deterministic staggered minute"
  would have produced three plausible, mutually inconsistent answers.
- **"Superset" was the load-bearing word, and it needed spelling out.** The
  audit (`_check_gitignore`) requires each baseline pattern verbatim, so a repo
  with a near-miss spelling passes by *appending* the canonical form beside its
  own. That satisfies the checker and leaves the file worse. Stating
  "standardize, don't append duplicates" up front is what made the diffs clean.
- **Verify sub-agent reports; don't just relay them.** All three agents reported
  success accurately, but diffing each `.gitignore` for *removed* pattern lines
  surfaced one real narrowing (§3) that no report mentioned, because the agent
  had classified it as a spelling fix. The check is cheap and belongs in the
  gate.

## §3 Compromises & tradeoffs

- **`melete` lost `.pyo` / `.pyd` coverage.** Its pre-existing `*.py[cod]` was
  replaced by the baseline's `*.pyc`, which is strictly narrower. This was
  flagged at the human gate as a decision, with the note that the baseline
  requires only a *superset* so keeping both lines would have been compliant;
  the batch was merged without a call either way, so the narrowing landed. Low
  practical impact — `.pyo` has not been produced since Python 3.5 and `.pyd` is
  Windows C-extension output on a Linux/macOS project — but it is a real
  regression, not a spelling fix. Logged in §4.
- **Only the local half of the audit was ever verified.**
  `vrg-github-repo-config audit` shells out to raw `gh api
  repos/<repo>/actions/permissions`, which returns `403 Resource not accessible
  by integration` under the USER identity. Every "clean audit" claim in this
  epic — including step 4 of each task — means `local.*` only. The rollout is
  entirely local-file work so this did not block, and it was reported rather
  than suppressed at every step, but the remote-side compliance of these three
  repos remains unverified by this epic.
- **No spec or plan, by design.** Correct for a pure operational arm of another
  org's design epic, and it kept the overhead proportionate. The cost is the one
  named in §1: no plan-delta to measure, and a scope error that a written plan's
  task list would likely have caught at authoring time rather than at
  execution time.

## §4 New problems & opportunities

- **The `.pyo` / `.pyd` gap in `melete`'s `.gitignore`** — restoring `*.py[cod]`
  alongside `*.pyc` is a one-line change and stays baseline-compliant. **Logged,
  not yet acted on**; the merged branch is frozen, so this needs a follow-up
  task if wanted.
- **The remote half of `repo-config audit` is unrunnable under the USER
  identity** — the `403` on `actions/permissions` means agents structurally
  cannot verify remote compliance. Worth deciding whether that check should be
  reachable via `vrg-gh`, or explicitly declared CI-only so the local tool stops
  reporting a partial result as an audit. **Logged, not yet acted on** — and
  relevant to #311's own follow-ons, since the tool ships from there.
- **`ops.yml` cannot detect a repo that has no `ops.yml`.** The wiring validator
  only fires inside a repo that already runs it; a managed repo that never
  adopts one is invisible to the nightly sweep. The tooling names this
  explicitly and defers the from-outside guarantee to follow-on **C (#315)**.
  This org is fully covered *today*, so the exposure is future repos — a new
  repo is only protected if `repo-init` gives it `ops.yml`, or #315 lands.
- **A stray empty directory named `mkdir`** sits untracked at the root of
  `mnemosys-project/.github`, apparently residue from a mistyped command.
  Trivial, out of scope, **logged**.

## §5 What's next

- **Outcomes roll up to
  [vergil-project/.github#311](https://github.com/vergil-project/.github/issues/311)'s
  retrospective §5**, per this epic's charter. The two findings worth carrying
  across the org boundary are the `403` on the audit's remote half (§4) and the
  missing-`ops.yml` blind spot already tracked as **#315** (§4) — both are
  properties of the shipped tooling, not of this org's rollout.
- **No follow-on brainstorm is warranted.** This was a bounded operational
  sweep against an external design; there is no forward-axis question left open
  by it.

## Appendix A — Operational notes

The mechanical sequence, for whoever runs this rollout in the next org:

1. **Confirm the precondition by observation, not by date.** The tooling is live
   when `vrg-github-repo-config audit` starts emitting `local.gitignore` and
   `local.ops_workflow` findings. That is a stronger signal than checking for a
   release tag.
2. **Derive the task list by testing `vergil.toml` presence** against every repo
   in the org (`vrg-gh repo list <org>`), not from memory. See §1.
3. **Read the canonical values out of the installed tooling** before dispatching
   any agent:
   - cron minute — `repo_init._ops_cron_minute(org, name)` =
     `sha256(f"{org}/{name}").digest()[0] % 60`
   - `ops.yml` body — `repo_init.render_ops_workflow(ctx)`, verbatim
   For this org: `melete` 23, `docs` 55, `.github` 27.
4. **Run the repos in parallel.** The tasks are non-interdependent same-repo
   changes; each gets its own worktree and branch, and each repo's PR is
   submitted as its own `vrg-submit-pr` batch (the tool takes a branch list
   within a single repo).
5. **Before handing PRs to the human, diff each `.gitignore` for *removed*
   pattern lines**, excluding comments. Relocations into canonical sections are
   expected and fine; an actually-dropped pattern is not. This is what caught
   §3's narrowing.
6. **Verify each `ops.yml` against the canonical renderer** rather than reading
   it, e.g. comparing the file bytes to `render_ops_workflow(ctx)` output.

Acceptance per repo was: zero `local.*` findings, `vrg-container-run --
vrg-validate` green, and the branch recorded via `vrg-pr-workflow report-ready`.
