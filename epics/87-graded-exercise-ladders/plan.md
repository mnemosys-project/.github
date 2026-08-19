# Graded Exercise Ladders Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Turn each session slot from one maximally-hard exercise into a graded
ladder — the bare shape, then the same shape accreting one technique per rung,
topped by a challenge rung the player is not expected to manage.

**Architecture:** An exercise's axes split into *identity* (fixed for the whole
ladder) and *deviations* (each with a plain value, switched on one per rung).
A ladder is the monotone chain from all-plain to the top. Every rung
materializes to an ordinary `ExerciseSpec`, so the ladder is a **selection-time
concept only** — `pipeline`, `layout`, `rhythm`, `score`, the families'
`generate` functions and the emitter's per-exercise engraving are untouched.

**Tech Stack:** Python 3.13, `uv`, pytest, ruff, mypy. Rendering through the
vendored `melete-render` Node tool (not touched by this epic).

**Spec:** [`spec.md`](spec.md) — decisions L1–L15 are referenced by number
throughout; read it alongside this plan.

## Global Constraints

- **Repository:** `mnemosys-project/melete`. All work happens in a worktree
  under `.worktrees/issue-<N>-<slug>/` on branch `feature/<N>-<slug>`. The main
  worktree is read-only.
- **Shell wrappers:** `vrg-git` instead of `git`, `vrg-gh` instead of `gh`,
  `vrg-commit` instead of `git commit`. Raw `git`/`gh` are denied.
- **Validation is one command:** `vrg-container-run -- vrg-validate`, run from
  inside the worktree. Do not invoke ruff, mypy or pytest directly as a
  substitute for it; do use `pytest <path> -v` for the fast inner TDD loop.
- **The renderer boundary holds.** Only `alphatab/emit.py`, `alphatab/render.py`,
  `melete-render/` and `tests/alphatab/golden/` may know a renderer exists.
  `ladder.py` must not import from `melete.alphatab`.
- **No silent failures.** Every refusal names the key, the family or the file
  and quotes the accepted values. Never default an unconfigured key, never
  clamp a value into range, never shorten a ladder to make it fit.
- **Docstrings carry the argument, not just the behaviour.** This codebase
  documents *why* a decision was taken and what failure it prevents; match that
  register. Cite decisions as `L<N>` and issues as `melete#<N>`.
- **Commit style:** `vrg-commit --type <type> --scope <scope> --message "…"`,
  conventional-commit types only.

---

### Task 1: `derive` fills only what is unset

Decision **L11**. Today `scales.derive` and `arpeggios.derive` decide `hands`
outright from the drawn `scale_type`/`quality`, and `selection._sample` applies
the result last-write-wins (`selection.py:440`). Under a ladder that would
overwrite rung 1's plain `hands`, tapping every rung of a tapped-eligible ladder
while the sheet still renders and still passes goldens.

**Files:**
- Modify: `src/melete/families/scales.py:229-265` (`derive`)
- Modify: `src/melete/families/arpeggios.py:226-268` (`derive`)
- Modify: `src/melete/selection.py:440` (the final `params.update(...)`)
- Test: `tests/families/test_scales.py`, `tests/families/test_arpeggios_tapping.py`

**Interfaces:**
- Consumes: nothing from earlier tasks.
- Produces: `derive(params: Params, tapped: frozenset[str] = frozenset()) ->
  dict[str, object]` — unchanged signature, new contract: **an axis already
  present in `params` is never returned**, and the coupled pins follow the
  effective `hands` rather than the drawn quality/scale type.

- [ ] **Step 1: Write the failing tests**

```python
# tests/families/test_scales.py

def test_derive_respects_an_explicitly_set_hands() -> None:
    """A rung that has already decided `hands` is not overruled (L11).

    The defect this pins: `derive` used to key `hands` off the scale type
    alone, so rung 1 of a tapped-eligible ladder — which sets `hands` to its
    plain value 1 — came back two-handed. The sheet renders, the rungs differ
    in pattern and rhythm, and only playing it reveals the plain scale was
    never engraved.
    """
    tapped = frozenset({"ionian"})

    derived = scales.derive({"scale_type": "ionian", "hands": 1}, tapped)

    assert "hands" not in derived
    assert "traversal" not in derived


def test_derive_pins_traversal_from_the_effective_hands() -> None:
    """The 3nps pin follows `hands == 2`, however that value was arrived at."""
    derived = scales.derive({"scale_type": "dorian", "hands": 2}, frozenset())

    assert derived["traversal"] == "three_note_per_string"


def test_derive_still_couples_an_undecided_hands_to_the_scale_type() -> None:
    """The pre-ladder behaviour is unchanged when nothing has decided `hands`."""
    tapped = frozenset({"ionian"})

    derived = scales.derive({"scale_type": "ionian"}, tapped)

    assert derived == {"hands": 2, "traversal": "three_note_per_string"}
```

```python
# tests/families/test_arpeggios_tapping.py

def test_derive_respects_an_explicitly_set_hands() -> None:
    """A rung that has decided `hands` keeps it, and keeps its inversion (L11)."""
    derived = arpeggios.derive({"quality": "min7", "hands": 1}, frozenset({"min7"}))

    assert "hands" not in derived
    assert "inversion" not in derived


def test_derive_pins_inversion_from_the_effective_hands() -> None:
    """The root-position pin follows `hands == 2`, not the quality."""
    derived = arpeggios.derive({"quality": "min7", "hands": 2}, frozenset())

    assert derived["inversion"] == "root"
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `pytest tests/families/test_scales.py -k derive -v` and
`pytest tests/families/test_arpeggios_tapping.py -k derive -v`

Expected: FAIL — the explicit-`hands` tests fail with `derived == {"hands": 2,
"traversal": "three_note_per_string"}`, because the current hooks ignore
`params["hands"]` entirely.

- [ ] **Step 3: Rewrite `scales.derive`**

```python
def derive(
    params: Mapping[str, object],
    tapped_scale_types: frozenset[str] = frozenset(),
) -> dict[str, object]:
    """The axes that *follow* from what has already been decided (§7, decision 8, L11).

    The contract is fill-what-is-unset, never overrule. Before graded ladders
    (`melete#87`) this hook decided `hands` from the drawn `scale_type` alone and
    the selector applied it last-write-wins, which was correct while every
    exercise was a single draw and silently wrong the moment a *rung* decides its
    own hand count: rung 1 sets `hands` to its plain 1, and the old hook handed
    back 2 for any tapped-eligible scale type. Every rung of the ladder tapped,
    the book rendered, the goldens passed, and only playing the sheet revealed
    that the bare scale the ladder exists to start from was never engraved.

    So an axis already present in `params` is returned untouched — absent from
    the mapping — and the coupled pin follows the *effective* hand count rather
    than the scale type:

    * `hands` — derived only when nothing has decided it: `2` for a scale type in
      `tapped_scale_types`, `1` otherwise, exactly as before.
    * `traversal` — pinned to `three_note_per_string` whenever the effective
      hands is `2`, because H1 realizes the two-hand tap only for the 3nps
      journey (`_positional_tapped_is_deferred`, corpus R9). An untapped scale
      leaves its traversal to be decided elsewhere.

    Called incrementally by the selector as it samples, so `scale_type` may not
    have been drawn yet; until it has, there is nothing to derive.
    """
    scale_type = params.get("scale_type")
    if scale_type is None:
        return {}

    derived: dict[str, object] = {}
    hands = params.get(HANDS)
    if hands is None:
        hands = _TWO_HANDS if cast("str", scale_type) in tapped_scale_types else _ONE_HAND
        derived[HANDS] = hands

    if hands == _TWO_HANDS and "traversal" not in params:
        derived["traversal"] = _THREE_NOTE_PER_STRING
    return derived
```

- [ ] **Step 4: Rewrite `arpeggios.derive` the same way**

```python
def derive(
    params: Mapping[str, object],
    tapped_qualities: frozenset[str] = frozenset(),
) -> dict[str, object]:
    """The axes that *follow* from what has already been decided (§7, decision 8, L11).

    See `scales.derive` for the argument: fill what is unset, never overrule, so
    a ladder rung that has decided its own hand count keeps it.

    * `hands` — derived only when nothing has decided it: `2` for a triad or a
      seventh the pool opted into tapping, `1` otherwise. (That triads are forced
      two-handed is a defect in its own right — `melete#227` — and is not this
      hook's to fix; when it lands, this branch is where it lands.)
    * `inversion` — pinned to root whenever the effective hands is `2`, because
      the captured tap boxes are root-position shapes (spec §2) and a non-root
      tap raises. It follows the hand count, not the quality.
    """
    quality = params.get("quality")
    if quality is None:
        return {}

    drawn = cast("str", quality)
    derived: dict[str, object] = {}
    hands = params.get(HANDS)
    if hands is None:
        hands = _TWO_HANDS if _is_triad(drawn) or drawn in tapped_qualities else _ONE_HAND
        derived[HANDS] = hands

    if hands == _TWO_HANDS and "inversion" not in params:
        derived["inversion"] = INVERSIONS[0]
    return derived
```

- [ ] **Step 5: Make the selector's final derive fill-only**

In `selection._sample`, replace the last-write-wins update at `selection.py:440`:

```python
    # was: params.update(cast("Mapping[str, AxisValue]", derive(params, tapped)))
    for axis, value in derive(params, tapped).items():
        params.setdefault(axis, cast("AxisValue", value))
```

Add to `_sample`'s docstring, after the existing paragraph about the hook:

```
    The hook fills only what is unset (L11), so this final pass adds the axes the
    family derives that are not in its sampled list without overruling anything
    already decided. `setdefault` rather than `update` is the whole of that rule.
```

- [ ] **Step 6: Run the full family and selection suites**

Run: `pytest tests/families/ tests/test_selection.py -v`

Expected: PASS. The pre-ladder behaviour is unchanged — nothing in the existing
pipeline sets `hands` before `derive` runs, so every existing draw takes the
`hands is None` branch.

- [ ] **Step 7: Validate and commit**

```bash
vrg-container-run -- vrg-validate
vrg-git add src/melete/families/scales.py src/melete/families/arpeggios.py \
    src/melete/selection.py tests/families/test_scales.py \
    tests/families/test_arpeggios_tapping.py
vrg-commit --type refactor --scope families \
  --message "derive fills only what is unset (#87)" \
  --body "The hook decided hands from the drawn scale type or quality and the
selector applied it last-write-wins, which is correct for a single draw and
silently wrong once a ladder rung decides its own hand count: rung 1 sets hands
to its plain 1 and the old hook handed back 2, tapping every rung while the
book still rendered and the goldens still passed.

An axis already present in params is now returned untouched, and the traversal
and inversion pins follow the effective hand count rather than the drawn value.
Pre-ladder behaviour is unchanged — nothing sets hands before derive runs.

Ref: mnemosys-project/.github#87"
```

---

### Task 2: `hands` becomes a real axis

Decision **L13**: every deviation axis is an `AXES` member, because `config`
validates pool keys against `REGISTRY[family].axes` (`config.py:375`) and an
axis outside that tuple has no candidate values for the draw to read and no
valid `[challenge.<family>]` entry.

This is safe to land before the ladder exists: `_sample` consults `derive`
*before* drawing each axis (`selection.py:435-437`), so an undecided `hands`
is filled from the hook and the pool is never consulted. The pool entry becomes
load-bearing only in Task 7.

**Files:**
- Modify: `src/melete/families/scales.py:134-139` (`AXES`)
- Modify: `src/melete/families/arpeggios.py:142-147` (`AXES`)
- Modify: `src/melete/config.py:360-400` (`_AXES_BY_FAMILY`, a new `_HANDS` axis)
- Modify: `examples/config.toml`
- Test: `tests/families/test_registry.py`, `tests/test_config.py`

**Interfaces:**
- Consumes: Task 1's `derive` contract.
- Produces: `"hands"` present in `scales.AXES` and `arpeggios.AXES`;
  `[pool.scales] hands` and `[pool.arpeggios] hands` accepted by `config`,
  values constrained to `(1, 2)`.

- [ ] **Step 1: Write the failing tests**

```python
# tests/families/test_registry.py

def test_hands_is_an_axis_of_every_family_that_taps() -> None:
    """A deviation must be an AXES member to have a pool to draw from (L13).

    `config` validates `[pool.<family>]` keys against `REGISTRY[family].axes`,
    so an axis outside that tuple has no configuration surface at all — the
    ladder's value draw would have nothing to read and `[challenge.<family>]`
    would reject the key.
    """
    assert "hands" in REGISTRY["scales"].axes
    assert "hands" in REGISTRY["arpeggios"].axes
```

```python
# tests/test_config.py

def test_hands_pool_accepts_one_and_two() -> None:
    active = load_string(
        HEADER
        + """
        [pool.scales]
        roots = "all"
        scale_types = ["ionian"]
        traversals = ["three_note_per_string"]
        patterns = ["straight"]
        hands = [1, 2]
        """
    )

    assert active.pool["scales"].values["hands"] == (1, 2)


def test_hands_pool_rejects_a_third_hand() -> None:
    with pytest.raises(ConfigError, match=r"pool\.scales\.hands"):
        load_string(
            HEADER
            + """
            [pool.scales]
            roots = "all"
            scale_types = ["ionian"]
            traversals = ["positional"]
            patterns = ["straight"]
            hands = [1, 2, 3]
            """
        )
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `pytest tests/families/test_registry.py -k hands tests/test_config.py -k hands -v`

Expected: FAIL — `"hands" in REGISTRY["scales"].axes` is `False`, and the config
tests fail with an unknown-key error naming `hands`.

- [ ] **Step 3: Add `hands` to both `AXES` tuples**

```python
# src/melete/families/scales.py
AXES = (
    "root",
    "scale_type",
    "traversal",
    "pattern",
    "hands",
)
```

```python
# src/melete/families/arpeggios.py
AXES = (
    "root",
    "quality",
    "inversion",
    "pattern",
    "hands",
)
```

`hands` goes last in both, after the axes `derive` keys on, because `_sample`
walks `AXES` in order and the hook can only decide `hands` once `scale_type` or
`quality` has been drawn.

- [ ] **Step 4: Add the config surface**

In `src/melete/config.py`, beside the other `_registry` axes:

```python
#: `hands` is a two-valued axis rather than a vocabulary one — one hand or two —
#: so it is bounded here rather than against `vocabulary.AXES`. It became a
#: sampled axis with graded ladders (`melete#87`, L13): a ladder switches tapping
#: on at a rung, so the value needs a pool to be drawn from and a
#: `[challenge.<family>]` entry to be valid. `tapped_scale_types` and
#: `tapped_qualities` did not become that pool — they stay what they read as,
#: the *precondition* on offering the deviation at all (spec §4).
_HANDS = _bounded("hands", "hands", allowed=(1, 2))
```

Add `_HANDS` to the `scales` and `arpeggios` entries of `_AXES_BY_FAMILY`:

```python
    "scales": (_ROOT, _SCALE_TYPE, _TRAVERSAL, _PATTERN, _HANDS),
    "arpeggios": (
        _ROOT,
        _QUALITY,
        _INVERSION,
        _PATTERN,
        _HANDS,
    ),
```

If no `_bounded` helper exists, write one modelled on `_registry`: same
signature shape, but validating membership in an explicit tuple of integers and
naming the accepted values in its error, per the no-silent-failures constraint.

- [ ] **Step 5: Run the tests to verify they pass**

Run: `pytest tests/families/test_registry.py tests/test_config.py -v`

Expected: PASS.

- [ ] **Step 6: Confirm the existing draw is untouched**

Run: `pytest tests/test_selection.py tests/test_cli_generate.py -v`

Expected: PASS **without** adding `hands` to any test configuration — this is
the point of Step 3's ordering. If a test fails with "configures no candidate
values for the 'hands' axis", `derive` is not being consulted before the draw;
re-check Task 1 Step 5 before adding pool entries to silence it.

- [ ] **Step 7: Add `hands` to the worked example**

In `examples/config.toml`, under `[pool.scales]` and `[pool.arpeggios]`:

```toml
hands = [1, 2]                       # one-hand or two-hand tapped (ladder deviation)
```

- [ ] **Step 8: Validate and commit**

```bash
vrg-container-run -- vrg-validate
vrg-git add src/melete/families/scales.py src/melete/families/arpeggios.py \
    src/melete/config.py examples/config.toml \
    tests/families/test_registry.py tests/test_config.py
vrg-commit --type feat --scope config \
  --message "make hands a configurable axis (#87)" \
  --body "A deviation axis must be an AXES member to have a pool: config
validates [pool.<family>] keys against REGISTRY[family].axes, so an axis outside
that tuple has no candidate values for a ladder to draw and no valid
[challenge.<family>] entry.

hands joins scales.AXES and arpeggios.AXES last, after the axes derive keys on.
The existing draw is unaffected because _sample consults derive before drawing
each axis, so an undecided hands is filled from the hook and the pool is never
read. tapped_scale_types and tapped_qualities stay what they read as — the
precondition on offering the deviation, not its pool.

Ref: mnemosys-project/.github#87"
```

---

### Task 3: The `Ladder` record and the four family declarations

Spec §4. `Ladder` joins `Family` as a sibling record holding *references* to
each family's own declarations — never copies — so adding a fifth family stays
one line of `REGISTRY`.

**Files:**
- Modify: `src/melete/families/__init__.py` (the `Ladder` record, `REGISTRY`)
- Modify: `src/melete/families/scales.py`, `arpeggios.py`, `intervals.py`,
  `chromatic.py` (four declaration blocks)
- Test: `tests/families/test_registry.py`

**Interfaces:**
- Consumes: Task 2's `AXES` tuples.
- Produces:
  - `families.Ladder(identity, plain, tiers, eligible)`
  - `families.REGISTRY[name].ladder -> Ladder`
  - `Identity = Callable[[Params], frozenset[str]]` — drawn identity → the axes
    that are identity for it (decision **L12**)
  - `Eligibility = Callable[[Params, frozenset[str], frozenset[str]], frozenset[str]]`
    — `(identity_params, tapped_values, already_chosen) -> offerable axes`

- [ ] **Step 1: Write the failing tests**

```python
# tests/families/test_registry.py

def test_every_family_declares_a_ladder() -> None:
    for name, family in REGISTRY.items():
        assert family.ladder is not None, f"{name} has no ladder declaration"


def test_identity_and_deviations_partition_the_axes() -> None:
    """Every axis a family reads is identity or deviation — never neither (L12).

    The defect this pins: `intervals.scale_type` belonged to neither column in
    the first draft of the spec, so a `context = "diatonic"` ladder would have
    materialized params with no scale type and raised on every rung.
    """
    for name, family in REGISTRY.items():
        rhythm_axes = frozenset(rhythm.AXES)
        deviations = frozenset(family.ladder.plain(probe))
        for probe in _identity_probes(name):
            identity = family.ladder.identity(probe)
            covered = identity | deviations | rhythm_axes
            assert frozenset(family.axes) <= covered, (
                f"{name}: {frozenset(family.axes) - covered} is neither identity nor deviation"
            )


def test_intervals_scale_type_is_identity_only_when_diatonic() -> None:
    ladder = REGISTRY["intervals"].ladder

    assert "scale_type" in ladder.identity({"context": "diatonic"})
    assert "scale_type" not in ladder.identity({"context": "chromatic"})


def test_tapping_is_not_offered_on_a_positional_scale() -> None:
    ladder = REGISTRY["scales"].ladder
    positional = {"scale_type": "ionian", "traversal": "positional"}

    offered = ladder.eligible(positional, frozenset({"ionian"}), frozenset())

    assert "hands" not in offered


def test_tapping_and_inversion_exclude_each_other() -> None:
    """Deviations only accrete, so a ladder cannot un-pin an inversion (L5)."""
    ladder = REGISTRY["arpeggios"].ladder
    identity = {"quality": "min7"}
    tapped = frozenset({"min7"})

    assert "inversion" not in ladder.eligible(identity, tapped, frozenset({"hands"}))
    assert "hands" not in ladder.eligible(identity, tapped, frozenset({"inversion"}))
```

Write `_identity_probes(name)` in the same test module as a small table of
representative identity dictionaries per family — for `intervals`, one with
`context = "diatonic"` and one with `"chromatic"`; for the others, a single
representative draw. It exists so the partition test exercises both halves of
the conditional identity.

- [ ] **Step 2: Run the tests to verify they fail**

Run: `pytest tests/families/test_registry.py -v`

Expected: FAIL with `AttributeError: 'Family' object has no attribute 'ladder'`.

- [ ] **Step 3: Add the `Ladder` record**

In `src/melete/families/__init__.py`:

```python
#: A family's identity axes, given the identity drawn so far (spec §3, L12).
#: A function rather than a tuple because identity can be *conditional*:
#: `intervals` reads `scale_type` only in its diatonic branch, which
#: `selection._CONDITIONAL_AXES` already records. A fixed tuple would have to
#: either omit the axis — leaving a diatonic ladder to raise on every rung with
#: `scale_type` unset — or carry it always, which reintroduces the accounting
#: defect that rule was written to prevent: a chromatic draw crediting §9 with
#: variety no exercise can hear.
type Identity = Callable[[Params], frozenset[str]]

#: Which deviations a family will *offer* on this ladder (spec §4).
#: `(identity, tapped_values, already_chosen) -> offerable axes`. Two kinds of
#: rule live here: preconditions on the identity (`scales` offers `hands` only
#: for a 3nps traversal whose scale type the pool taps) and exclusions between
#: deviations (`arpeggios` refuses `inversion` once `hands` is chosen, because
#: tapping pins the inversion to root and deviations only ever accrete — a
#: ladder that took first inversion at rung 2 cannot tap at rung 3 without
#: silently un-deviating an axis).
type Eligibility = Callable[[Params, frozenset[str], frozenset[str]], frozenset[str]]

#: A family's deviation axes and their plain values, given the drawn identity.
#: All three draw-dependent fields of `Ladder` have this shape, so a reader
#: learns one convention rather than three.
type Plain = Callable[[Params], Mapping[str, AxisValue]]


@dataclass(frozen=True)
class Ladder:
    """How a family grades from its plain shape to its hardest (spec §3, §4).

    Every field is a *reference* to the family module's own declaration, never a
    copy — the same rule `Family` states, and for the same reason: a parallel
    table beside `REGISTRY` is a second line to add and a second one to forget.

    The family is the authority on what can escalate into what. That is not
    politeness about layering: the couplings are facts about the instrument and
    the captured tap shapes, and the recency-weighted sampler cannot express
    them, so they have to be declared somewhere the draw can consult *before*
    committing to a rung.
    """

    #: The axes fixed for every rung, given the identity drawn so far.
    identity: Identity
    #: Each deviation axis and its plain value, **given the identity**.
    #: Membership defines the family's deviation set, and every key is an `AXES`
    #: member (L13): `config` validates pool keys against `Family.axes`, so an
    #: axis outside it has no candidate values to draw and no valid
    #: `[challenge.<family>]` entry.
    #:
    #: A function rather than a flat mapping, for the same reason `identity` is:
    #: the plain value can depend on what was drawn. A triad's plain hand count
    #: is *two* today, because `arpeggio_shapes` carries no one-hand triad seed
    #: shape — so a flat `hands: 1` would make every rung of every triad ladder
    #: unrealizable, rejected, and resampled to exhaustion before surfacing as an
    #: over-constrained pool. It does not bite while the shipped pools list only
    #: sevenths, and it bites the day anyone adds `"maj"` to `qualities`. When
    #: `melete#227` lands, the triad branch is deleted and nothing else moves,
    #: which is the one-line change spec §13 promises.
    plain: Plain
    #: The deviation axes in escalation order, drawn *within* a tier and never
    #: across it (L7). Rotation without ever producing a rung harder than the
    #: one after it.
    tiers: tuple[frozenset[str], ...]
    #: Preconditions and exclusions.
    eligible: Eligibility
```

Add the field to `Family`:

```python
    #: How this family grades from plain to hard (spec §3). Required: a family
    #: with no ladder cannot fill a session slot, and defaulting it to "no
    #: deviations" would engrave five copies of the same plain exercise rather
    #: than failing.
    ladder: Ladder
```

and extend each `REGISTRY` entry with `ladder=<module>.LADDER`.

- [ ] **Step 4: Declare `scales.LADDER`**

```python
# src/melete/families/scales.py

_TAPPABLE_TRAVERSAL = _THREE_NOTE_PER_STRING


def _identity(_params: Params) -> frozenset[str]:
    """What the page is *about*, fixed for every rung (spec §3).

    `traversal` is identity rather than a deviation because the stretch
    fingering **is** the exercise: bare C Ionian three-notes-per-string is a
    drill in its own right, and the positional version of the same scale is a
    different page rather than an easier rung of this one.
    """
    return frozenset({"root", "scale_type", "traversal"})


def _eligible(
    identity: Params,
    tapped_scale_types: frozenset[str],
    _chosen: frozenset[str],
) -> frozenset[str]:
    """Which deviations this identity can be graded with (spec §4).

    Tapping is offered only where the identity already admits it — H1 realizes
    the two-hand tap for the 3nps journey alone (corpus R9) and the pool decides
    which scale types tap at all. A positional ladder therefore has no tapping
    rung and takes its deviations from pattern and rhythm instead, which is the
    price of keeping identity genuinely fixed across the page.
    """
    offered = {"pattern", "accent_pattern", "note_value_pattern"}
    taps = identity.get("traversal") == _TAPPABLE_TRAVERSAL and (
        cast("str", identity.get("scale_type")) in tapped_scale_types
    )
    if taps:
        offered.add(HANDS)
    return frozenset(offered)


def _plain(_identity: Params) -> dict[str, AxisValue]:
    """Every deviation's plain value. No scale type changes them."""
    return {
        "pattern": _STRAIGHT,
        HANDS: _ONE_HAND,
        "accent_pattern": "none",
        "note_value_pattern": "straight",
    }


LADDER = Ladder(
    identity=_identity,
    plain=_plain,
    tiers=(
        frozenset({"pattern"}),
        frozenset({HANDS}),
        frozenset({"accent_pattern", "note_value_pattern"}),
    ),
    eligible=_eligible,
)
```

- [ ] **Step 5: Declare `arpeggios.LADDER`**

```python
# src/melete/families/arpeggios.py

def _identity(_params: Params) -> frozenset[str]:
    return frozenset({"root", "quality"})


def _eligible(
    identity: Params,
    tapped_qualities: frozenset[str],
    chosen: frozenset[str],
) -> frozenset[str]:
    """Which deviations this quality can be graded with (spec §4, L5).

    `hands` and `inversion` exclude each other. Tapping pins the inversion to
    root — the captured tap boxes are root-position shapes — and deviations only
    ever accrete, so a ladder that took first inversion at rung 2 could not tap
    at rung 3 without silently un-deviating an axis. Declaring the pair
    exclusive is how the model refuses that instead of papering over it.
    """
    offered = {"pattern", "accent_pattern", "note_value_pattern"}
    quality = cast("str", identity.get("quality"))
    if (_is_triad(quality) or quality in tapped_qualities) and "inversion" not in chosen:
        offered.add(HANDS)
    if HANDS not in chosen:
        offered.add("inversion")
    return frozenset(offered)


def _plain(identity: Params) -> dict[str, AxisValue]:
    """Every deviation's plain value — which for `hands` depends on the quality.

    A triad's plain hand count is **two**, because `arpeggio_shapes` carries no
    one-hand triad seed shape (decision 10). Returning 1 here would make rung 1
    of every triad ladder unrealizable: the ladder is rejected, resampled to
    exhaustion, and reported as an over-constrained pool rather than as the
    missing seed shape it is. It costs nothing while the shipped pools list only
    sevenths and it fails the day anyone adds `"maj"` to `qualities`.

    `melete#227` is the fix, and when it lands this branch is deleted and
    nothing else moves — the one-line change spec §13 promises.
    """
    return {
        "inversion": INVERSIONS[0],
        "pattern": _STRAIGHT,
        HANDS: _TWO_HANDS if _is_triad(cast("str", identity["quality"])) else _ONE_HAND,
        "accent_pattern": "none",
        "note_value_pattern": "straight",
    }


LADDER = Ladder(
    identity=_identity,
    plain=_plain,
    tiers=(
        frozenset({"inversion", "pattern"}),
        frozenset({HANDS}),
        frozenset({"accent_pattern", "note_value_pattern"}),
    ),
    eligible=_eligible,
)
```

A triad ladder therefore starts *tapped* and cannot escalate into tapping —
`_eligible` must not offer `HANDS` when `_plain` already sets it to two, or the
ladder would draw a deviation that changes nothing. Add that to `_eligible`'s
triad branch, and the `test_identity_and_deviations_partition_the_axes` probe
table gains a triad quality so the case is exercised.

Use the module's existing straight-pattern constant for `"pattern"`; if it is
spelled differently here than in `scales`, use this module's own name.

- [ ] **Step 6: Declare `intervals.LADDER` — the conditional identity**

```python
# src/melete/families/intervals.py

def _identity(params: Params) -> frozenset[str]:
    """Identity, which for this family depends on what has been drawn (L12).

    `scale_type` is read in the diatonic branch only, which
    `selection._CONDITIONAL_AXES` already states. Declaring it identity
    unconditionally would draw a scale type for a chromatic exercise that cannot
    hear it — §9's accounting credited with variety that does not exist, and a
    scale type pushed down the pool without a note of it being played.
    """
    fixed = {"root", "interval", "context"}
    if params.get("context") == _DIATONIC:
        fixed.add("scale_type")
    return frozenset(fixed)


def _eligible(
    _identity: Params,
    _tapped: frozenset[str],
    _chosen: frozenset[str],
) -> frozenset[str]:
    return frozenset({"string_skip", "pattern", "accent_pattern", "note_value_pattern"})


def _plain(_identity: Params) -> dict[str, AxisValue]:
    return {
        "string_skip": "0",
        "pattern": "ascending_pairs",
        "accent_pattern": "none",
        "note_value_pattern": "straight",
    }


LADDER = Ladder(
    identity=_identity,
    plain=_plain,
    tiers=(
        frozenset({"string_skip", "pattern"}),
        frozenset(),
        frozenset({"accent_pattern", "note_value_pattern"}),
    ),
    eligible=_eligible,
)
```

The empty middle tier keeps every family's tier *indices* meaning the same
thing — tier 2 is "technique" everywhere — so a reader comparing two families is
comparing like with like. Ladder construction must tolerate an empty tier.

- [ ] **Step 7: Declare `chromatic.LADDER`**

```python
# src/melete/families/chromatic.py

def _identity(_params: Params) -> frozenset[str]:
    """The four frets and the string they start on, fixed for every rung.

    `span` is identity, not a deviation: three-fret and four-fret spans are
    different exercises for the hand rather than two difficulties of one, and a
    ladder that widened the span mid-page would change what is being practised
    rather than how hard it is.
    """
    return frozenset({"start_string", "start_fret", "span"})


def _eligible(
    _identity: Params,
    _tapped: frozenset[str],
    _chosen: frozenset[str],
) -> frozenset[str]:
    return frozenset(
        {"permutation", "string_traversal", "shift", "accent_pattern", "note_value_pattern"}
    )


def _plain(_identity: Params) -> dict[str, AxisValue]:
    return {
        "permutation": (1, 2, 3, 4),
        "string_traversal": "adjacent",
        "shift": _NO_SHIFT,
        "accent_pattern": "none",
        "note_value_pattern": "straight",
    }


LADDER = Ladder(
    identity=_identity,
    plain=_plain,
    tiers=(
        frozenset({"permutation"}),
        frozenset({"string_traversal", "shift"}),
        frozenset({"accent_pattern", "note_value_pattern"}),
    ),
    eligible=_eligible,
)
```

- [ ] **Step 8: Run the tests to verify they pass**

Run: `pytest tests/families/test_registry.py -v`

Expected: PASS, all five tests.

- [ ] **Step 9: Validate and commit**

```bash
vrg-container-run -- vrg-validate
vrg-git add src/melete/families/
vrg-commit --type feat --scope families \
  --message "declare each family's ladder (#87)" \
  --body "Ladder joins Family as a sibling record: identity, plain values,
escalation tiers and the eligibility rule, each a reference to the family's own
declaration rather than a copy.

Identity is a function of the drawn identity rather than a fixed tuple, because
intervals reads scale_type in its diatonic branch alone: a fixed tuple would
either omit it — leaving a diatonic ladder to raise on every rung — or draw it
always, reintroducing the accounting defect _CONDITIONAL_AXES exists to prevent.

Eligibility carries both preconditions (scales offers tapping only for a 3nps
traversal the pool taps) and exclusions (arpeggios refuses inversion once hands
is chosen, because tapping pins the inversion to root and deviations only
accrete).

Ref: mnemosys-project/.github#87"
```

---

### Task 4: `ladder.py` — building the chain

Spec §3, §5 steps 3–6. Pure construction: given an identity, a `Ladder`
declaration and a draw callback, produce the cumulative rung dictionaries. No
config, no realization, no I/O.

**Files:**
- Create: `src/melete/ladder.py`
- Test: `tests/test_ladder.py`

**Interfaces:**
- Consumes: `families.Ladder`, `families.REGISTRY`.
- Produces:

```python
@dataclass(frozen=True)
class LadderSpec:
    family: str
    identity: dict[str, AxisValue]
    rungs: tuple[dict[str, AxisValue], ...]
    challenge: dict[str, AxisValue] | None

DEVIATION: str = "deviation"

def build(
    family: str,
    identity: Mapping[str, AxisValue],
    *,
    rungs: int,
    tapped: frozenset[str],
    draw_axes: Callable[[Sequence[str], int], Sequence[str]],
    draw_value: Callable[[str, str], AxisValue],   # (axis, pool_name) -> value
    challenge: bool,
) -> LadderSpec: ...

def materialize(spec: LadderSpec, rung: int) -> dict[str, AxisValue]: ...
def rungs_of(spec: LadderSpec) -> tuple[dict[str, AxisValue], ...]: ...
```

`draw_axes` and `draw_value` are injected rather than imported so this module
stays free of `selection` and `config`; the selector supplies recency-weighted
implementations in Task 7, and the tests supply deterministic ones.
`rungs_of` returns `rungs` plus `challenge` when present — the one place the
"all engraved rungs" sequence is defined, so the emitter, the session writer and
the validity gate cannot disagree about it.

- [ ] **Step 1: Write the failing tests**

```python
# tests/test_ladder.py
"""Tests for ladder construction (epic #87, spec §3 and §5).

Ordered the way the spec's argument is: monotonicity first, because it is what
makes "what was added at this rung" a well-defined question; then tier ordering,
then the two eligibility rules, then the challenge rung's two routes.
"""

from __future__ import annotations

import pytest

from melete import ladder
from melete.families import REGISTRY


def picker(order: list[str]):
    """A deterministic `draw_axes`: take the named axes, in the named order."""

    def draw_axes(candidates, count):
        chosen = [axis for axis in order if axis in candidates][:count]
        assert len(chosen) == count, f"{order} cannot supply {count} of {sorted(candidates)}"
        return chosen

    return draw_axes


def valuer(values: dict[str, object]):
    def draw_value(axis, _pool):
        return values[axis]

    return draw_value


def test_rungs_are_monotone() -> None:
    """Rung k's deviations are a superset of rung k-1's (spec §3).

    Monotonicity is load-bearing rather than incidental: it is what lets a rung
    title say what was *added*, and what forbids a ladder from switching a
    technique off again.
    """
    spec = ladder.build(
        "scales",
        {"root": 24, "scale_type": "ionian", "traversal": "positional"},
        rungs=4,
        tapped=frozenset(),
        draw_axes=picker(["pattern", "accent_pattern", "note_value_pattern"]),
        draw_value=valuer(
            {
                "pattern": "groups_of_3",
                "accent_pattern": "every_3",
                "note_value_pattern": "long_short",
            }
        ),
        challenge=False,
    )

    assert spec.rungs[0] == {}
    for earlier, later in zip(spec.rungs, spec.rungs[1:], strict=False):
        assert set(earlier) < set(later)
        assert all(later[axis] == value for axis, value in earlier.items())


def test_a_later_tier_never_precedes_an_earlier_one() -> None:
    """Order is drawn within a tier, never across it (L7)."""
    spec = ladder.build(
        "scales",
        {"root": 24, "scale_type": "ionian", "traversal": "positional"},
        rungs=4,
        tapped=frozenset(),
        draw_axes=picker(["accent_pattern", "note_value_pattern", "pattern"]),
        draw_value=valuer(
            {
                "pattern": "groups_of_3",
                "accent_pattern": "every_3",
                "note_value_pattern": "long_short",
            }
        ),
        challenge=False,
    )

    # `pattern` is tier 1 and was picked last; it must still switch on first.
    assert set(spec.rungs[1]) == {"pattern"}


def test_tapping_is_not_offered_on_a_positional_identity() -> None:
    with pytest.raises(ladder.LadderError, match=r"hands"):
        ladder.build(
            "scales",
            {"root": 24, "scale_type": "ionian", "traversal": "positional"},
            rungs=4,
            tapped=frozenset({"ionian"}),
            draw_axes=picker(["hands", "pattern", "accent_pattern"]),
            draw_value=valuer({}),
            challenge=False,
        )


def test_the_challenge_rung_stacks_when_the_menu_has_room() -> None:
    spec = ladder.build(
        "scales",
        {"root": 24, "scale_type": "ionian", "traversal": "three_note_per_string"},
        rungs=4,
        tapped=frozenset({"ionian"}),
        draw_axes=picker(["pattern", "hands", "accent_pattern", "note_value_pattern"]),
        draw_value=valuer(
            {
                "pattern": "groups_of_3",
                "hands": 2,
                "accent_pattern": "every_3",
                "note_value_pattern": "short_long",
            }
        ),
        challenge=True,
    )

    assert set(spec.challenge) - set(spec.rungs[-1]) == {"note_value_pattern"}


def test_the_challenge_rung_redraws_a_value_when_the_menu_is_exhausted() -> None:
    """Half of all `scales` ladders reach this: positional prunes `hands`,
    leaving a menu of three against the three rungs 2-4 need (L14)."""
    drawn = {"pattern": "groups_of_3", "accent_pattern": "every_3",
             "note_value_pattern": "long_short"}

    def draw_value(axis, pool):
        return "numeric_1235" if pool == "challenge" and axis == "pattern" else drawn[axis]

    spec = ladder.build(
        "scales",
        {"root": 24, "scale_type": "ionian", "traversal": "positional"},
        rungs=4,
        tapped=frozenset(),
        draw_axes=picker(["pattern", "accent_pattern", "note_value_pattern"]),
        draw_value=draw_value,
        challenge=True,
    )

    assert set(spec.challenge) == set(spec.rungs[-1])
    assert spec.challenge["pattern"] == "numeric_1235"


def test_materialize_fills_plain_for_every_untouched_deviation() -> None:
    spec = ladder.build(
        "scales",
        {"root": 24, "scale_type": "ionian", "traversal": "positional"},
        rungs=2,
        tapped=frozenset(),
        draw_axes=picker(["pattern"]),
        draw_value=valuer({"pattern": "groups_of_3"}),
        challenge=False,
    )

    plain = ladder.materialize(spec, 0)

    assert plain["pattern"] == "straight"
    assert plain["accent_pattern"] == "none"
    assert plain["hands"] == 1
    assert plain["scale_type"] == "ionian"
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `pytest tests/test_ladder.py -v`

Expected: FAIL with `ModuleNotFoundError: No module named 'melete.ladder'`.

- [ ] **Step 3: Write `src/melete/ladder.py`**

Module docstring first, in this codebase's register — it must carry the
argument, not merely the behaviour:

```python
"""The graded ladder: one identity, played at escalating difficulty (spec §3, §5).

Every axis melete samples is drawn independently, and nothing in that model says
some axes are harder than others. So a slot arrives with its pattern, its rhythm,
its accents and its technique all already deviating from plain, and the odds of
drawing five plain values at once are negligible: the sheet lands at the top of
its difficulty range every day, and the plain shape — a legitimate exercise, and
often a demanding one — is never engraved at all.

This module is the answer. An exercise's axes split into **identity**, fixed for
the whole page, and **deviations**, each with a plain value; a rung is defined by
which deviations have left plain, and a ladder is the monotone chain from
all-plain to the top.

## Monotonicity is the load-bearing property

Rung k's deviations are a strict superset of rung k-1's. That is what makes "what
was added here" a well-defined question at every rung — the thing a page title
has to answer — and it is what forbids a ladder from switching a technique off
again. It is also why `arpeggios` declares `hands` and `inversion` mutually
exclusive: tapping pins the inversion to root, so a ladder that took first
inversion at rung 2 could not tap at rung 3 without silently un-deviating an
axis, and a silent un-deviation is a page whose titles lie about what it plays.

## Why the draws are injected

`build` takes `draw_axes` and `draw_value` as callbacks rather than importing the
selector. Construction is then a pure function of its inputs and testable against
a fixed pick order, and — more importantly — this module cannot acquire an
opinion about *weighting*, which belongs to §9 and is the seam through which
technique-weighted history arrives later.

## The challenge rung escalates; it does not merely stack

Menu exhaustion is arithmetic, not a corner case. The `scales` menu is four
deviations and a four-rung ladder draws three of them; a *positional* identity
has `hands` pruned by the family's eligibility rule, leaving exactly three, and
`traversal` is drawn from two values — so roughly half of all `scales` ladders
have nothing left to stack. The challenge rung therefore adds an unused deviation
when one remains and re-draws an active deviation's value from the challenge pool
when none does (L14). Both routes reach the same place, and neither is a fallback
for the other: on an already-tapped scale in groups of three, moving the pattern
to 1-2-3-5 is the sharper step anyway.
"""
```

Then the implementation:

```python
#: The pseudo-axis the *set* of deviations is drawn on (spec §5 step 3). Its
#: values are axis names, which is what makes "I have not been made to play with
#: accents in nine days" expressible in §9's existing machinery — and it is the
#: single seam through which technique-weighted history arrives later, as a
#: different weight function over this axis rather than a new mechanism.
DEVIATION = "deviation"

#: The pool name `draw_value` is told to read for the challenge rung.
CHALLENGE = "challenge"

#: The pool name `draw_value` is told to read for an ordinary rung.
ORDINARY = "pool"


class LadderError(ValueError):
    """A ladder that cannot be built from the declaration and the identity."""


@dataclass(frozen=True)
class LadderSpec:
    """One slot: an identity, and the rungs it is graded across (spec §3)."""

    family: str
    identity: dict[str, AxisValue]
    rungs: tuple[dict[str, AxisValue], ...]
    challenge: dict[str, AxisValue] | None


def rungs_of(spec: LadderSpec) -> tuple[dict[str, AxisValue], ...]:
    """Every engraved rung, challenge included.

    The one place that sequence is defined. The validity gate, the session
    writer and the emitter all need it, and three private answers to "how many
    exercises is this slot" is three chances to disagree about whether the
    challenge rung counts.
    """
    if spec.challenge is None:
        return spec.rungs
    return (*spec.rungs, spec.challenge)


def build(family, identity, *, rungs, tapped, draw_axes, draw_value, challenge):
    declaration = REGISTRY[family].ladder
    chosen: list[str] = []
    menu = declaration.eligible(identity, tapped, frozenset())

    wanted = rungs - 1
    if len(menu) < wanted:
        msg = (
            f"{family}: the identity {dict(identity)!r} offers {len(menu)} deviations "
            f"({sorted(menu)}) but [ladder] rungs.{family} = {rungs} needs {wanted}. "
            f"The ladder is never shortened to fit (spec §10)"
        )
        raise LadderError(msg)

    for _ in range(wanted):
        offered = declaration.eligible(identity, tapped, frozenset(chosen)) - frozenset(chosen)
        if not offered:
            msg = (
                f"{family}: {sorted(chosen)} exhausted the deviation menu after "
                f"{len(chosen)} of {wanted} rungs"
            )
            raise LadderError(msg)
        chosen += list(draw_axes(sorted(offered), 1))

    ordered = _by_tier(declaration.tiers, chosen, family)
    values = {axis: draw_value(axis, ORDINARY) for axis in ordered}

    accumulated: list[dict[str, AxisValue]] = [{}]
    for axis in ordered:
        accumulated.append({**accumulated[-1], axis: values[axis]})

    return LadderSpec(
        family=family,
        identity=dict(identity),
        rungs=tuple(accumulated),
        challenge=(
            _challenge(
                declaration, identity, tapped, accumulated[-1], draw_axes, draw_value, family
            )
            if challenge
            else None
        ),
    )
```

`_by_tier` sorts the chosen axes by the index of the tier containing each, and
raises a `LadderError` naming the axis and the family if an axis appears in no
tier — a declaration that offers a deviation it cannot order is a bug in the
family, not a value to guess at.

`_challenge` implements the two routes:

```python
def _challenge(declaration, identity, tapped, top, draw_axes, draw_value, family):
    """The level++ rung: stack an unused deviation, or re-draw an active one (L14).

    **The axis is drawn, never picked.** An earlier draft took `sorted(...)[0]`
    on both branches, which meant the challenge rung escalated the
    alphabetically-first available axis every time — `accent_pattern` on
    essentially every `scales` page. Deterministic tie-breaking is right for
    *ordering within a tier*, where the tier has already fixed the difficulty;
    it is wrong for *choosing which technique escalates*, and it would have made
    the one rung the player is meant to find unfamiliar the only rung that never
    varies. Drawing it also keeps the accounting honest: the selector records a
    use on the `DEVIATION` axis for every deviation the ladder carries, and a
    hardcoded axis would record a use no draw ever made.
    """
    spare = declaration.eligible(identity, tapped, frozenset(top)) - frozenset(top)
    candidates = sorted(spare) if spare else sorted(top)
    if not candidates:
        msg = (
            f"{family}: a challenge rung needs either an unused deviation to stack or an "
            f"active one to re-draw, and this ladder has neither"
        )
        raise LadderError(msg)
    axis = draw_axes(candidates, 1)[0]
    return {**top, axis: draw_value(axis, CHALLENGE)}
```

`sorted` survives only as the *candidate ordering* handed to `draw_axes`, which
a weighted draw needs to be deterministic under a seed; the choice among them is
the draw's.

`materialize` composes identity, plain values and the rung's deviations, then
lets the family's `derive` fill what follows — which after Task 1 never
overrules the rung:

```python
def materialize(spec: LadderSpec, rung: int) -> dict[str, AxisValue]:
    """One rung as an ordinary `ExerciseSpec`'s params (spec §3).

    This is the containment decision the whole epic rests on: downstream of here
    nothing knows ladders exist. `pipeline.realize`, the fitter, the rhythm
    modifier, every family's `generate` and the emitter's per-exercise engraving
    receive exactly what they received before.
    """
    declaration = REGISTRY[spec.family].ladder
    params: dict[str, AxisValue] = {
        **spec.identity,
        **dict(declaration.plain(spec.identity)),
        **rungs_of(spec)[rung],
    }
    for axis, value in REGISTRY[spec.family].derive(params, frozenset()).items():
        params.setdefault(axis, cast("AxisValue", value))
    return params
```

- [ ] **Step 4: Run the tests to verify they pass**

Run: `pytest tests/test_ladder.py -v`

Expected: PASS, all six tests.

- [ ] **Step 5: Validate and commit**

```bash
vrg-container-run -- vrg-validate
vrg-git add src/melete/ladder.py tests/test_ladder.py
vrg-commit --type feat --scope ladder \
  --message "build the graded ladder chain (#87)" \
  --body "An identity plus a monotone chain of deviations, each rung a strict
superset of the one below it. Draw callbacks are injected rather than imported,
so construction is pure and cannot acquire an opinion about weighting.

The challenge rung escalates by stacking an unused deviation or re-drawing an
active one. Menu exhaustion is arithmetic rather than a corner case: a
positional scales identity prunes tapping, leaving exactly three deviations
against the three a four-rung ladder needs, and traversal is drawn from two
values.

Ref: mnemosys-project/.github#87"
```

---

### Task 5: `[ladder]` and `[challenge.<family>]` configuration

Spec §7, decision **L15**.

**Files:**
- Modify: `src/melete/config.py` (a `LadderConfig`, the `[ladder]` and
  `[challenge.*]` readers, `Config`, and `_fingerprint` in `session.py`)
- Modify: `examples/config.toml`
- Test: `tests/test_config.py`

**Interfaces:**
- Consumes: Task 3's `Ladder` declarations (for the menu-size check).
- Produces:
  - `Config.ladder: LadderConfig` with `rungs: Mapping[str, int]` and
    `challenge: bool`
  - `Config.challenge: Mapping[str, FamilyPool]` — same shape as `Config.pool`,
    validated identically

- [ ] **Step 1: Write the failing tests**

```python
# tests/test_config.py

def test_ladder_rungs_nest_under_their_own_key() -> None:
    """L15: family counts nest, so no reserved non-family name shares the table."""
    active = load_string(HEADER + """
        [ladder]
        rungs = { scales = 4, chromatic = 3 }
        challenge = true
    """)

    assert active.ladder.rungs["scales"] == 4
    assert active.ladder.challenge is True


def test_ladder_rejects_an_unknown_family() -> None:
    with pytest.raises(ConfigError, match=r"ladder\.rungs.*bagpipes"):
        load_string(HEADER + """
            [ladder]
            rungs = { bagpipes = 4 }
        """)


def test_a_rung_count_beyond_the_family_menu_is_refused() -> None:
    """No draw could ever satisfy it, so it fails at load, not at draw (spec §10)."""
    with pytest.raises(ConfigError, match=r"ladder\.rungs\.scales"):
        load_string(HEADER + """
            [ladder]
            rungs = { scales = 12 }
        """)


def test_challenge_pool_is_validated_like_an_ordinary_pool() -> None:
    with pytest.raises(ConfigError, match=r"challenge\.scales\.patterns"):
        load_string(HEADER + """
            [challenge.scales]
            patterns = ["not_a_pattern"]
        """)


def test_a_challenge_pool_that_cannot_escalate_is_refused() -> None:
    """§10 row 4. Without this the draw resamples 500 times and reports an
    over-constrained pool — the wrong problem, at the wrong layer."""
    with pytest.raises(ConfigError, match=r"challenge\.scales.*could never escalate"):
        load_string(HEADER + """
            [ladder]
            rungs = { scales = 4 }
            challenge = true

            [challenge.scales]
            roots = [0, 1]
        """)


def test_challenge_requested_without_a_pool_for_a_shaped_family_is_refused() -> None:
    """A challenge rung silently identical to rung 4 is the failure §10 exists
    to prevent, so the absence is loud rather than a fallback to [pool]."""
    with pytest.raises(ConfigError, match=r"challenge\.scales"):
        load_string(HEADER + """
            [ladder]
            rungs = { scales = 4 }
            challenge = true
        """)
```

`HEADER` is the existing fixture in this module supplying `[instrument]`,
`[session]` and the pools; extend it if it does not already shape `scales`.

- [ ] **Step 2: Run the tests to verify they fail**

Run: `pytest tests/test_config.py -k "ladder or challenge" -v`

Expected: FAIL — `[ladder]` and `[challenge]` are rejected as unknown top-level
sections by `config.py:721`'s `_reject_unknown("config", raw, ("instrument",
"session", "pool"))`.

- [ ] **Step 3: Add the reader**

```python
@dataclass(frozen=True)
class LadderConfig:
    """§7's `[ladder]`: how many rungs each family grades across.

    The family counts nest inside `rungs` rather than sitting loose beside
    `challenge`, mirroring `[session] shape` — the same shape of problem, already
    solved once in this file. Flat keys would put a reserved non-family name in a
    table validated against `families.REGISTRY`, costing one hardcoded exemption
    forever, making a future family named `challenge` unreachable, and reading as
    though `challenge = 4` were meaningful when the boolean is global and the
    counts are per-family.
    """

    rungs: Mapping[str, int]
    challenge: bool
```

Read it beside `_shape`:

```python
def _ladder(raw: object) -> LadderConfig:
    section = _table("ladder", raw)
    _reject_unknown("ladder", section, ("rungs", "challenge"))
    counts = _table("ladder.rungs", section.get("rungs", {}))
    _reject_unknown("ladder.rungs", counts, vocabulary.accepted("family"))
    return LadderConfig(
        rungs={
            family: _positive(f"ladder.rungs.{family}", count)
            for family, count in counts.items()
        },
        challenge=_flag("ladder.challenge", section.get("challenge", False)),
    )
```

Add `"ladder"` and `"challenge"` to the top-level `_reject_unknown` at
`config.py:721`, and read `[challenge.<family>]` through the *same* pool reader
`[pool.<family>]` uses — the surfaces are identical by design, and a second
reader is a second place for them to drift.

- [ ] **Step 4: Add the two cross-checks**

Both are load-time refusals, because neither can ever be satisfied by a draw:

```python
def _ladder_fits_the_families(ladder: LadderConfig, shaped: Iterable[str]) -> None:
    """A rung count no identity could supply is refused at load (spec §10).

    Checked against the family's *whole* declared deviation set, not against the
    menu a particular identity leaves — that one is a per-draw rejection. This is
    the case where no draw could ever succeed, and discovering it 500 resamples
    later would report it as an over-constrained pool rather than as the typo it
    is.
    """
    for family, count in ladder.rungs.items():
        menu = len(REGISTRY[family].ladder.plain(_widest_identity(family)))
        if count - 1 > menu:
            _fail(
                f"ladder.rungs.{family}",
                f"asks for {count} rungs, which needs {count - 1} deviations, but "
                f"{family} declares only {menu} deviations",
            )
```

and, when `challenge` is true, that every family named in `[session] shape` has a
`[challenge.<family>]` section — with the message saying explicitly that falling
back to `[pool]` would make the challenge rung indistinguishable from rung 4.

A third check covers §10's fourth row: a challenge section that exists but names
**no axis the ladder could ever escalate**.

```python
def _challenge_can_escalate(family: str, pool: FamilyPool, ladder: LadderConfig) -> None:
    """A challenge pool disjoint from the family's deviations is refused (spec §10).

    Checked at load because the alternative is a misdiagnosis. Without it the
    draw reaches `draw_value(axis, CHALLENGE)`, `_candidates` raises "configures
    no candidate values", `_fill` counts that as a rejection reason, and the run
    resamples `MAX_ATTEMPTS` times before reporting an over-constrained *pool* —
    the wrong problem, at the wrong layer, after a visible delay, when the real
    repair is a two-line configuration edit.

    The intersection is with the family's whole declared deviation set rather
    than with the axes a particular ladder happens to draw: this is the case no
    draw could satisfy, and the narrower one is a per-draw rejection.
    """
    deviations = frozenset(REGISTRY[family].ladder.plain(_widest_identity(family)))
    if not deviations & frozenset(pool.values):
        _fail(
            f"challenge.{family}",
            f"configures {sorted(pool.values)}, none of which {family} grades on "
            f"({sorted(deviations)}), so the challenge rung could never escalate anything",
        )
```

`_widest_identity(family)` returns a representative identity for the family —
the one that yields its largest deviation set. Both this check and
`_ladder_fits_the_families` need it, so write it once, next to them, and give it
a docstring saying it exists to ask "could *any* identity satisfy this?" rather
than "does this one?".

- [ ] **Step 5: Extend the configuration fingerprint**

In `session._fingerprint`, add the two new sections beside `pool` and `rhythm`,
so editing a ladder or a challenge pool re-seeds the draw exactly as editing
`[pool]` does:

```python
        "ladder": {
            "rungs": dict(config.ladder.rungs),
            "challenge": config.ladder.challenge,
        },
        "challenge": {
            family: {axis: [_jsonable(v) for v in values]
                     for axis, values in pool.values.items()}
            for family, pool in config.challenge.items()
        },
```

- [ ] **Step 6: Run the tests to verify they pass**

Run: `pytest tests/test_config.py tests/test_session.py -v`

Expected: PASS. `test_session.py`'s hash tests will need the new sections in
their fixture configurations; update them rather than exempting the sections
from the fingerprint.

- [ ] **Step 7: Update the worked example**

Add to `examples/config.toml`, after `[session]`:

```toml
[ladder]
# Ordinary rungs per family, rung 1 (the plain shape) included. Four is the
# starting point, not a tuned value: the point of the first iteration is to see
# the whole spectrum before calibrating it (spec §7).
rungs     = { chromatic = 4, scales = 4, arpeggios = 4, intervals = 4 }
challenge = true

# The challenge pool is deliberately disjoint from [pool.*]: these values appear
# on the last rung of a page and nowhere else, which is what makes it reliably
# unfamiliar rather than merely longer.
[challenge.chromatic]
accent_patterns     = ["every_5", "displaced"]
note_value_patterns = ["short_long"]
shifts              = ["position_per_cycle"]

[challenge.scales]
patterns            = ["numeric_1235", "fourths"]
accent_patterns     = ["every_5", "displaced"]
note_value_patterns = ["short_long"]

[challenge.arpeggios]
patterns            = ["numeric_1353"]
inversions          = ["third"]
accent_patterns     = ["every_5", "displaced"]
note_value_patterns = ["short_long"]

[challenge.intervals]
string_skips        = [2]
accent_patterns     = ["every_5", "displaced"]
note_value_patterns = ["short_long"]
```

Every value above is already in `vocabulary.AXES` and drawn by no everyday pool
— that unused harder vocabulary is exactly what the challenge rung exists to
reach.

- [ ] **Step 8: Validate and commit**

```bash
vrg-container-run -- vrg-validate
vrg-git add src/melete/config.py src/melete/session.py examples/config.toml \
    tests/test_config.py tests/test_session.py
vrg-commit --type feat --scope config \
  --message "read [ladder] and [challenge.<family>] (#87)" \
  --body "Rung counts nest as [ladder] rungs = { ... }, mirroring [session]
shape, so no reserved non-family name shares a table validated against
families.REGISTRY.

[challenge.<family>] is read through the same reader [pool.<family>] uses — the
surfaces are identical by design and a second reader is a second place to drift
— but stays a disjoint pool consulted only for the final rung.

Two refusals happen at load rather than at draw, because no draw could satisfy
either: a rung count beyond the family's whole declared deviation set, and a
requested challenge rung with no pool for a shaped family. The second matters
most — falling back to [pool] would make the challenge rung indistinguishable
from rung 4 while looking like a working feature.

Both sections enter the configuration fingerprint, so editing them re-seeds the
draw exactly as editing [pool] does.

Ref: mnemosys-project/.github#87"
```

---

### Task 6: Measure the ladder validity rate

Decision **L10**, spec §6. This task produces a **number and a report**, not a
feature. It runs before the gate is wired so that per-deviation repair is built
only if the measurement demands it — the epic's answer to complexity accretion.

**Files:**
- Create: `docs/reports/ladder-validity-rate.md`
- Test: none (a measurement, not behaviour). The harness lives in
  `tests/test_fit_sweep.py`'s style but is run manually.

**Interfaces:**
- Consumes: `ladder.build`, `ladder.materialize`, `Config.ladder`.
- Produces: a measured composite valid rate per family, and a recommended
  `MAX_ATTEMPTS`, cited by Task 7's docstring.

- [ ] **Step 1: Write the measurement harness**

A script under the scratchpad (not committed): for each family, draw 20,000
ladders from a broad pool on `bass6` with `rungs = 4` and `challenge = true`,
materialize **every** rung, run each through `pipeline.realize` and the
`max_notes` / `max_fret_span` bounds, and count a ladder valid only when every
rung is. Mirror the existing single-spec measurement in `MAX_ATTEMPTS`'s
docstring (`selection.py:175`) so the two tables are comparable.

- [ ] **Step 2: Run it and record the numbers**

Expect the composite rate to be **well below** the single-spec rates the
docstring records (0.278 `scales`, 0.347 `chromatic`, 0.579 `arpeggios`), since
a ladder is valid only if all five rungs are. Record the actual figure per
family; do not estimate it.

- [ ] **Step 3: Write the report**

`docs/reports/ladder-validity-rate.md`, following the house style of the other
files in `docs/reports/`: what was measured, how, the table, and the conclusion.
The conclusion must answer exactly one question — **does whole-ladder resampling
converge at a defensible `MAX_ATTEMPTS`, or is per-deviation repair required?**

- [ ] **Step 4: Act on the answer**

- **If the rate supports it:** state the `MAX_ATTEMPTS` the numbers justify.
  Repair is never built; note that in the report and in §13 of the spec.
- **If it does not:** file a task for per-deviation repair under epic #87 —
  keep the identity, redraw the offending deviation's value, then its axis, and
  only then abandon the ladder — with this report as its evidence. Task 7 then
  consumes that task rather than the simple gate.

- [ ] **Step 5: Commit**

```bash
vrg-container-run -- vrg-validate
vrg-git add docs/reports/ladder-validity-rate.md
vrg-commit --type docs --scope reports \
  --message "measure the ladder validity rate (#87)" \
  --body "A ladder is valid only if every rung is, so the composite rate is
materially below the single-spec rates MAX_ATTEMPTS was derived from. Measured
rather than estimated, per decision L10, so that per-deviation repair is built
only if the numbers demand it.

Ref: mnemosys-project/.github#87"
```

---

### Task 7: `select()` draws ladders

Spec §5. The slot now fills a `LadderSpec`, and the validity gate covers every
rung.

**Files:**
- Modify: `src/melete/selection.py` (`select`, `_fill`, `_sample` → identity
  sampling, `_uses`, `_rejected`, `MAX_ATTEMPTS`'s docstring)
- Modify: `src/melete/cli.py:262-293` (`_generate`), `:312` (`_score`)
- Test: `tests/test_selection.py`

**Interfaces:**
- Consumes: Tasks 3–6.
- Produces: `select(config, history, rng) -> list[tuple[LadderSpec, WeightInputs]]`

- [ ] **Step 1: Write the failing tests**

```python
# tests/test_selection.py

def test_select_returns_one_ladder_per_slot() -> None:
    active = load_string(LADDER_CONFIG)

    picks = select(active, [], seeded(7))

    assert len(picks) == sum(active.session.shape.values())
    for spec, _inputs in picks:
        assert len(ladder.rungs_of(spec)) == active.ladder.rungs[spec.family] + 1


def test_rung_one_is_the_plain_shape() -> None:
    """The whole point of the epic: the bare exercise is engraved (spec §1)."""
    active = load_string(LADDER_CONFIG)

    for spec, _inputs in select(active, [], seeded(11)):
        plain = ladder.materialize(spec, 0)
        for axis, value in REGISTRY[spec.family].ladder.plain(spec.identity).items():
            assert plain[axis] == value


def test_a_tapped_eligible_ladder_still_starts_one_handed() -> None:
    """The L11 regression, end to end: derive must not overrule rung 1.

    This is the one failure the acceptance sheet would not reveal by eye — the
    book renders, the rungs differ in pattern and rhythm, and only playing it
    shows the plain scale was never engraved.
    """
    active = load_string(TAPPED_SCALES_CONFIG)

    for spec, _inputs in select(active, [], seeded(3)):
        if spec.family == "scales":
            assert ladder.materialize(spec, 0)["hands"] == 1


def test_the_deviation_axis_is_recency_weighted() -> None:
    """A deviation drawn today is drawn less often tomorrow (spec §5 step 3)."""
    active = load_string(LADDER_CONFIG)
    first = select(active, [], seeded(5))
    used = {axis for spec, _ in first for rung in spec.rungs for axis in rung}

    later = select(active, [tuple(spec for spec, _ in first)], seeded(5))
    repeated = {axis for spec, _ in later for rung in spec.rungs for axis in rung}

    assert repeated != used


def test_every_rung_passes_the_validity_gate() -> None:
    active = load_string(LADDER_CONFIG)

    for spec, _inputs in select(active, [], seeded(13)):
        for index in range(len(ladder.rungs_of(spec))):
            params = ladder.materialize(spec, index)
            pipeline.realize(active.instrument, spec.family, params)  # must not raise
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `pytest tests/test_selection.py -k ladder -v`

Expected: FAIL — `select` returns `ExerciseSpec`s with no `rungs` attribute.

- [ ] **Step 3: Split identity sampling out of `_sample`**

Rename the per-axis loop to `_sample_identity(family, config, slot, axes)`,
taking the axis set from `REGISTRY[family].ladder.identity(params)` re-evaluated
as each axis is drawn — that re-evaluation is what makes conditional identity
work (L12), since `scale_type` only joins the set once `context` has been drawn.
Keep `_CONDITIONAL_AXES` as the cross-check it already is.

- [ ] **Step 4: Wire the ladder draw**

```python
def _fill(family: str, config: Config, slot: _Slot) -> LadderSpec:
    """One valid ladder for `family`, resampling until the budget runs out.

    Whole-ladder resampling rather than per-deviation repair (L10): the simple
    gate ships first and `docs/reports/ladder-validity-rate.md` is the
    measurement that says whether it converges. A ladder is valid only when
    *every* rung realizes, so the reason counter aggregates by rung index — an
    over-constrained draw should say which rung was the obstacle, not merely
    that one was.
    """
    reasons: Counter[str] = Counter()
    for _ in range(MAX_ATTEMPTS):
        identity = _sample_identity(family, config, slot)
        try:
            spec = ladder.build(
                family,
                identity,
                rungs=config.ladder.rungs[family],
                tapped=config.pool[family].tapped_values,
                draw_axes=lambda candidates, count: [
                    slot.draw(ladder.DEVIATION, candidates) for _ in range(count)
                ],
                draw_value=lambda axis, pool: slot.draw(
                    axis, _candidates(family, axis, _pool_for(config, family, pool))
                ),
                challenge=config.ladder.challenge,
            )
        except ladder.LadderError as exc:
            reasons[str(exc)] += 1
            continue
        reason = _rejected_ladder(spec, config)
        if reason is None:
            return spec
        reasons[reason] += 1
    raise _over_constrained(family, reasons, slot)
```

`_pool_for` returns `config.pool[family].values` for `ladder.ORDINARY` and
`config.challenge[family].values` for `ladder.CHALLENGE`.

`_rejected_ladder` materializes each rung, calls the existing `_rejected`, and
prefixes the reason with `rung {index}: ` so the aggregated counter names the
obstacle.

- [ ] **Step 5: Record the right uses**

`_uses(spec)` becomes: the identity values, one use of each deviation *axis* on
the `DEVIATION` axis, and each deviation's *value* on its own axis. Plain values
are never drawn, so they are never counted — which is what stops rung 1's
mandatory `straight` and `none` from poisoning the weighting for those values as
deviations. Put that sentence in the docstring; it is the non-obvious half.

- [ ] **Step 6: Update the CLI**

`cli._generate` maps each `LadderSpec` through `ladder.rungs_of` and
`ladder.materialize` into the `scores` list, so one slot contributes N scores.
`cli._score` keeps taking a family and a params mapping — change its signature to
`(active, family, params)` rather than teaching it about ladders.

- [ ] **Step 7: Run the tests to verify they pass**

Run: `pytest tests/test_selection.py tests/test_cli_generate.py -v`

Expected: PASS.

- [ ] **Step 8: Validate and commit**

```bash
vrg-container-run -- vrg-validate
vrg-git add src/melete/selection.py src/melete/cli.py tests/test_selection.py
vrg-commit --type feat --scope selection \
  --message "draw a graded ladder per slot (#87)" \
  --body "A slot fills a LadderSpec: identity on the existing per-axis
weighting, the deviation set on a new DEVIATION pseudo-axis whose values are
axis names, the route ordered within declared tiers, and each deviation's value
on its own axis.

Identity axes are re-evaluated as each is drawn, which is what makes conditional
identity work: intervals.scale_type joins the set only once context has been
drawn chromatic or diatonic.

The gate covers every rung and resamples the whole ladder, per L10 — the reason
counter aggregates by rung index so an over-constrained draw names which rung
was the obstacle.

Plain values are never drawn and never counted, so rung 1's mandatory straight
and none cannot poison the weighting for those values as deviations.

Ref: mnemosys-project/.github#87"
```

---

### Task 8: The reshaped session record, and the replay path

Spec §8, §11, decision **L9** — clean reset, no back-compatibility.

The record and everything that reads it land together. `cli.ordered`
(`cli.py:461-489`) walks `spec.params` to restore the drawn axis order, and both
`replay` and `show` depend on it — its own docstring says why: without it a
replayed sheet lists each exercise's axes in a different order than the sheet it
reproduces, "the one visible difference between the two, in the one place a
reader compares them." This task deletes `params` from the record, so leaving
`ordered` to a later task would merge a green test suite over a broken `replay`.

**Files:**
- Modify: `src/melete/session.py` (`_document`, `_exercise`, `_EXERCISE_KEYS`,
  `history`, `replay`)
- Modify: `src/melete/cli.py:461-489` (`ordered`), `:514` (`_replay`)
- Test: `tests/test_session.py`, `tests/test_cli_query.py`

**Interfaces:**
- Consumes: Task 7's `LadderSpec`.
- Produces: `Session.exercises: tuple[LadderSpec, ...]`; a `session.json` with
  `identity`, `rungs` and `challenge` per exercise;
  `cli.ordered(spec: LadderSpec) -> LadderSpec`.

- [ ] **Step 1: Write the failing tests**

```python
# tests/test_session.py

def test_a_ladder_round_trips(tmp_path: Path) -> None:
    written = session.write(tmp_path, _ladder_session(), force=True)

    restored = session.read(written)

    assert restored == _ladder_session()


def test_rungs_are_recorded_resolved_not_as_diffs(tmp_path: Path) -> None:
    """Replay re-derives nothing (decision #14): it reads back what was drawn."""
    session.write(tmp_path, _ladder_session(), force=True)

    document = json.loads((tmp_path / "sessions" / "2026-08-18" / "session.json").read_text())
    rungs = document["exercises"][0]["rungs"]

    assert rungs[0] == {}
    assert set(rungs[1]) < set(rungs[2])
    assert rungs[2]["pattern"] == rungs[1]["pattern"]


def test_a_pre_ladder_record_is_refused(tmp_path: Path) -> None:
    """L9: clean reset. An old record fails loudly rather than being read as a
    one-rung ladder, because a degenerate ladder would weight every later draw
    against a day whose deviations were never recorded."""
    directory = tmp_path / "sessions" / "2026-08-01"
    directory.mkdir(parents=True)
    (directory / "session.json").write_text(json.dumps(_PRE_LADDER_DOCUMENT))

    with pytest.raises(session.SessionError, match=r"identity"):
        session.read(directory)
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `pytest tests/test_session.py -k "ladder or rungs" -v`

Expected: FAIL — `_document` writes `params`, not `identity`/`rungs`.

- [ ] **Step 3: Reshape the document**

`_document`'s exercise entries become:

```python
            {
                "family": spec.family,
                "identity": {axis: _jsonable(v) for axis, v in spec.identity.items()},
                "rungs": [
                    {axis: _jsonable(v) for axis, v in rung.items()} for rung in spec.rungs
                ],
                "challenge": (
                    None
                    if spec.challenge is None
                    else {axis: _jsonable(v) for axis, v in spec.challenge.items()}
                ),
            }
```

and `_exercise` reads them back through the existing `_axis_value`, so JSON's
arrays are restored to tuples — the serialization trap the module docstring
opens with applies unchanged to a rung's `permutation`.

`_EXERCISE_KEYS` becomes `("family", "identity", "rungs")`, which is what makes
a pre-ladder record fail by name.

- [ ] **Step 3b: Make `ordered` and `_replay` rung-aware**

`ordered` restores the drawn order across two levels now: identity axes in the
family's declared `AXES` order, then each rung's deviations in the order they
switched on. Keep its existing rule for an axis neither declares — kept at the
end, never dropped, because "dropping a recorded parameter to tidy an ordering
would be a silent loss of the thing being reproduced."

`_replay` maps the recorded ladder through `ladder.rungs_of` and
`ladder.materialize`, exactly as `_generate` does in Task 7 — the two must build
the same scores from the same record or replay is not a reproduction. Extract
that mapping into one helper both call rather than writing it twice; two private
answers to "what scores does this slot engrave" is the disagreement Task 4's
`rungs_of` docstring already argues against.

- [ ] **Step 3c: Write the §11 replay round-trip test**

```python
# tests/test_cli_query.py

def test_replay_reproduces_the_sheet_it_recorded(tmp_path: Path) -> None:
    """§11's round-trip. Replay re-derives nothing (decision #14) — it reads the
    record back, so the scores it builds must equal the ones that were written."""
    generated = _generate_session(tmp_path, on=datetime.date(2026, 8, 18))
    recorded = session.replay(tmp_path, datetime.date(2026, 8, 18))

    replayed = [
        cli._score(active, spec.family, ladder.materialize(spec, index))
        for spec in recorded.exercises
        for index in range(len(ladder.rungs_of(spec)))
    ]

    assert replayed == generated


def test_ordered_restores_identity_then_switch_on_order(tmp_path: Path) -> None:
    spec = cli.ordered(_alphabetised_ladder())

    assert list(spec.identity) == ["root", "scale_type", "traversal"]
    assert list(spec.rungs[2]) == ["pattern", "accent_pattern"]
```

- [ ] **Step 4: Add the module docstring paragraph**

The docstring is this module's argument, so the reshape needs its own paragraph:
why `challenge` is a separate key rather than the last element of `rungs` (a
reader should not need the configuration to identify the level++ exercise), and
why rungs are resolved rather than diffed (decision #14 — replay re-derives
nothing).

- [ ] **Step 5: Run the tests to verify they pass**

Run: `pytest tests/test_session.py tests/test_cli_query.py tests/test_cli_generate.py -v`

Expected: PASS.

- [ ] **Step 6: Validate and commit**

```bash
vrg-container-run -- vrg-validate
vrg-git add src/melete/session.py src/melete/cli.py tests/test_session.py \
    tests/test_cli_query.py
vrg-commit --type feat --scope session \
  --message "record a ladder per slot, and replay it (#87)" \
  --body "Exercises gain identity, rungs and challenge. Rungs are recorded
resolved rather than as diffs to be recomposed, per decision #14: replay
re-derives nothing, it reads back what was drawn. challenge is its own key so a
reader can identify the level++ exercise without consulting the configuration
that produced it.

cli.ordered and _replay land in the same commit rather than a later one. ordered
walks spec.params, which this change deletes, and both replay and show depend on
it — splitting them would merge a green suite over a broken replay. ordered now
restores two levels of order (identity in declared order, then each rung's
deviations in switch-on order) and _replay maps the record through the same
rungs_of/materialize helper _generate uses, because the two must build identical
scores or replay is not a reproduction.

No back-compatibility and no version key (L9): a pre-ladder record fails loudly
on the required-key check. The project is experimental, the logs live in
gitignored build/, and config_hash changes regardless.

Ref: mnemosys-project/.github#87"
```

---

### Task 9: Two-level numbering and rung titles

Spec §9.

**Files:**
- Modify: `src/melete/alphatab/emit.py:711-744` (`emit_book`)
- Modify: `src/melete/cli.py:397-415` (`_preview`), `:590` (`phrase`), `_show`,
  `_replay`
- Test: `tests/alphatab/test_emit.py`, `tests/test_cli_query.py`

**Interfaces:**
- Consumes: Task 7's `LadderSpec`, Task 8's records.
- Produces: `emit_book(scores, cover, numbers: Sequence[str])` — the caller
  supplies each exercise's label, so the emitter never learns what a ladder is.

- [ ] **Step 1: Write the failing tests**

```python
# tests/alphatab/test_emit.py

def test_book_sections_carry_two_level_numbers() -> None:
    text = emit.emit_book(scores, cover, numbers=["1.1", "1.2", "2.1"])

    assert '\\section "1.1.' in text
    assert '\\section "2.1.' in text
```

```python
# tests/test_cli_query.py

def test_a_rung_title_names_what_was_added() -> None:
    title = cli.rung_title("scales", identity, rung={"pattern": "groups_of_3"})

    assert title == "C Ionian — three-notes-per-string, groups of 3"


def test_the_challenge_rung_is_marked() -> None:
    assert cli.rung_title("scales", identity, rung=rung, challenge=True).startswith("⚡")
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `pytest tests/alphatab/test_emit.py tests/test_cli_query.py -k "number or rung" -v`

Expected: FAIL — `emit_book` takes no `numbers`, `cli.rung_title` does not exist.

- [ ] **Step 3: Give the emitter the labels rather than the concept**

`emit_book` takes `numbers` and uses `f"{numbers[i]}. {score.title}"` in place of
`f"{number}. {score.title}"`. The `systemsLayout` split is unchanged: it is
per-exercise already, so a ladder's rungs each start a fresh system for free.
Add one docstring sentence recording that the emitter is deliberately told the
label and not the structure — the renderer boundary means it must not learn what
a ladder is.

- [ ] **Step 4: Build the titles in the CLI**

`cli.rung_title` composes the identity phrase and the active deviations **in the
order they switched on** — which is why the rungs are cumulative dictionaries in
insertion order rather than sets. Reuse `vocabulary.display` and the existing
`phrase`; add no second display vocabulary.

- [ ] **Step 5: Extend `--dry-run`, `show` and `replay`**

`_preview` prints the slot, then each rung indented beneath it with its label and
the deviations active at it. A ladder must be inspectable before it is printed —
that is the point of a 25-exercise sheet.

- [ ] **Step 6: Run the tests to verify they pass**

Run: `pytest tests/alphatab/ tests/test_cli_query.py tests/test_cli_generate.py -v`

Expected: PASS. Per-family goldens are unaffected — a rung is an ordinary spec —
but the **book-level** golden changes, because every section title does. Re-freeze
it deliberately and review the diff rather than accepting it wholesale.

- [ ] **Step 7: Validate and commit**

```bash
vrg-container-run -- vrg-validate
vrg-git add src/melete/alphatab/emit.py src/melete/cli.py tests/
vrg-commit --type feat --scope emit \
  --message "number rungs two-level and title what they add (#87)" \
  --body "emit_book takes the labels rather than the concept: the emitter is
told '2.3' and never learns what a ladder is, which is what keeps the renderer
boundary where it is.

A rung title is the identity phrase plus its active deviations in the order they
switched on, so the page is its own account of what is being added. The
challenge rung is marked.

Ref: mnemosys-project/.github#87"
```

---

### Task 10: Spike — can alphaTab break a page at a ladder boundary?

Spec §9. A question, not a feature; the output is an answer and a
recommendation.

**Files:**
- Create: `docs/reports/alphatex-page-breaks.md`

- [ ] **Step 1: Establish what exists**

The emitter's only layout levers today are `\track { systemslayout … }` and
`\section` markers (`emit.py:171-183`). Read the alphaTab documentation and the
vendored `melete-render/` for any page-level directive, and check whether the
`.gp` format carries a page break alphaTab can be made to write.

- [ ] **Step 2: Try the cheapest thing that could work**

Generate a two-ladder book by hand, attempt a page break at the boundary, and
open the result. Timebox it — this is a spike, and its deliverable is the
answer.

- [ ] **Step 3: Write the report and recommend**

Following `docs/reports/` house style. If reachable, recommend a follow-up task
under epic #87 and describe the directive. If not, say so plainly: two-level
numbering carries the grouping, and the question is closed rather than deferred
indefinitely.

- [ ] **Step 4: Commit**

```bash
vrg-git add docs/reports/alphatex-page-breaks.md
vrg-commit --type docs --scope reports \
  --message "spike page breaks at ladder boundaries (#87)" \
  --body "Ref: mnemosys-project/.github#87"
```

---

### Task 11: Acceptance — generate a sheet and play it

An **operational (validation) task**, filed with `vrg-issue-create --kind
validation`, blocked by Tasks 7–9. It has no PR: it is run, and its result is
recorded as a comment.

**Precondition self-check:** `melete generate` completes against
`examples/config.toml` with `[ladder]` configured, and the `.gp` opens in Guitar
Pro. If it does not, comment "blocked: preconditions not met" and stop.

**Procedure:**

1. Generate a session with the shipped example configuration.
2. Confirm the sheet has `count × (rungs + 1)` exercises, numbered two-level.
3. For each slot: rung 1 is a shape the player can already play; each later rung
   adds exactly one *nameable* technique to the same shape; the last ordinary
   rung is comparable to what melete generated before this epic; the challenge
   rung is beyond current reach.
4. Confirm at least one tapped ladder is present and that **its rung 1 is
   one-handed** — the L11 regression, checked by eye at the stand rather than
   only in a unit test.
5. Play it.

**Acceptance:** the player can point at any rung and say what was added to the
rung before it. Record `Outcome: SUCCESS` or `Outcome: FAILURE` with specifics;
on failure the task stays open and the epic stays open.

---

## Self-Review

**Spec coverage.** §3 model → Tasks 3, 4. §4 declarations → Task 3. §5 draw →
Tasks 4, 7. §6 gate → Tasks 6, 7. §7 configuration → Task 5. §8 record → Task 8.
§9 sheet → Tasks 9, 10. §10 error handling → Tasks 4, 5, 7 (each row has a
refusal in one of them). §11 testing → the test steps throughout, with the four
spec-named tests landing in Tasks 1, 4, 7 and 8. §12 decisions: L1 Task 4
(`materialize`), L2–L3 Task 3, L4–L5 Task 3, L6 Tasks 4–5, L7 Tasks 3–4, L8
Task 5's example, L9 Task 8, L10 Task 6, L11 Task 1, L12 Tasks 3, 7, L13 Task 2,
L14 Task 4, L15 Task 5.

**Placeholders.** None: every code step carries the code, every test step the
test, and the two report tasks (6 and 10) define their deliverable and the
question it must answer rather than deferring it.

**Alignment pass (paad:alignment, 2026-08-19).** Four issues, all applied:

- §10's fourth row — a `[challenge.<family>]` naming no axis the family grades
  on — had no implementation, so it would have surfaced 500 resamples later as
  an over-constrained pool. Now a load-time check in Task 5
  (`_challenge_can_escalate`), with `_widest_identity` shared with
  `_ladder_fits_the_families`.
- `cli.ordered` walks `spec.params`, which Task 8 deletes, and no task named it.
  Task 8 now absorbs the whole replay path plus §11's round-trip test, rather
  than merging a green suite over a broken `replay`.
- The challenge rung picked `sorted(...)[0]`, escalating the alphabetically
  first axis every time — `accent_pattern` on nearly every `scales` page,
  contradicting §1's rotation goal on the one rung meant to be unfamiliar. The
  axis is now drawn on the `DEVIATION` axis like every other.
- `Ladder.plain` was a flat mapping, so `arpeggios` set `hands: 1` for triads —
  unrealizable until `melete#227`, and rejected-then-resampled rather than
  reported. `plain` is now identity-aware, matching `identity` and `eligible`,
  and the triad branch is the single line `melete#227` deletes.

**Type consistency.** `LadderSpec(family, identity, rungs, challenge)`,
`ladder.build(...) -> LadderSpec`, `ladder.materialize(spec, rung) -> dict`,
`ladder.rungs_of(spec) -> tuple[dict, ...]`, `ladder.DEVIATION`,
`ladder.ORDINARY`, `ladder.CHALLENGE`, `families.Ladder(identity, plain, tiers,
eligible)` — all three of `identity`, `plain` and `eligible` taking the drawn
identity — `Family.ladder`, `LadderConfig(rungs, challenge)`,
`Config.ladder`, `Config.challenge` — used identically in Tasks 3 through 9.
`derive(params, tapped)` keeps its signature throughout; only its contract
changes, in Task 1.

**Known gap, deliberate.** Task 6's outcome can add a twelfth task
(per-deviation repair). That is the point of running it before Task 7 rather
than after.
