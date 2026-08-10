# Contributing to the Mnemosys Project

Thank you for your interest. This document covers how work is organized here
and what you need to get started.

## How the project works

The organization is named for memory; each tool under it is named for a Muse.
That scheme is not decorative and it is not optional — read
[`NAMING.md`](NAMING.md) before proposing a tool or a name for one.

| Repository | Purpose |
| --- | --- |
| [.github](https://github.com/mnemosys-project/.github) | Org metadata, community health files, the naming convention, and the home for all epics |
| `docs` | The organization site. Not yet created. |
| `melete` | Practice exercise generator. Not yet created. |

The organization is new. Repositories appear as the bootstrap epic delivers
them, and this table is updated when they do.

## Work is tracked as epics and tasks

**Every pull request has a primary issue, with no exceptions.** The issue comes
first; work does not begin until it exists; a PR targets one primary issue.

- An **epic** is an initiative spanning several PRs. Epics live in this
  repository, never in a member repo.
- A **task** is a unit of work closed by a single PR. A task lives in the
  repository where its closing PR lands — because a PR can only close an issue
  in its own repo. Cross-repo relationships are references, never closing
  keywords.

Never hand-roll a sub-issue link. Use the sanctioned tools:

```bash
# create a task already linked under an epic
vrg-issue-create --epic mnemosys-project/.github#<N> --repo <owner>/<repo> --title "…"

# link an existing task under an epic
vrg-epic-link --epic mnemosys-project/.github#<N> --task <owner>/<repo>#<task>
```

If the work reduces to one PR, it is a task, not an epic.

## Development setup

### Prerequisites

- **Docker** — the only host-side dependency for validation. Linting, type
  checking, and tests all run inside dev containers.
- **[uv](https://docs.astral.sh/uv/)** — used to install the host-side CLI.

### Install the tooling

```bash
uv tool install --python 3.14 \
  'vergil-tooling @ git+https://github.com/vergil-project/vergil-tooling@v2.1'
```

This installs the `vrg-*` commands used throughout this document.

## Workflow

All repositories use `develop` as the integration branch. **There are no direct
commits to `develop`.**

1. **Branch from `develop`**, named `feature/<issue>-<slug>` or
   `chore/<issue>-<slug>`, where `<issue>` is the GitHub issue number.

2. **Commit with `vrg-commit`.** It enforces conventional commit format, branch
   policy, and attribution. Raw `git commit` is blocked by the pre-commit hook.

   ```bash
   vrg-commit --type feat --scope selection --message "add coverage-aware sampling"
   ```

3. **Validate locally.** This is the *only* validation command — do not run
   individual linters or formatters outside it. If a tool is not invoked by
   `vrg-validate`, it is not part of the pipeline.

   ```bash
   vrg-container-run -- vrg-validate
   ```

4. **Submit the PR with `vrg-submit-pr`.** It builds a standards-compliant body
   linked to the issue.

5. **Wait for review.** Every PR needs human approval and passing CI.

### Closing issues

Use a closing keyword only when the issue has no special acceptance criteria.
When it does, reference the issue without closing it and close it by hand once
the criteria are actually satisfied.

A closed issue must reflect completed work. If part of it is deferred, keep it
open or file a follow-up and link the two explicitly.

## Working with AI agents

This project is built with AI assistance, deliberately and openly. The banner on
the org profile states the position: the automata build and maintain the stage,
and they are not among the Muses. AI built this project; AI is not in the
product. `melete` embeds no model and calls no inference endpoint.

Two rules govern agent contributions:

**You are accountable for what your agent produces.** "The AI did it" is not a
defense. Submitting the work asserts that you reviewed it, understand it, and
stand behind it.

**Agents do not submit or merge pull requests, and do not create repositories.**
An agent records readiness with `vrg-pr-workflow report-ready`; a human runs
`vrg-submit-pr` and merges. Repository creation is likewise a human act. These
boundaries are enforced by the tooling, not by convention.

### Parallel agents

Multiple agents work in parallel via git worktrees under `.worktrees/`, one per
issue, each on its own feature branch. Sessions always start at the project
root, never inside a worktree, and the main worktree is read-only. The
convention is documented in each repository's `CLAUDE.md`.

## Contributing without the tooling

You are welcome to contribute without installing any of this. Fork, branch,
change, and open a PR that links its issue, says what changed and why, and shows
how you verified it. A maintainer will route it.

The same quality bar applies regardless of how the work was produced.

## Template inheritance

The files in this repository are org-wide defaults. GitHub's inheritance model
is worth knowing before you override one:

- **Standalone files** — `CONTRIBUTING.md`, `SECURITY.md`, `CODE_OF_CONDUCT.md`,
  `SUPPORT.md` — inherit **independently**. A repository can override one and
  keep the rest.
- **Template directories** — `ISSUE_TEMPLATE/` and `pull_request_template.md` —
  are **all-or-nothing**. A repository containing *any* file in its own
  `.github/ISSUE_TEMPLATE/` replaces the entire org-level set for that
  repository, not just the file it supplied.

So a member repo that needs one custom issue form must provide the full set.

This repository must stay **public** for any of that inheritance to work.

## License

All Mnemosys Project repositories are licensed under the
[MIT License](https://opensource.org/licenses/MIT). By contributing, you agree
that your contributions are licensed under the same terms.
