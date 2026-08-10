<!--
  Do not fill this in by hand.

  Pull requests in this organization are created by tooling, not written in
  the web UI. If you are seeing this template, you are probably creating a PR
  the wrong way.
-->

## Do not create this pull request manually

Pull requests here are submitted with **`vrg-submit-pr`**, which builds a
standards-compliant PR body, links it to its issue, and records the validation
evidence. A hand-written PR will be missing all of that.

```bash
vrg-submit-pr
```

Agents do not submit pull requests at all. An agent records readiness with
`vrg-pr-workflow report-ready` and a human runs `vrg-submit-pr`.

## If you are an outside contributor

You are welcome here, and you are not expected to install this tooling. Open
the PR from your fork with:

- **A link to the issue it addresses.** Every PR needs a primary issue, and the
  issue comes first — see
  [CONTRIBUTING.md](https://github.com/mnemosys-project/.github/blob/develop/CONTRIBUTING.md).
- **What changed and why**, in a sentence or two.
- **How you verified it** — the command you ran and what it printed.

A maintainer will handle the rest.
