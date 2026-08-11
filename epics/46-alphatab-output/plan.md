# alphaTab/`.gp` Output Pipeline — Implementation Plan

> **For agentic workers:** this plan is implemented through the Vergil
> epic/task framework — **each task below is a separate GitHub issue, filed
> under epic `mnemosys-project/.github#46`, implemented via `issue-implement`
> as its own PR.** Steps within a task use checkbox (`- [ ]`) syntax for
> tracking; the small commits inside a task use `vrg-commit`.

**Goal:** replace melete's LilyPond output with a pipeline that emits alphaTex
text and runs alphaTab to produce Guitar Pro `.gp` files, porting current v1
functionality with no change to the generation core.

**Architecture:** the Python core is unchanged. A new `Measure` model and a pure
`bar()` pass sit between the families and a new alphaTex **emitter**
(`Score`/measures → alphaTex text). A dumb, vendored Node tool (`melete-render/`)
runs alphaTab's `Gp7Exporter` to turn that text into a `.gp`. The Python↔alphaTab
interface is **alphaTex text over a subprocess** — the same blast-door shape as
today's LilyPond adapter. LilyPond stays in place and green until the final task.

**Tech Stack:** Python 3.14 (uv, pytest, ruff, mypy, ty); Node.js + alphaTab
(MPL-2.0, npm, `Gp7Exporter`); Guitar Pro 8 for the manual fidelity check.

## Global Constraints

Every task's requirements implicitly include these (verbatim from the spec and
repo standards):

- **Python 3.14**; **100% branch coverage**; `ruff`/`mypy`/`ty` clean.
- **Validation is only `vrg-container-run -- vrg-validate`** — never individual
  linters.
- **The interface is alphaTex text.** Python emits a string; the Node renderer
  consumes it and is a **black box** — Python tests assert the alphaTex, never
  the `.gp` bytes' internals beyond a structural parse.
- **The IR change is additive.** `Measure` is consumed only by the new emitter;
  new `Note` fields are default-valued; families keep producing the flat `Voice`;
  the nested-tuplet lift is permissive. It **must not touch `lilypond/emit.py`'s
  inputs** — `lilypond/` and `tests/lilypond/golden/` stay green, untouched,
  until Task 10.
- **No LilyPond material is deleted before Task 10.** Task 10 preserves one
  committed sample of v1 output and git-tags the pre-removal commit.
- **The renderer (`melete-render/`) is dumb-as-possible** — parse alphaTex,
  `Gp7Exporter`, write bytes; no musical logic; self-contained; extraction-ready.
- **Exact alphaTex syntax** for the emitter and barring is **pinned by Task 1's
  spike report** (`epics/46-alphatab-output/spike-findings.md`); Tasks 4–6
  consume those confirmed tokens rather than guessing.

## File Structure

- `melete-render/` (new, repo root) — vendored Node tool: `package.json`,
  `render.mjs`, `README.md`. One responsibility: alphaTex stdin → `.gp` stdout.
- `src/melete/score.py` (modify) — add `Measure`, add `Note.tied`, lift the
  nesting invariant, add `bar()`. IR types + pure voice functions.
- `src/melete/alphatab/__init__.py`, `emit.py`, `render.py` (new) — the alphaTex
  emitter and the Python blast door, paralleling `src/melete/lilypond/`.
- `src/melete/cli.py` (modify) — swap the generate pipeline's output half.
- `vergil.toml` (modify) — add the Node/alphaTab container capability (Task 2);
  remove the LilyPond capability (Task 10).
- `tests/alphatab/` (new) — `test_emit.py`, `test_render.py`, `golden/`.
- `tests/test_score.py` (modify) — `Measure`, `tied`, nested tuplets, `bar()`.
- `epics/46-alphatab-output/spike-findings.md` (new, in `.github`) — Task 1's
  recorded outcome.

---

### Task 1: Spike — validate alphaTex bar/tie semantics & `Gp7Exporter` fidelity

An **investigation task**, not TDD. Its deliverable is a written findings
document that pins the unknowns Tasks 3–6 depend on. It is filed as a
`validation`-kind operational task (its acceptance is a recorded outcome, not a
PR) OR a small docs PR carrying `spike-findings.md` — file it as a docs task so
the findings are committed under the epic.

**Files:**
- Create: `epics/46-alphatab-output/spike-findings.md` (in `.github`).
- Scratch: a throwaway `melete-render/` prototype + a hand-written `.atex`.

**Produces (consumed by Tasks 3–6):** confirmed answers to —
- Does alphaTab **auto-bar** a duration stream, or must the emitter place `|`?
- How is a **tie across a barline** expressed in alphaTex (token, placement)?
- The exact alphaTex for: a **6-string bass tuning**, a **string.fret** note, a
  **written duration** (`:8` etc.), a **tuplet** (and a **nested** tuplet), a
  **tempo**, **left-hand fingering**, an **accent**.
- The exact alphaTab **Node API** to parse alphaTex and run `Gp7Exporter`.
- Whether the exported `.gp` **opens cleanly in Guitar Pro 8** with the above
  intact.

- [ ] **Step 1: Stand up a throwaway prototype.** In a scratch dir, `npm init -y`,
  `npm install @coderline/alphatab` (confirm the package name from npm), and a
  minimal `render.mjs` that reads stdin, parses alphaTex, runs `Gp7Exporter`,
  writes stdout. (This is the Task 3 tool in draft; discard after.)
- [ ] **Step 2: Hand-write a representative `.atex`.** One track, 6-string bass
  tuning, a bar of quarter notes with string/fret and a fingering, a triplet, a
  nested triplet, and a note whose duration overflows a bar (to force a tie).
- [ ] **Step 3: Export and open.** Run it through the prototype, open the `.gp`
  in Guitar Pro 8. Record what rendered correctly and what did not.
- [ ] **Step 4: Answer the questions.** Determine auto-bar vs manual, the tie
  token, and every syntax element above — by experiment, not assumption.
- [ ] **Step 5: Write `spike-findings.md`.** Record each answer with the exact
  alphaTex snippet that produced it, and the confirmed Node API. Flag anything
  alphaTex/`Gp7Exporter` could **not** represent (a disqualifier check).
- [ ] **Step 6: Commit** the findings doc (`vrg-commit`, `Ref #<task-1-issue>`).

**Gate:** if the spike finds a disqualifier (a port feature alphaTex/`Gp7Exporter`
cannot carry), stop and escalate — the architecture assumption failed. Otherwise
Tasks 3–6 proceed using the recorded syntax.

---

### Task 2: Container capability — Node + alphaTab

**Files:**
- Modify: `vergil.toml` (add a Node/alphaTab container capability).
- Create: `melete-render/package.json` (pins alphaTab; installed at image build).

**Interfaces:**
- Produces: Node + the pinned alphaTab dependency present on `PATH`/in the
  container, so Task 3's `render.mjs` runs.

- [ ] **Step 1: Add the vendored package manifest.** Create
  `melete-render/package.json` pinning the alphaTab version confirmed in Task 1:

```json
{
  "name": "melete-render",
  "private": true,
  "type": "module",
  "version": "0.0.0",
  "description": "Dumb alphaTex -> .gp renderer over alphaTab. Vendored, extraction-bound (see README).",
  "dependencies": { "@coderline/alphatab": "<pinned-version-from-spike>" }
}
```

- [ ] **Step 2: Declare the container capability in `vergil.toml`.** Mirror the
  LilyPond system-package mechanism (issue #51) — add the Node runtime as a
  system package and arrange `npm ci` of `melete-render/` at image build. Use the
  container mechanism the tooling provides; do not hand-roll a Dockerfile if
  `[container]` covers it. Keep the existing `system-packages = ["lilypond"]`
  (removed only in Task 10).
- [ ] **Step 3: Cold-rebuild and verify.** `vrg-container-run -- bash -lc 'node
  --version && node -e "require.resolve(\"@coderline/alphatab\")"'` — Node and
  alphaTab resolve.
- [ ] **Step 4: Validate.** `vrg-container-run -- vrg-validate` green (no code
  changed; the container just gained a capability).
- [ ] **Step 5: Commit** (`vrg-commit`, `Ref #<task-2-issue>`).

A cold-rebuild **validation** task (operational, `--kind validation`) **is
filed**, blocked-by Tasks 2 and 8: on a cold image, Node + alphaTab present and
`generate` produces a valid `.gp` end-to-end. See spec §10.

---

### Task 3: The renderer (`melete-render/`)

**Files:**
- Create: `melete-render/render.mjs`, `melete-render/README.md`.

**Interfaces:**
- Consumes: alphaTex text on **stdin**.
- Produces: `.gp` bytes on **stdout**; non-zero exit + stderr on failure. This
  contract is frozen — the Python blast door (Task 7) and any future standalone
  package depend on exactly it.

- [ ] **Step 1: Write `render.mjs` — dumb and complete.** Using the API confirmed
  in Task 1 (illustrative shape):

```js
// melete-render: alphaTex (stdin) -> Guitar Pro .gp (stdout). No musical logic.
import * as alphaTab from '@coderline/alphatab';

const chunks = [];
for await (const c of process.stdin) chunks.push(c);
const alphaTex = Buffer.concat(chunks).toString('utf8');

try {
  const score = alphaTab.importer.ScoreLoader.loadScoreFromBytes(
    Buffer.from(alphaTex, 'utf8'), // or the AlphaTexImporter path confirmed in Task 1
  );
  const data = new alphaTab.exporter.Gp7Exporter().export(score, null);
  process.stdout.write(Buffer.from(data));
} catch (err) {
  process.stderr.write(String(err?.stack ?? err) + '\n');
  process.exit(1);
}
```

- [ ] **Step 2: Write the README** stating it is acknowledged, extraction-bound
  technical debt: dumb by design, driven only over stdin/stdout, destined for its
  own TypeScript-engineered repo once Vergil supports TypeScript. The extraction
  **disposition** (defer / follow-on epic / drop) is owned by the follow-on
  brainstorm bookend, melete#82. No tests here (black box; the interface is
  exercised from Python in Task 9).
- [ ] **Step 3: Smoke it by hand.** Pipe the Task 1 `.atex` in, confirm a `.gp`
  comes out and opens in Guitar Pro 8.
- [ ] **Step 4: Validate + commit** (`vrg-validate` unaffected; `vrg-commit`,
  `Ref #<task-3-issue>`).

---

### Task 4: IR evolution (additive) — `Measure`, `Note.tied`, lift nesting

**Files:**
- Modify: `src/melete/score.py`.
- Test: `tests/test_score.py`.

**Interfaces:**
- Produces (consumed by Tasks 5–6):
  - `Note.tied: bool = False` — "this note is tied to the following note."
  - `Measure(voice: Voice)` frozen dataclass — one bar's flat voice.
  - `Tuplet` may now contain `Note` **or** `Tuplet` (nesting permitted).
- Additive guarantee: `Voice`, `emit`/`render` under `lilypond/`, and the golden
  tests are untouched.

- [ ] **Step 1: Failing test — `tied` defaults False and round-trips.**

```python
def test_note_tied_defaults_false_and_can_be_set():
    n = Note(pitch=60, string=0, fret=0, duration=Fraction(1, 4), finger=None, accent=False)
    assert n.tied is False
    assert n.__class__(**{**n.__dict__, "tied": True}).tied is True
```

- [ ] **Step 2: Run — fails** (`TypeError`/`AttributeError`; no `tied`).
  `pytest tests/test_score.py::test_note_tied_defaults_false_and_can_be_set -v`
- [ ] **Step 3: Add `tied: bool = False`** as the last field of `Note`
  (default-valued → additive; existing constructors unaffected).
- [ ] **Step 4: Run — passes.**
- [ ] **Step 5: Failing test — nested tuplets are allowed.**

```python
def test_tuplet_may_contain_a_tuplet():
    inner = Tuplet(ratio=(3, 2), notes=[_note(), _note(), _note()])
    outer = Tuplet(ratio=(3, 2), notes=[inner, _note(), _note()])  # nested
    assert outer.notes[0] is inner
```

  (`_note()` is a local factory returning a valid `Note`.)
- [ ] **Step 6: Run — fails** (current `_first_foreign(self.notes, (Note,))`
  rejects a `Tuplet`).
- [ ] **Step 7: Lift the invariant.** Allow `Tuplet.notes` to hold `Note | Tuplet`;
  update the `__post_init__` foreign-check to `(Note, Tuplet)`; update the module
  docstring's "one level of nesting" paragraph to record it as
  LilyPond-imposed and lifted (ref epic #1 §4 correction). **Do not** touch
  `lilypond/emit.py` — the port's families still emit no nested tuplets, so its
  one-level walk never meets one.
- [ ] **Step 8: Run — passes;** run the full `tests/lilypond/` golden suite to
  confirm it is still green (additive proof).
- [ ] **Step 9: Failing test — `Measure` wraps a voice.**

```python
def test_measure_holds_a_voice():
    m = Measure(voice=[_note(), _note()])
    assert list(notes(m.voice)) == m.voice
```

- [ ] **Step 10: Run — fails** (no `Measure`).
- [ ] **Step 11: Add the frozen `Measure` dataclass** (field `voice: Voice`,
  same shallow-frozen stance as `Score`).
- [ ] **Step 12: Run — passes. Coverage 100%. Commit** (`vrg-commit`,
  `Ref #<task-4-issue>`).

---

### Task 5: The barring pass

**Files:**
- Modify: `src/melete/score.py` (add `bar()` beside `notes()`/`sounding_duration()`).
- Test: `tests/test_score.py`.

**Interfaces:**
- Consumes: `Voice`, `time_signature: tuple[int, int]`, and the confirmed answer
  from Task 1 (auto-bar vs manual). **If Task 1 finds alphaTab auto-bars and
  auto-ties, this task shrinks to a no-op/validation and the emitter (Task 6)
  emits a flat stream** — record that and skip to Task 6.
- Produces (consumed by Task 6): `bar(voice, time_signature) -> list[Measure]`.

- [ ] **Step 1: Failing test — a whole-bar voice yields one measure, untied.**

```python
def test_bar_exact_fit_is_one_measure():
    voice = [_q(), _q(), _q(), _q()]                    # four quarters in 4/4
    measures = bar(voice, (4, 4))
    assert len(measures) == 1
    assert all(n.tied is False for n in measures[0].voice)
```

- [ ] **Step 2: Run — fails** (`bar` undefined).
- [ ] **Step 3: Implement `bar()` — accumulate sounding time; close a `Measure`
  at each barline.** Use `sounding_duration` semantics (a tuplet's sounding time
  is scaled). No note is split yet.
- [ ] **Step 4: Run — passes.**
- [ ] **Step 5: Failing test — a note crossing a barline is split and tied.**

```python
def test_bar_splits_and_ties_across_the_barline():
    # a half note starting on beat 4 of 4/4 overflows into the next bar
    voice = [_q(), _q(), _q(), _half()]
    measures = bar(voice, (4, 4))
    assert len(measures) == 2
    tail_of_bar1 = measures[0].voice[-1]
    head_of_bar2 = measures[1].voice[0]
    assert tail_of_bar1.duration == Fraction(1, 4)   # remainder of bar 1
    assert tail_of_bar1.tied is True                 # tied into bar 2
    assert head_of_bar2.duration == Fraction(1, 4)   # continuation
    assert head_of_bar2.pitch == tail_of_bar1.pitch  # same note
```

- [ ] **Step 6: Run — fails.**
- [ ] **Step 7: Implement split-and-tie:** when a note's sounding time overflows
  the current bar, emit the in-bar remainder with `tied=True`, close the bar,
  and continue with the remaining duration in the next bar(s). The split
  durations must be individually writable (`duration_token`-representable); where
  a remainder is not a single writable value, split it into writable pieces all
  tied. Preserve `string`/`fret`/`finger`; `accent` only on the first piece.
- [ ] **Step 8: Run — passes.**
- [ ] **Step 9: Edge-case tests** — short final measure (spec §7: the last
  measure may be under-full and is **not** padded), a tuplet that fits, and a
  nested tuplet passes through untouched. Real assertions, no placeholders.
- [ ] **Step 10: Run all; coverage 100%. Commit** (`vrg-commit`,
  `Ref #<task-5-issue>`).

---

### Task 6: The alphaTex emitter

**Files:**
- Create: `src/melete/alphatab/__init__.py`, `src/melete/alphatab/emit.py`.
- Test: `tests/alphatab/test_emit.py`, `tests/alphatab/golden/`.

**Interfaces:**
- Consumes: `Score`, `list[Measure]` from `bar()`, and the **exact alphaTex
  tokens from Task 1's `spike-findings.md`** (tuning, string.fret, duration,
  tuplet, nested tuplet, tie, tempo, fingering, accent).
- Produces (consumed by Tasks 7–8):
  - `emit_score(score: Score) -> str` — one exercise as alphaTex.
  - `emit_book(scores: Sequence[Score], cover: Cover) -> str` — a day's session.

- [ ] **Step 1: Failing test — a single note emits the spike-confirmed
  string.fret + duration token.** Assert the *structure* against the tokens Task
  1 recorded (not a guessed syntax): e.g. that the emitted text contains the
  tuning header, the string and fret in the confirmed order, and the duration
  token. Use a golden file under `tests/alphatab/golden/` for a whole exercise.

```python
def test_single_note_emits_string_fret_and_duration():
    score = _one_note_score()               # C on a known string/fret, quarter
    text = emit.emit_score(score)
    assert TUNING_HEADER in text            # constants sourced from spike-findings
    assert f"{STRING_TOKEN}.{FRET_TOKEN}" in text
    assert DURATION_TOKEN_QUARTER in text
```

- [ ] **Step 2: Run — fails** (module/functions absent).
- [ ] **Step 3: Implement `emit.py` as a pure pass-through**, structured like
  `lilypond/emit.py`: per-note token, per-measure join with the bar separator,
  tuplet wrapping, the tie token on `Note.tied`, the tuning/tempo/fingering
  headers — all using the spike-confirmed alphaTex. No filesystem, no subprocess.
  Reuse `theory.spell`/`SpelledPitch` exactly as `lilypond/emit.py` does (the
  spelling model is renderer-neutral and survives — epic #1 §4).
- [ ] **Step 4: Run — passes.**
- [ ] **Step 5: Golden test for a full exercise and a full book** (cover +
  several exercises), including a tuplet, a nested tuplet, and a tied
  cross-barline note. Commit the golden `.atex` files. These goldens are the
  content-correctness gate (spec §1 acceptance).
- [ ] **Step 6: Run all; coverage 100%. Commit** (`vrg-commit`,
  `Ref #<task-6-issue>`).

---

### Task 7: The Python blast door (`alphatab/render.py`)

**Files:**
- Create: `src/melete/alphatab/render.py`.
- Test: `tests/alphatab/test_render.py`.

**Interfaces:**
- Consumes: alphaTex `str`, an output dir, `stem`.
- Produces: `render(alphatex: str, out_dir: Path, *, stem: str) -> Path`
  returning the written `.gp` path; raises `RenderError` on failure. Mirrors
  `lilypond/render.py`'s contract.

- [ ] **Step 1: Failing test — a missing Node names the resolution.** Use the
  fake-binary-on-`PATH` technique from `tests/lilypond/test_render.py`: point
  `PATH` at an empty dir and assert `RenderError` mentions Node and how to
  install it (no stack trace).

```python
def test_missing_node_states_the_resolution(tmp_path, monkeypatch):
    monkeypatch.setenv("PATH", str(tmp_path / "nothing"))
    with pytest.raises(RenderError) as exc:
        render("...", tmp_path, stem="practice")
    assert "node" in str(exc.value).lower()
```

- [ ] **Step 2: Run — fails.**
- [ ] **Step 3: Implement `render.py`** paralleling `lilypond/render.py`: resolve
  `node` via `shutil.which` (explicit `RenderError` with a `MISSING_NODE`
  resolution message if absent); write the alphaTex beside the output (kept on
  disk, as the `.ly` is); `subprocess.run(["node", RENDER_MJS], input=alphatex,
  capture_output=True)`; on non-zero exit raise `RenderError` with stderr
  **verbatim** and the kept-alphaTex path; write stdout bytes to
  `out_dir/<stem>.gp`; return it. No silent fallback.
- [ ] **Step 4: Failing test — a failing renderer keeps the alphaTex and surfaces
  stderr verbatim.** Use a fake `node` on `PATH` that writes a known message to
  stderr and exits 1 (the same pattern as the LilyPond adapter's fake).
- [ ] **Step 5: Implement to pass; add the success-path test** with a fake `node`
  that writes known bytes to stdout, asserting the `.gp` is written and returned.
- [ ] **Step 6: Run all; coverage 100%. Commit** (`vrg-commit`,
  `Ref #<task-7-issue>`).

---

### Task 8: Wire CLI/session to the new pipeline

**Files:**
- Modify: `src/melete/cli.py` (the `generate` output half, lines ~365–377).
- Test: `tests/test_cli_generate.py`.

**Interfaces:**
- Consumes: `alphatab.emit.emit_book`/`emit_score`, `alphatab.render.render`,
  `score.bar`.
- Produces: a `generate` run that writes a day's `.gp` + the kept alphaTex + the
  unchanged session log.

- [ ] **Step 1: Failing test — `generate` writes a `.gp` (fake `node`).** Extend
  `tests/test_cli_generate.py` with a fake `node` on `PATH` (as in Task 7);
  assert the session dir gets `practice.gp` and the kept `.atex`, and the session
  log is unchanged.
- [ ] **Step 2: Run — fails** (CLI still calls LilyPond).
- [ ] **Step 3: Swap the pipeline.** Replace `from melete.lilypond import emit,
  render` with `from melete.alphatab import emit, render` in the generate path;
  run `bar()` per score before emit; write `.gp` instead of `.pdf`; keep the
  `--split` behaviour (one `.gp` per exercise) analogous to today. **Leave the
  LilyPond modules in place** (imported by nothing in the generate path now, but
  not deleted until Task 10).
- [ ] **Step 4: Run — passes.**
- [ ] **Step 5: Update the `staves`/PDF-specific CLI copy** only as needed so the
  command's help and outputs describe `.gp`. Do not remove LilyPond config keys
  yet.
- [ ] **Step 6: Run all; coverage 100%. Commit** (`vrg-commit`,
  `Ref #<task-8-issue>`).

---

### Task 9: Black-box integration test (Node-gated)

**Files:**
- Create/extend: `tests/alphatab/test_render.py` (the real-renderer test).

**Interfaces:**
- Consumes: the real `melete-render/` tool + Node (present in the container).
- Produces: proof that real alphaTex → the real renderer → a **valid `.gp`**.

- [ ] **Step 1: Write the integration test**, marked `@pytest.mark.integration`
  and `@pytest.mark.skipif(shutil.which("node") is None, reason=...)` — the exact
  discipline `tests/lilypond/test_render.py` used for `lilypond`. Feed a known
  alphaTex through `render()`, then assert the bytes are a valid `.gp`: parse them
  back (via a second tiny Node call, or a structural byte check confirmed in Task
  1) and assert expected **track and bar counts**. Do **not** assert `.gp`
  internals beyond structure — the content gate is Task 6's alphaTex goldens.
- [ ] **Step 2: Run in-container** (`vrg-container-run -- vrg-validate`) — the
  test runs (Node present) and passes.
- [ ] **Step 3: Keep `integration-tests = false`** in `vergil.toml` until the
  status-context/enablement decision is made separately (same discipline as the
  LilyPond integration test). Commit (`vrg-commit`, `Ref #<task-9-issue>`).

---

### Task 10: Remove LilyPond (final)

Runs **only after Tasks 1–9 are merged and the `.gp` pipeline is green.**

**Files:**
- Delete: `src/melete/lilypond/`, `tests/lilypond/`.
- Modify: `vergil.toml` (drop `system-packages = ["lilypond"]` and the LilyPond
  `integration-tests` framing), `docs/design.md`, `CLAUDE.md` (melete#83).
- Create: `docs/reports/samples/` — one committed v1 sample (`.ly` + rendered
  `.pdf`).

- [ ] **Step 1: Preserve the comparison baseline.** Before deleting anything,
  generate one v1 day-session with LilyPond, and commit the resulting `.ly` +
  `.pdf` under `docs/reports/samples/` with a short README noting the seed/config.
- [ ] **Step 2: Tag the pre-removal commit.** `vrg-git tag` an annotated tag
  (e.g. `lilypond-final`) at the commit where LilyPond last works, so the code is
  recoverable. (If tagging is human-gated in this tooling, record the request in
  the task for the human to run.)
- [ ] **Step 3: Delete the LilyPond surface.** Remove `src/melete/lilypond/` and
  `tests/lilypond/` (including `golden/`). Confirm nothing imports them
  (`grep -rn "melete.lilypond\|lilypond" src/ tests/`).
- [ ] **Step 4: Remove the container capability.** Drop
  `system-packages = ["lilypond"]` from `vergil.toml`; keep Node/alphaTab.
- [ ] **Step 5: Sync `docs/design.md`.** Update its renderer-boundary and
  "engraves through LilyPond" text to the alphaTab reality. **`CLAUDE.md`
  de-pollution is NOT done here** — it is owned by melete#83 (its own PR, run
  late in the epic so it reflects the final reality).
- [ ] **Step 6: Validate** (`vrg-container-run -- vrg-validate`) green with no
  LilyPond present; coverage 100%. **Commit** (`vrg-commit`, `Ref #<task-10-issue>`).

---

## Self-Review

**Spec coverage** — every spec section maps to a task: §3 strategic frame →
Global Constraints; §4 renderer-boundary corrections → Task 4 (nesting) + §6/Task
5 (measures); §6 IR evolution (Measure, `tied`, additive) → Task 4; barring pass
→ Task 5; §7 renderer → Tasks 2–3, 7; §8 removal → Task 10 (+ sample/tag); §9
testing → Tasks 4–9 (Python goldens + black-box + smoke); §10 tasks → Tasks
1–10; §11 open questions → Task 1 spike. Bookends (#47/#48/#81/#82/#83) are
tracked outside this plan.

**Placeholder scan** — the only deferred specifics are the exact alphaTex tokens
and the alphaTab Node API, which are **genuine dependencies on Task 1's recorded
findings**, not lazy placeholders; every code task states real files, signatures,
and test code. Task 1 is explicitly an investigation whose output unblocks the
rest.

**Type consistency** — `Note.tied: bool`, `Measure(voice: Voice)`,
`bar(voice, time_signature) -> list[Measure]`, `emit_score(score) -> str`,
`emit_book(scores, cover) -> str`, `render(alphatex, out_dir, *, stem) -> Path`
are used consistently across Tasks 4–9.

**Contingency** — if Task 1 finds alphaTab auto-bars *and* auto-ties, Task 5
collapses to validation and Task 6 emits a flat stream; the plan says so at Task
5. If Task 1 finds a disqualifier, the epic stops at the gate.
