# Security Policy

## Reporting a vulnerability

Report security vulnerabilities through
[GitHub's private vulnerability reporting](https://github.com/mnemosys-project/.github/security/advisories/new),
which keeps the report confidential.

If private reporting is unavailable, email **w.phillip.moore@gmail.com** with
the subject line "Mnemosys Security Report".

**Do not open a public issue for a security vulnerability.**

## Scope

| Component | Status |
| --- | --- |
| `.github` — org metadata, epics, community health files | In scope |
| `docs` — the organization site | In scope once created |
| `melete` — practice exercise generator | In scope once created |

`melete` is a local command-line tool that reads a configuration file and writes
PDFs. It has no network surface, no server, no database, and no authentication.
It embeds no model and calls no inference endpoint. The realistic security
surface is therefore small and mostly concerns input handling: a malformed
configuration file, a crafted session log, or the behaviour of the bundled
LilyPond binary when handed generated input.

Reports in those areas are welcome and will be taken seriously.

## Out of scope

- Vulnerabilities in upstream dependencies — report those to the upstream
  maintainer. The one runtime dependency is the PyPI `lilypond` redistribution,
  which packages the LilyPond binary.
- Vulnerabilities in GitHub, Docker, or other third-party platforms.
- Social engineering against contributors.

## Response commitment

- **Acknowledgment** — within 7 days.
- **Initial severity assessment** — within 14 days.
- **Fix or mitigation plan** — target 30 days from acknowledgment, depending on
  severity and complexity.

These timelines reflect the project's scale: it is a small personal project
maintained by one person. Response times may vary, but every report will be
acknowledged and investigated.

## Disclosure

We follow coordinated disclosure. Once a fix exists we will release it, publish
a GitHub security advisory, and credit the reporter unless anonymity is
requested. Please allow reasonable time for a fix before disclosing publicly.
