# .github

Organization metadata, epic home, and community health files for the Mnemosys Project.

## Table of Contents

- [Status](#status)
- [Overview](#overview)
- [What is here](#what-is-here)
- [Getting Started](#getting-started)
- [License](#license)

## Status

Active. This repository holds org-wide metadata and is the home for every epic
in the organization.

## Overview

This is the organization's `.github` repository. It serves two distinct roles:

- **Community health defaults.** `CONTRIBUTING.md`, `SECURITY.md`,
  `SUPPORT.md` and `CODE_OF_CONDUCT.md`, plus the org-wide issue and pull
  request templates, are inherited by every repository in the organization that
  does not override them. This repository must stay public for that to work.
- **The epic home.** Every epic in the organization lives here, under
  `epics/<N>-<slug>/`, regardless of which repository its work lands in. A task
  is filed in the repository where its closing PR lands; the epic that groups
  those tasks is filed here.

The organization profile page rendered at
<https://github.com/mnemosys-project> is `profile/README.md`.

## What is here

| Path | What it holds |
| --- | --- |
| [`NAMING.md`](NAMING.md) | The authoritative naming convention — the Muse scheme, the roster, and the rules |
| [`profile/`](profile) | The org profile README and its banner |
| [`epics/`](epics) | One directory per epic: `spec.md`, `plan.md`, `retrospective.md` |
| [`docs/epic-document-formats.md`](docs/epic-document-formats.md) | The house standard those three documents follow |
| [`docs/templates/`](docs/templates) | Templates for each of the three epic documents |
| [`docs/branding/`](docs/branding) | The banner design record, its prompts, and every generation |
| [`ISSUE_TEMPLATE/`](ISSUE_TEMPLATE), [`pull_request_template.md`](pull_request_template.md) | Org-wide issue and PR templates |

## Getting Started

Read [`CONTRIBUTING.md`](CONTRIBUTING.md) for how work is organized and how to
set up the tooling. Read [`NAMING.md`](NAMING.md) before proposing a name for
anything.

## License

MIT — see [LICENSE](LICENSE).
