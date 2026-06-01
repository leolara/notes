# leanSpec vs. consensus-specs — a framework comparison and the cost of closing the gap

## Executive summary

consensus-specs and leanSpec solve the same problem — an executable Ethereum spec
that also generates the inter-client conformance test vectors. Across the twelve
engineering dimensions below (type system, test framework, fork handling, build,
packaging, repository layout, …), leanSpec's framework is, on the evidence,
materially better engineered; each dimension is documented with line-level citations.

The facts that bear on strategy:

- The gap is rooted in a few foundational choices in consensus-specs — an untyped
  `(spec, state)` god object and a package whose source lives inside its own test
  tree — that are load-bearing for most of the other problems.
- Closing that gap *in place* is an interlocking, multi-quarter refactor carried out
  on a live, mainnet-critical artifact, where a behaviour-preserving slip is a
  consensus bug.
- The same end-state already exists, built and validated independently of mainnet:
  leanSpec's framework. Reaching it by building consensus-specs on that framework —
  greenfield, or by moving the framework into the consensus-specs repo — is a
  smaller, lower-risk body of work than the in-place rebuild, because the framework
  layers are already done and only the spec content and forks need porting.
- leanSpec's framework is also **more than a spec**: it is a runnable,
  interop-tested reference consensus node (networking, sync, validator, API,
  post-quantum crypto) — a capability consensus-specs has no counterpart for (see
  [Beyond the comparison](#beyond-the-comparison--leanspec-capabilities-with-no-consensus-specs-counterpart)).

This document lays out those facts and their costs. It does not prescribe a course
of action; it states what each side does and what each path would take, and leaves
the conclusion to the reader.

---

This document compares the **engineering infrastructure** of two Ethereum
specification codebases that solve the same problem — an executable reference spec
plus a conformance test-vector generator. The comparison is one-sided on the
evidence: leanSpec's framework is markedly better engineered across the dimensions
below. The document then quantifies what it would take to close that gap two ways —
repairing consensus-specs in place, or building consensus-specs on leanSpec's
framework — and sets the effort and risk of each side by side. (Written for EF
decision-makers: the strategic picture is in the executive summary, the synthesis,
and the closing cost analysis; the dimensions are the line-level evidence.)

It sits alongside the rest of the consensus-specs tech-debt audit in this directory
— the [README](README.md) and the deep-dives. That audit enumerates the debt and, in each
deep-dive, contrasts a single concern with how leanSpec and execution-specs
handle it. This document pulls those scattered contrasts into one head-to-head,
re-grounds every leanSpec citation in the **current** source tree (leanSpec has
been heavily refactored since the audit was written), and adds the dimensions
the audit only treated as cross-cutting.

The engineering vocabulary used throughout — *Shotgun Surgery*, *Primitive
Obsession*, *god module*, *tautological oracle*, the SOLID principles, and the
rest — is defined in plain English in [`reports/glossary.md`](glossary.md).

> **Scope and caveats.** leanSpec's framework is the better-engineered of the two
> on the dimensions below; this section records the constraints that do *not*
> explain the gap, so the reader can discount them. consensus-specs carries years of
> successive mainnet forks, multi-client coordination, and a deliberate
> markdown-first authoring choice; leanSpec is a young single-fork research spec. But
> most of the gap is not explained by age or scale — it is explained by *framework
> choices*. Those choices are portable in principle, though [the repair-cost
> section](#the-repair-cost-asymmetry) shows that retrofitting them into
> consensus-specs *in place* is a coordinated, multi-front change, not a backlog of
> independent PRs.

---

## Subjects under comparison

| | consensus-specs | leanSpec |
|---|---|---|
| Pinned commit | `90cee983d` (2026-05-05, `master`) | `d80e55d` (2026-05-30, `main`) |
| Spec authoring | Python embedded in Markdown, extracted at build time | Plain Python (Pydantic models) |
| Runtime distribution(s) | one: `eth-consensus-specs` | two: `lean-spec` + `lean-ethereum-testing` |
| Build entry point | 332-line `Makefile` | 142-line `Justfile` |
| Type checker | mypy (scoped to the generated spec packages only) | `ty`, error-on-warning, whole tree |
| Codegraph index (Python) | `state_transition` lives in `.md` → 0 nodes in the code graph | spec is Python → fully indexed (`src/` = 133 modules) |

The "codegraph index" row is itself a finding: a graph database (FalkorDB via
tree-sitter) indexes leanSpec's entire spec because it is Python, but cannot see
consensus-specs' `state_transition` — its canonical definition lives in Markdown
and the generated `.py` form is gitignored (un-built). It's a real indexing and
authoring-directness gap, but a narrow one: once built, the generated spec is
ordinary typed Python — in fact the part of the tree mypy *does* check (§6).

---

## Scorecard

Each row links to its full treatment. On every dimension leanSpec's approach is the
better-engineered one; the load-bearing content is the *mechanism* and the
*magnitude*, in the two middle columns.

| # | Dimension | consensus-specs | leanSpec |
|---|---|---|---|
| 1 | [Type system & validation](#1-type-system--validation) | Untyped `(spec, state)` god object; raw `int` for slot/epoch/index | Strict immutable Pydantic domain types; `Slot` ≠ `Epoch` at the type level |
| 2 | [Test framework](#2-test-framework) | 7+ order-sensitive stacked decorators; module-global BLS/preset state | One named fixture param via a pytest plugin; collector holds no global state |
| 3 | [Fork registration](#3-fork-registration) | Fork tree restated across ~11 files / 15+ edit sites in 3 formats | One `ForkRegistry`, one list append, validated at construction |
| 4 | [Fork-conditional logic](#4-fork-conditional-logic) | `is_post_<fork>` cascades in 10k-line shared helpers — **220 occurrences** | Polymorphic fork methods — **0** `is_post_*` in `src/` |
| 5 | [Build orchestration](#5-build-orchestration) | 332-line procedural Makefile + 7 workflows + bash/jq CI matrix | 142-line declarative Justfile + uv groups + 1 CI file, 5 jobs, `concurrency` |
| 6 | [Static analysis & quality gates](#6-static-analysis--quality-gates) | mypy reaches only the generated spec packages (~877 errors unchecked in helpers/pysetup/tests); 5 ruff families (more not yet enabled); no coverage gate; codespell/mdformat via Makefile | `ty` error-on-warning; 9 ruff families; coverage gate; `--strict-markers`; codespell; mdformat |
| 7 | [Packaging & public-API boundary](#7-packaging--public-api-boundary) | One kitchen-sink wheel; 5 generic top-level names; no `__all__` | Two distributions; project-prefixed names; curated `__all__` |
| 8 | [Test oracle & vector formats](#8-test-oracle--vector-formats) | Tautological round-trip oracle; ~35 vector-format specs (Markdown) | Typed Pydantic fixtures serialized to JSON by one filler |
| 9 | [Repository & package layout](#9-repository--package-layout) | Package source lives inside `tests/` → two import names (mypy "found twice", dual-mutation conftest); tests 8 levels deep; `pip install` needs `make` | Canonical `src/` layout; one import name per file; tests mirror source at ~4 levels; clean `pip install` |
| 10 | [Documentation & CLI](#10-documentation--cli) | MkDocs without mkdocstrings; CLI is Makefile targets | MkDocs + mkdocstrings (docstrings → API docs); Click CLI (`fill`, `apitest`) |
| 11 | [Engineering discipline](#11-engineering-discipline) | Conventions only; quality gates reach only the generated spec packages | Property tests, enforced coverage gate, NO-BACKCOMPAT / NO-ABBREVIATIONS rules, 6 machine-readable rule files |
| 12 | [Spec authoring](#12-spec-authoring) | Python in Markdown → `pysetup/` extracts gitignored `.py`; tooling-blind | Plain importable Python; no build step |

---

## 1. Type system & validation

- **consensus-specs:** untyped `(spec, state)` god object; slot, epoch and index are all raw `int`.
- **leanSpec:** strict, immutable Pydantic domain types — a `Slot` can't be passed where an `Epoch` is expected.
- **Net:** type errors surface at construction or import, not three layers deep at runtime.

**consensus-specs** threads an untyped `(spec, state)` pair through its entire
helper layer. The `Spec` Protocol declares two attributes and is imported in one
place; in practice `spec` is a bare namespace with ~200 attributes, and slot,
epoch, and validator index are all raw `int`
(`consensus-specs/tests/core/pyspec/eth_consensus_specs/test/helpers/attestations.py:21,40,86`).
And the type checker barely sees these signatures: `MYPY_SCOPE` analyses only the
generated spec packages, so the helper layer's untyped `(spec, state)` parameters
are never checked at all, and `ignore_missing_imports = true`
(`consensus-specs/pyproject.toml:64-70`) lets the SSZ types they carry — imported
from untyped third-party libraries — pass as `Any` (it applies to stub-less
imports; it does not by itself disable the strict directives). This is Primitive
Obsession (Fowler) at module scale; see
[helper-layer](deep-dives/helper-layer.md).

**leanSpec** defines narrow, immutable, strictly-validated domain types. The root
model forbids unknown fields and refuses implicit coercion:

```python
# leanSpec/src/lean_spec/base.py:48
class StrictBaseModel(CamelModel):
    model_config = CamelModel.model_config | {"extra": "forbid", "strict": True}
```

Scalars range-check at construction and reject look-alikes (`bool` is not an
`int` here), and domain types are distinct classes carrying their own behaviour:

```python
# leanSpec/src/lean_spec/spec/ssz/uint.py:22   (BaseUint.__new__)
if not isinstance(value, int) or isinstance(value, bool):
    raise SSZTypeError(f"Expected int, got {type(value).__name__}")
...
if not (0 <= int_value <= max_value):
    raise SSZValueError(f"{int_value} out of range for {cls.__name__} [0, {max_value}]")
```

`Slot` and `ValidatorIndex` are separate `Uint64` subclasses
(`leanSpec/src/lean_spec/spec/forks/lstar/slot.py`,
`.../lstar/containers.py:46`), so a function that takes a `Slot` cannot be called
with an `Epoch` — the type checker rejects it.

**Why it matters.** Strict typing as a force multiplier. A `Slot` cannot
masquerade as an `Epoch`; a malformed field cannot construct; a renamed attribute
fails at import, not three layers deep at runtime. consensus-specs announces
strict typing and then disables it; leanSpec makes it real and gets it enforced
by `ty` (see §6).

*Already in the framework:* typed domain types (`Slot`, `Bytes32`, `ValidatorIndex`) and typed fork classes in place of an untyped `(spec, state)` god object — so the helper→spec boundary is checked, not `Any`.

---

## 2. Test framework

- **consensus-specs:** 7+ stacked, order-sensitive decorators per test; ordering rules live only in prose comments; module-global BLS/preset state.
- **leanSpec:** one fixture class injected by name via a pytest plugin; fork validity declared by markers; no module-level state.
- **Net:** no decorator-ordering footguns; authoring a test is one parameter, not a five-layer stack.

**consensus-specs** stacks decorators on every spec test —
`@with_phases([...])`, `@spec_state_test`, `@with_state`, `@with_meta_tags`,
`@with_presets`, `@with_config_overrides`, `@always_bls` — whose *ordering* is
load-bearing but documented only in prose comments
(`consensus-specs/tests/core/pyspec/eth_consensus_specs/test/context.py:342-345,766-798`).
The wrong order silently leaks BLS state across tests or drops config overrides.
Decorator and fixture behaviour is steered by mutable module-level globals
(`DEFAULT_TEST_PRESET`, `bls.bls_active`) rebound by autouse fixtures. Full
treatment: [decorator-stack](deep-dives/decorator-stack.md).

**leanSpec** injects a fixture *class by name* through a pytest plugin. A test
declares one parameter and applies one marker:

```python
# leanSpec/tests/consensus/lstar/fc/test_attestation_target_selection.py
pytestmark = pytest.mark.valid_until("Lstar")

def test_attestation_target_at_genesis_initially(
    fork_choice_test: ForkChoiceTestFiller,
) -> None:
    fork_choice_test(steps=[...])
```

The plugin's `FixtureCollector` carries no module-level mutable state
(`leanSpec/packages/testing/src/framework/pytest_plugins/filler.py:15`), and fork
applicability is declared with markers (`valid_from` / `valid_until` /
`valid_at`) evaluated at collection time, not with an imperative decorator chain.

**Why it matters.** Dependency injection and "lean on pytest." Fork selection
is markers (pytest's own mechanism); state setup is fixtures; vector emission is a
collector. The seven-decorator stack, the prose ordering rules, and the rebind-
the-global autouse fixtures all collapse into framework-standard parts. There are
no order-sensitivity footguns because there is no stack to order.

*Already in the framework:* a pytest-plugin filler with named fixtures and fork-validity markers — no seven-decorator stack and no module-global BLS/preset state.

---

## 3. Fork registration

- **consensus-specs:** the fork tree is restated across ~11 files / 15+ sites in three formats, cross-checked nowhere.
- **leanSpec:** one `ForkRegistry` plus one list append, validated (monotonic version, unique name) at construction.
- **Net:** adding a fork is one edit, not a shotgun with silent partial-registration failure modes.

**consensus-specs** reconstructs the fork tree ad hoc in roughly **11 files at
15+ edit sites in three formats**: a bare-string list in `pysetup/constants.py`,
a genealogy dict `PREVIOUS_FORK_OF` in `pysetup/md_doc_paths.py:16-27`, a builder
tuple in `pysetup/spec_builders/__init__.py`, plus `Makefile` variables,
`.gitignore` blocks, CI matrices, and labeler config. None is derived from
another, nothing cross-checks them, and the genealogy dict is validated nowhere —
a typo introducing a cycle recurses forever at runtime. Adding one fork is a
manual, error-prone ritual; forgetting one site fails silently. Full treatment:
[fork-registration](deep-dives/fork-registration.md).

**leanSpec** has one registry that validates itself at construction:

```python
# leanSpec/src/lean_spec/spec/forks/registry.py:11
def __init__(self, forks: list[ForkProtocol]) -> None:
    if not forks:
        raise ValueError("ForkRegistry requires at least one fork")
    versions = [f.VERSION for f in forks]
    if any(a >= b for a, b in pairwise(versions)):
        raise ValueError(f"Forks must be ordered by strictly increasing VERSION: {versions}")
    names = [f.NAME for f in forks]
    if len(set(names)) != len(names):
        raise ValueError(f"Fork names must be unique: {names}")
```

Adding a fork is one import and one list append:

```python
# leanSpec/src/lean_spec/spec/forks/__init__.py:35
FORK_SEQUENCE: list[ForkProtocol] = [LstarSpec()]
DEFAULT_REGISTRY: ForkRegistry = ForkRegistry(FORK_SEQUENCE)
```

Identity and genealogy live with the fork as typed class attributes
(`NAME`, `VERSION`, `previous`) on the concrete `LstarSpec(ForkProtocol)`
(`.../lstar/spec.py:58-77`).

**Why it matters.** Open/Closed plus DRY. The fork tree has a single canonical
home; monotonic ordering and name-uniqueness are checked at startup, so partial
or contradictory registration cannot exist. consensus-specs is declarative-by-
ritual across a dozen files; leanSpec is declarative-by-structure in one.

*Already in the framework:* a runtime-validated `ForkRegistry` where adding a fork is one class plus one list append — not a shotgun across ~11 sites.

---

## 4. Fork-conditional logic

- **consensus-specs:** `is_post_<fork>` predicate cascades in 10k-line shared helpers — **220 occurrences**.
- **leanSpec:** polymorphic per-fork classes — **0** `is_post_*` in `src/`.
- **Net:** a new fork adds a subclass instead of editing predicate chains across god modules.

This is the most directly quantifiable contrast in the comparison: a single `grep`
returns 220 vs. 0.

**consensus-specs** centralises fork-specific behaviour in a ~10,000-line shared
helper layer and gates it with `is_post_<fork>(spec)` predicate cascades. A
direct grep is unambiguous:

> `is_post_*` predicates in `consensus-specs/tests/core/pyspec/.../test/helpers/`:
> **220 occurrences.**
> `is_post_*` anywhere in `leanSpec/src/`: **0.**

A single helper interleaves two fork algorithms behind five `is_post_altair`
branches in 67 lines (`.../test/helpers/rewards.py:232-298`); fork identity is an
unvalidated string (`SpecForkName = NewType('SpecForkName', str)`). Adding a fork
edits these shared files; renaming a `spec` attribute fails only at runtime. Full
treatment: [helper-layer](deep-dives/helper-layer.md).

**leanSpec** makes each fork a class that owns its behaviour as typed methods.
`LstarSpec` carries 29 typed methods (a `def` count over the class) — `process_slots`, `process_block`,
`process_attestations` (`.../lstar/spec.py:351`), `state_transition`, `on_block`,
`on_tick`, and so on — and the registry returns a fork object the caller invokes
polymorphically. There is no `spec` god object to gate, hence zero predicates.

> An honest note on size: `lstar/spec.py` is itself ~1,890 lines. The difference
> is not "small file vs. big file" — it is that those lines are **one fork's
> behaviour, fully typed and fork-local**, extended by *adding* a subclass, versus
> a **shared, untyped layer threaded through every fork** by runtime predicates and
> edited in place for each new fork.

**Why it matters.** Replace Conditional with Polymorphism (Fowler) and
Open/Closed (Meyer): a new fork is a new class overriding only what changed, not
220 predicate edits across a god module. Type checking catches a renamed method
at import; in consensus-specs the same mistake is a latent runtime failure in
whichever helper runs first.

*Already in the framework:* polymorphic per-fork classes (`ForkProtocol` plus concrete forks) — fork behaviour is a method, not an `is_post_<fork>` cascade through shared helpers.

---

## 5. Build orchestration

- **consensus-specs:** a 332-line procedural Makefile + 7 GitHub workflows + a bash/jq-built CI matrix.
- **leanSpec:** a 142-line declarative Justfile + uv dependency-groups + one concurrency-aware CI file.
- **Net:** one recipe per concern, lint (check) split from fix (mutate), one toolchain shared by contributors and CI.

**consensus-specs** drives development through a 332-line procedural `Makefile`
whose `test:` target exposes nine user-facing knobs via a 15-variable
substitution cascade (`consensus-specs/Makefile:201-220`), and whose `lint:`
target mixes mutating steps (`ruff --fix`, `ruff format`) with read-only ones,
so CI cannot tell "passed clean" from "passed after reformatting"
(`Makefile:276-293`). The CI matrix is built dynamically with bash + jq
(`.github/workflows/comptests.yml:60-170`), the `setup-python`/`setup-uv`
bootstrap is duplicated across the workflow files, and `pyproject.toml` has no
`[tool.uv]` section to constrain a contributor's local toolchain. Full treatment:
[build-orchestration](deep-dives/build-orchestration.md).

**leanSpec** uses a 142-line declarative `Justfile` with recipes grouped by
concern and composed by reference, and it splits read-only checks from mutating
fixes *structurally*:

```make
# leanSpec/Justfile
check: lint format-check typecheck spellcheck mdformat lock-check   # all read-only

lint *args:
    uv run --group lint ruff check --no-fix --show-fixes "$@"

fix:                                                                # separate, mutating
    uv run --group lint ruff check --fix
    uv run --group lint ruff format
```

`pyproject.toml` pins the toolchain for contributors and CI alike — `[tool.uv]
required-version = ">=0.7.0"`, `[tool.uv.workspace]`, and PEP 735
`[dependency-groups]` (`test`, `lint`, `docs`, `dev` with `include-group`
aggregation) (`leanSpec/pyproject.toml:133-175`). CI is a single
`ci.yml` with five jobs (lint, test, coverage-gate, fill-tests, interop), a
`concurrency: cancel-in-progress` block, and a clean `3.12/3.13/3.14 × ubuntu/macos`
matrix that simply calls `just check` / `just test` (`leanSpec/.github/workflows/ci.yml:13-77`).

**Why it matters.** Configure-don't-integrate and Single Responsibility. One
recipe per concern, composed; one declarative dependency manifest shared by local
and CI; `concurrency` cancels stale runs. consensus-specs encodes the same intents
as a procedural Makefile cascade plus duplicated workflow boilerplate.

*Already in the framework:* a declarative `Justfile`, uv dependency-groups, and a concurrency-aware single CI file — not a 332-line Makefile to untangle.

---

## 6. Static analysis & quality gates

- **consensus-specs:** mypy reaches only the generated spec packages — **~877 errors** in helpers/`pysetup`/tests never reach CI; 5 ruff families (turning on the 9 leanSpec runs surfaces **~21,000 violations**, ≈17,000 missing-docstring); no coverage gate.
- **leanSpec:** `ty` error-on-warning over the whole tree; 9 ruff families; `--strict-markers`; an enforced coverage gate.
- **Net:** the gates reach all the code by default, not a handful of generated files.

**consensus-specs** has the right tools but narrow reach. Its strict mypy
directives (`disallow_untyped_defs`, `warn_unused_ignores`) apply to almost no
code, because `MYPY_SCOPE` (`consensus-specs/Makefile:272-273`) checks only the
generated per-fork spec packages — the 10k-line helper layer, `pysetup/`,
`scripts/`, and `tests/infra/` are never checked, so a full-tree run surfaces
**~877 type errors across ~98 files** that never reach CI (the cross-cutting
"typing deficit" section of [`reports/README.md`](README.md)). Within the
slice that *is* checked, `ignore_missing_imports = true`
(`consensus-specs/pyproject.toml:64-70`) makes the untyped SSZ types (from
remerkleable, which thread through nearly everything the tests touch) resolve to
`Any`, so even that slice is verified only thinly — and the same `Any` would
pervade the helpers if the scope ever reached them. (It does not *literally*
rewrite the strict directives, as a maintainer noted on the audit PR — but with the
core types untyped, the effective coverage is thin.) Ruff enables five families and silences nine
complexity rules; bringing the tree up to the nine families leanSpec runs would
fire ~21,000 times today (≈17,000 of them missing-docstring `D` — essentially every
function and class), so adopting them is a staged migration, not a config flip
(counts measured for the audit, PR #5233). `codespell` and
`mdformat` do run, but via the `Makefile` (`Makefile:280-287`) rather than as
configured, declarative recipes, and pytest registers no markers,
`--strict-markers`, or coverage gate. The gates are real; their *reach* is the
gap. Full treatment:
[static-analysis-config](deep-dives/static-analysis-config.md).

**leanSpec** makes the gates real and enforced:

- **Type checking:** `ty` with `error-on-warning = true` over the tree
  (`leanSpec/pyproject.toml:83-90`).
- **Lint:** nine ruff families `["E","F","B","W","I","A","N","D","C"]` with
  targeted, mostly per-file ignores rather than global silencing
  (`pyproject.toml:66-69`).
- **Tests:** `--strict-markers`, `--cov=src --cov-branch`, and all custom markers
  registered (`pyproject.toml:92-117`).
- **Coverage gate:** `[tool.coverage.report] fail_under = 90` (`pyproject.toml:126-127`),
  with a dedicated CI job running `just test-cov-gate` (`--cov-fail-under=80`)
  (`ci.yml:84-108`, `Justfile:75-76`). *(The two thresholds are inconsistent — the
  config default is stricter than the CI gate — but a gate is genuinely enforced
  on every PR, which is the point of contrast.)*
- **Prose:** `codespell` with a custom ignore-words list and `mdformat`, both run
  in `just check`.

**Why it matters.** Reach, not just intent. `ty` checks the whole tree under
`error-on-warning` because the spec is Python (§12) and nothing scopes it down to a
handful of files; the broad ruff selection, the registered markers, and the
coverage gate apply everywhere by default. consensus-specs has the same classes of
tools — mypy, ruff, codespell, mdformat — but their reach stops at the generated
spec packages, so most of the code the audit flags is never inspected.

*Already in the framework:* `ty` error-on-warning over the whole tree, a nine-family ruff selection, `--strict-markers`, and an enforced coverage gate — gates that reach all the code by default.

---

## 7. Packaging & public-API boundary

- **consensus-specs:** one kitchen-sink wheel; 5 generic top-level names; no `__all__`.
- **leanSpec:** two distributions (runtime + testing) with project-prefixed names and a curated `__all__`.
- **Net:** a consumer can install just the spec, and the public surface is declared rather than accidental.

**consensus-specs** ships a single distribution, `eth-consensus-specs`, that
mixes runtime spec, the test suite, the 10k-line test-helper layer, and debug
tooling under **five top-level names** — `eth_consensus_specs`, plus the generic
`configs`, `presets`, `specs`, `sync` (`consensus-specs/setup.py:10-28`; the
installed wheel's `top_level.txt` confirms all five). There is no `__all__` and no
docstring on the package root, so by Hyrum's Law every reachable path is a de-facto
public API, and the generic names collide with any other distribution. A downstream
consumer cannot install just the spec. Full treatment:
[package-export-boundary](deep-dives/package-export-boundary.md).

**leanSpec** ships **two** distributions with a clean dependency direction:

- `lean-spec` (runtime) — one project-prefixed top-level name `lean_spec`,
  wheel scoped to `packages = ["src/lean_spec"]` (`leanSpec/pyproject.toml:5-8,57-58`).
- `lean-ethereum-testing` (framework + CLI) — its own project, depending on
  `lean-spec`, exporting the `fill` / `apitest` entry points
  (`leanSpec/packages/testing/pyproject.toml:5-8,22-38`).

The testing package curates its surface with an explicit `__all__`
(`packages/testing/src/consensus_testing/__init__.py`), and the runtime's
`forks/__init__.py` likewise exports a curated `__all__`.

**Why it matters.** Single Responsibility at the distribution boundary and a
correctly-directed dependency (framework → runtime, never the reverse). Explicit
`__all__` declares what is stable; project-prefixed names avoid namespace
pollution. consensus-specs publishes one wheel where everything is public and the
namespace is squatted with generic nouns.

*Already in the framework:* two clean distributions (runtime plus testing) with project-prefixed names and a curated `__all__` — not one kitchen-sink wheel.

---

## 8. Test oracle & vector formats

- **consensus-specs:** SSZ-generic tests use the library as its own oracle (round-trip only); ~35 markdown-described vector formats.
- **leanSpec:** typed Pydantic fixtures serialized to JSON by one filler.
- **Net:** the format is a Python type instead of 35 README docs, and validity is defined independently of the codec.

**consensus-specs** SSZ-generic tests use the implementation as its own oracle:
the only assertion is round-trip equality `deserialize(serialize(x)) == x`, and an
"invalid" case is whatever the library happens to reject
([ssz-generic-vectors](deep-dives/ssz-generic-vectors.md), §83-129). That
proves internal self-consistency, not conformance to the spec — and since these
vectors *are* the inter-client conformance spec, a bug in the Python library
propagates to every consumer. Vector layouts are described across ~35 vector-format specifications in
`tests/formats/` (22 README files among 53 Markdown docs) that mostly restate one of two underlying patterns
([vector-formats](deep-dives/vector-formats.md)).

**leanSpec** makes the fixture a typed Pydantic model and serializes it through a
single filler. The base fixture owns serialization:

```python
# leanSpec/packages/testing/src/framework/test_fixtures/base.py:60
@cached_property
def json_dict(self) -> dict[str, Any]:
    """Return the JSON representation of the fixture."""
    return self.to_json(exclude_none=True, exclude={"info"})
```

Each test family is a typed subclass (e.g. `StateTransitionTest` with
`pre: State`, `blocks: list[BlockSpec]`, `post: StateExpectation | None` —
`.../consensus_testing/test_fixtures/state_transition.py:26`), and one
`FixtureCollector` discovers, fills, and writes every format
(`.../framework/pytest_plugins/filler.py:15`).

**Why it matters.** The format is a Python type, not 35 Markdown documents;
the two underlying patterns are one base class each, not many copies (DRY); and
the oracle is decoupled from the codec — validity is defined by typed validators,
not "whatever the library accepts." A format change is a model edit, not an
N-place README rewrite.

*Already in the framework:* typed Pydantic fixture models serialized to JSON by one filler — a single type hierarchy in place of ~35 markdown format specs.

---

## 9. Repository & package layout

- **consensus-specs:** package source lives inside `tests/` → reachable under two import names → mypy can't run, `pip install` needs `make`, a dual-mutation conftest; tests sit 8 path components deep.
- **leanSpec:** canonical `src/` layout; one import name per file; tests mirror source; clean `pip install`.
- **Net:** the deepest structural root — it blocks typing the helpers and forces a v2.0-scale move to fix.

**consensus-specs** makes the package depend on itself. The runtime package
`eth_consensus_specs` is declared with its source *inside the test tree* (`setup.py`
maps it to `tests/core/pyspec/eth_consensus_specs/`), and the repo imports it under
that installed name while `pythonpath = ['.']` (`consensus-specs/pyproject.toml:61`)
*also* exposes it under the long `tests.core.pyspec.…` path. The same files are now
reachable under **two dotted names**, so Python loads them as two module objects
with two copies of every global — which is why a `conftest` fixture has to mutate a
preset on *both*
(`tests/core/pyspec/eth_consensus_specs/test/conftest.py:113-120`) and why `mypy`
aborts before checking anything: `Source file found twice under different module
names` (`self-referential-package-layout.md:111-119`). On top of that, a spec test
sits eight path components deep —
`tests/core/pyspec/eth_consensus_specs/test/phase0/sanity/test_blocks.py` — with
helpers and infrastructure as siblings, and `pip install .` doesn't yield a working
package because the spec modules don't exist until `make _pyspec` generates them.
Full treatment:
[self-referential-package-layout](deep-dives/self-referential-package-layout.md)
and [directory-structure](deep-dives/directory-structure.md).

**leanSpec** uses the canonical `src/` layout, so none of this can arise. Three
shallow top-level buckets — `src/lean_spec/` (runtime), `packages/testing/`
(framework), `tests/` — keep the package source and the tests in *different*
directories, so every file is reachable under exactly one import name. Tests
*mirror* the source at ~4 levels (`tests/lean_spec/spec/ssz/test_uint.py` mirrors
`src/lean_spec/spec/ssz/uint.py`) and import from the installed package
(`from lean_spec.spec.ssz import Uint64`), not from test-tree siblings; the wheel
points at `src/lean_spec` with no `pythonpath` hack
(`leanSpec/pyproject.toml:57-58`).

**Why it matters.** This is the deepest structural difference. The `src/` layout
gives every file one identity, so there is no dual-module hazard, no `pythonpath`
workaround, a working `pip install`, and standard-tool support (mypy, IDEs,
coverage, the code graph) by default — whereas the self-referential layout is
exactly what makes `mypy` unrunnable on the helpers and forces a "v2.0"-scale move
to fix (see the [repair-cost section](#the-repair-cost-asymmetry)).
Separation of concerns also makes "where does my test go?" answer itself from the
mirror.

*Already in the framework:* the canonical `src/` layout — package source separate from tests, one import name per file, a working `pip install`, and no `pythonpath` hack or dual-mutation conftest.

---

## 10. Documentation & CLI

- **consensus-specs:** MkDocs without mkdocstrings; commands are Makefile targets, not a CLI.
- **leanSpec:** MkDocs + mkdocstrings (docs generated from docstrings) and a typed Click CLI (`fill`, `apitest`).
- **Net:** documentation and command help live once, in the code.

**consensus-specs** runs MkDocs without mkdocstrings, so there is no API
documentation generated from docstrings (the spec is Markdown anyway — §12), and
`serve_docs` requires a manual `_pyspec` build first
(`consensus-specs/Makefile:261-263`). Its "CLI" is a set of Makefile targets
(`make _pyspec` invoking `python -m pysetup.generate_specs`,
`Makefile:185-188`); `setup.py` defers spec generation to the Makefile, so
`pip install -e .` alone does not produce a working install.

**leanSpec** pairs MkDocs with **mkdocstrings**, auto-generating API docs from
Google-style docstrings (`leanSpec/mkdocs.yml:36-47`), and exposes a typed Click
CLI wired as console scripts:

```toml
# leanSpec/packages/testing/pyproject.toml:36
[project.scripts]
fill    = "framework.cli.fill:fill"
apitest = "framework.cli.apitest:apitest"
```

`fill` is a Click command with typed, documented options (`--fork`, `--layer`,
`--clean`, `--scheme` — `.../framework/cli/fill.py:12`), and the node entry point
is a structured `cli/` package validating arguments through a Pydantic model
(`leanSpec/src/lean_spec/cli/main.py`).

**Why it matters.** Because the spec is Python, docstrings are first-class
documentation surfaced automatically; because the CLI is Click, each command is a
typed function with generated help, not a Makefile target parsing shell strings.
Documentation and command help each live once, in the code.

*Already in the framework:* MkDocs + mkdocstrings API docs and a typed Click CLI (`fill`, `apitest`) — documentation generated from docstrings, commands as functions.

---

## 11. Engineering discipline

- **consensus-specs:** no property-based tests, no enforced coverage gate, no machine-readable contributor rules.
- **leanSpec:** Hypothesis property tests, an enforced coverage gate, and six `.claude/rules/` files.
- **Net:** discipline is encoded and checkable, not re-negotiated per PR.

Several leanSpec strengths have no consensus-specs counterpart at all; they are
*enforced* rather than left to convention.

- **Property-based testing** with Hypothesis — e.g. `@given(...)` generating 100
  random slots to check a time invariant
  (`leanSpec/tests/lean_spec/spec/forks/lstar/forkchoice/test_time_management.py`).
  consensus-specs has none.
- **Unit-tested framework and types** — `tests/` mirrors `src/` (220 test files),
  so the spec's types and the test framework are themselves covered; consensus-specs
  has **zero** unit tests for its 10,000-line helper layer and **zero** for the
  `pysetup/` extractor that produces the entire spec (the cross-cutting section of
  [`reports/README.md`](README.md)).
- **An enforced coverage gate** in CI (§6), not an aspirational metric.
- **A strict NO BACKWARD COMPATIBILITY rule** — no shims, aliases, or deprecated
  re-exports; old patterns are deleted and all call sites updated
  (`leanSpec/CLAUDE.md:23-30`).
- **A strict NO ABBREVIATIONS IN IDENTIFIERS rule** — `att → attestation`,
  `sig → signature`, `idx → index` — so a reference spec reads unambiguously
  (`leanSpec/CLAUDE.md:31-57`).
- **Six machine-readable contributor rule files** under `leanSpec/.claude/rules/`
  (`code-style.md`, `documentation.md`, `test-framework.md`, `testing-style.md`,
  `ssz-patterns.md`, `workflow.md`) that encode the standards above as checkable
  guidance, e.g. "one sentence per line," "full equality assertions, never
  per-field."

By contrast, consensus-specs' quality gates reach only a handful of generated spec files (§6),
and the audit's cross-cutting sections record no shared contract on backward
compatibility, naming, or test structure — so each reviewer re-negotiates these on
every PR.

**Why it matters.** Every engineering decision has a single canonical home and,
where possible, an automated check. Discipline is configured, not tribal; the
machine, not the reviewer, enforces it.

*Already in the framework:* Hypothesis property tests, an enforced coverage gate, and machine-readable contributor rules (`.claude/rules/`) — discipline encoded, not left to convention.

---

## 12. Spec authoring

- **consensus-specs:** the spec is Python embedded in Markdown, extracted by `pysetup/` into gitignored `.py`.
- **leanSpec:** the spec is plain importable Python — no extraction or build step.
- **Net:** the most *arguable* item — Markdown has real readability value, and the cost is tooling indirection, not (as is sometimes claimed) an untyped spec.

Listed last, and with the lightest weight — this is the most *arguable* item in
the comparison. Many people value the Markdown spec precisely because it reads as
prose, and that readability is a real, deliberate benefit; unlike the structural
items above, the cost here is to *tooling*, not to readers. It compounds a few of
the issues already covered, but it is not the root of consensus-specs' debt, and
reasonable engineers disagree on whether it is debt at all.

**consensus-specs** authors the executable specification as Python inside fenced
code blocks in Markdown (`consensus-specs/specs/phase0/beacon-chain.md` — the
`state_transition` function is defined at `:1370`). A custom build step,
`pysetup/`, parses the Markdown and assembles `.py` files into
`tests/core/pyspec/eth_consensus_specs/<fork>/`, which are **gitignored**
(`consensus-specs/.gitignore:18-27`). The parser matches a heading against a
class name by string equality (`pysetup/md_to_spec.py:177`) and concatenates
fork-builder fragments as strings before re-parsing them
(`pysetup/helpers.py:47-258`). Full treatment:
[markdown-as-source-of-truth](deep-dives/markdown-as-source-of-truth.md).

The costs are real but narrower than they first look. The spec is authored in
Markdown but *checked* in its generated form: `pysetup` emits per-fork `.py` files,
and those generated packages are in fact exactly what `mypy` checks (§6) — so the
spec is **not** the untyped part of the tree. What's lost is *directness*: you edit
Markdown and get no type or IDE feedback on it; the checked artifact is a generated
copy you don't author; the generated runtime is gitignored, so a fresh checkout (and
a `.py` grep, and the code graph) can't find `state_transition` until `make _pyspec`
runs; `pip install .` doesn't yield a working package without it; and the extractor —
the single most load-bearing piece of tooling in the repo — has no characterisation
tests.

**leanSpec** authors the spec as ordinary, importable Python. There is no
extraction step and no generated runtime; the build backend is stock hatchling
with nothing custom (`leanSpec/pyproject.toml:1-3`). A core state-transition
routine is a normal, fully-annotated method:

```python
# leanSpec/src/lean_spec/spec/forks/lstar/spec.py:122
def process_slots(self, state: State, target_slot: Slot) -> State:
    """Advance the state through empty slots up to, but not including, target_slot."""
    assert state.slot < target_slot, "Target slot must be in the future"
    state = copy.deepcopy(state)
    while state.slot < target_slot:
        needs_state_root = state.latest_block_header.state_root == Bytes32.zero()
        ...
        state.slot = Slot(state.slot + Slot(1))
    return state
```

**Why it matters.** Separation of concerns plus Dependency Inversion: the
spec is the source, not an artifact derived from prose by a bespoke tool. Every
type checker, IDE, doc generator, and graph indexer sees the same file the author
wrote. consensus-specs couples documentation and code so tightly that it must
*generate* the code, and pays for that coupling at every tooling layer.

*Already in the framework:* the spec as ordinary importable Python, directly tool-legible. This is the one dimension where a framework-based path also gives up Markdown's prose readability — a genuine, separate trade-off the team can weigh on its own.

---

## Beyond the comparison — leanSpec capabilities with no consensus-specs counterpart

The twelve dimensions above are head-to-head. Some of leanSpec's strengths have
nothing to compare against, because consensus-specs is a *specification plus a
test-vector generator* while leanSpec is **also a runnable reference client**. That
is partly a scope choice — consensus-specs deliberately specifies behaviour and
leaves implementation and cryptography to the client teams — so these are *additive*
capabilities, not a quality gap. They matter here because adopting leanSpec's
framework brings them along.

- **An executable reference consensus node — almost a full client.** `node/` is
  **18,644 LoC** (≈63% of the spec). `python -m lean_spec` boots a real node
  (`src/lean_spec/cli/run.py`): a from-scratch p2p stack — QUIC transport, gossipsub
  with mesh/mcache/heartbeat, req/resp, ENR (`node/networking/`, **9,770 LoC**) — plus
  a sync service (`node/sync/`, 2,381), validator duties (`node/validator/`, 1,095),
  a beacon API (`node/api/`, 542), persistent storage (`node/storage/`, 978), a
  snappy codec (1,969), and Prometheus metrics (436). It ships a `Dockerfile` and a
  `docker.yml` workflow. consensus-specs has **no** runnable node; its "networking"
  is the p2p-interface *specification in Markdown*, not an implementation.
- **Multi-node interop testing.** `tests/interop/test_consensus_lifecycle.py` runs a
  **three-node cluster** through a real consensus lifecycle — gossip, block
  production, attestation flow — under `just interop` in CI. With no node,
  consensus-specs has nothing equivalent.
- **In-repo cryptography, including post-quantum.** leanSpec implements its own
  crypto: hash-based **XMSS / Winternitz** one-time signatures
  (`spec/crypto/xmss/`, **2,111 LoC** — post-quantum), Poseidon2, the KoalaBear
  field, and SNARK-based multi-signature aggregation (the Rust `lean-multisig-py`).
  consensus-specs specifies signatures abstractly and pulls BLS in through external
  bindings; it implements no signature scheme, and nothing post-quantum or
  SNARK-based.
- **A beacon API server + an API-conformance harness.** `node/api/endpoints/` serves
  states, fork-choice, checkpoints, aggregator, health and metrics endpoints, and the
  `apitest <server_url>` command (`packages/testing`) runs API-conformance tests
  against an external client. consensus-specs serves no API and has no such runner.

In fairness, this is a *research-grade* client: a single fork (`lstar`), Python (not
performance-tuned), consensus-layer only (no execution layer yet). It is "almost a
full client," not a production one. But the capability — a spec that is
simultaneously an executable, interop-tested, containerised reference node with its
own post-quantum cryptography — is real, and consensus-specs has no counterpart to
any of it.

---

## Synthesis — why the gap is structural, not incidental

The twelve dimensions are not twelve independent problems. Most of the
consensus-specs debt traces to a small number of **root framework choices**, and
each of leanSpec's counter-choices dissolves a whole *class* of downstream debt:

- **An untyped `(spec, state)` god object** (§1) is what *forces* the 10k-line
  helper layer and its 220 `is_post_*` predicates (§4). leanSpec's typed domain
  types and fork classes remove both the clump and the cascade together.
- **Decorator-based test configuration** (§2) and **scattered, unvalidated fork
  registration** (§3) are the same failure — knowledge with no canonical home,
  enforced by prose and ritual. leanSpec replaces both with framework-standard
  mechanisms (markers, fixtures, a validated registry) that fail loudly.
- **One kitchen-sink wheel** (§7) and **a package that lives inside its own test
  tree** (§9) are the absence of a separation that leanSpec draws once, cleanly,
  between runtime and testing.
- **Authoring the spec in Markdown** (§12) compounds a couple of these — the
  gitignored generated runtime and the missing API-doc site (§10) — by putting the
  canonical spec in a form tools check only after a build step. It is the most
  *arguable* item: the generated spec is itself ordinary typed Python (the spec is
  *not* the untyped part of the tree), and the prose form is a deliberate readability
  choice many value — so it trades a genuine human benefit for an indirection cost
  rather than being pure debt.

This is exactly what the audit's cross-cutting sections — *the typing deficit*
and *the absence of a common test structure* in
[`reports/README.md`](README.md) — predict: the individual deep-dives
converge on the same handful of fixes. leanSpec shows those fixes composing in one
codebase — so far for a single fork (its multi-fork machinery is built but not yet
exercised at consensus-specs' fork count). It is younger and smaller, but the
properties that make it better engineered — spec-as-Python, strict typed domain
models, polymorphic forks, a pytest-plugin filler, declarative build and quality
gates, two clean distributions — are portable, and they are precisely the
direction every consensus-specs deep-dive already points.

---

## The repair-cost asymmetry

One thing the scorecard understates: leanSpec did not *solve* these problems — it
never *incurred* them. Every property in the
scorecard (`src/` layout, spec-as-Python, strict typed domain models, a
pytest-plugin filler, a validated fork registry, two distributions) holds **by
construction**, from the first commit. In consensus-specs each is a *retrofit*,
and the retrofits are **interlocking**: the debts are load-bearing for one
another, so they cannot be fixed independently or in an arbitrary order. The audit
says exactly this — most findings are "local — fixable in one PR," but a few are
"foundational" (`self-referential-package-layout.md:188`), and "several of these
refactors are intertwined and the right sequencing depends on which constraints
the team prioritises first" (`reports/README.md:96-99`).

That asymmetry — *free-by-construction* on one side, *coordinated multi-front
refactor* on the other — is the true distance between the two codebases, and the
scorecard only hints at it.

### The flagship chain: getting the typing leanSpec has for free

leanSpec type-checks its whole tree with `ty` under `error-on-warning` (§6)
because the spec is Python and its values are validated Pydantic types (§1). The
typing is the *starting condition*. To reach the same place, consensus-specs must
clear an interlocking stack — and the self-referential package layout sits right
in the middle of it:

1. **The checker reaches almost nothing.** `MYPY_SCOPE` analyses only the
   generated spec packages, so the ~877 errors across the helper layer, `pysetup/`,
   and `tests/infra/` are never checked (§6). Widening the scope is necessary — but
   not nearly sufficient.
2. **mypy cannot even *run* on the helpers.** Because the package source lives
   inside the test tree *and* is added to `sys.path` (§9), the same files are importable
   under two dotted names, and mypy aborts before checking anything: `Source file
   found twice under different module names`
   (`self-referential-package-layout.md:111-119`). `--explicit-package-bases` is a
   band-aid; the structural fix is moving the package out of `tests/` — "a major
   refactor — touching `setup.py`, `pyproject.toml`, `pysetup/`, every test file's
   resolution path, the editable-install machinery, `.gitignore`, and CI … the kind
   of cleanup that justifies a 'v2.0' release"
   (`self-referential-package-layout.md:229-234`). Until that lands, the
   helper-layer errors stay out of CI, checkable only via the
   `--explicit-package-bases` band-aid (`self-referential-package-layout.md:128`).
3. **Even scoped and runnable, the helper→spec boundary is `Any`.** The helpers
   reach the spec through the untyped `(spec, state)` god object — `spec` is typed as
   the empty `Spec` Protocol (§1) — so a call like `spec.get_beacon_committee(…)` is
   unchecked *no matter how the spec was authored*. (The generated spec `.py` is
   itself typed and is exactly what `MYPY_SCOPE` checks; the gap is the god-object
   parameter, not the spec's format.) Closing it is the typed-fork-class restructure
   of §1 — a large, behaviour-touching change across the 10k-line layer.
4. **You cannot safely type 10,000 lines without a net.** The helper layer has no
   characterisation tests, and neither does the `pysetup/` extractor that generates
   the entire spec, so behaviour-preserving refactoring has nothing to refactor
   *against* (`reports/README.md:568-590`). Writing that net is itself gated on the
   spec-as-code decision in step 3.

So one visible goal — "add types" — unfolds into four entangled, order-dependent,
very-high-cost refactors, two of which are pure behaviour-preserving prep that
improve nothing observable on their own. On the leanSpec side the same goal is the
empty set: it is already done. That is the asymmetry the audit keeps running into.

### The same shape, everywhere

Typing is not special. The deepest structural roots are **the self-referential
package layout** (§9, with the deep test-tree nesting that shares its move) and
**the untyped `(spec, state)` god object** (§1): together they keep the helper layer from
being typed and gate most other fixes, which route through a v2.0-scale change
before they can land:

- **Decorator stack → pytest plugin** is blocked by the dual-module `conftest`
  workaround (a layout artefact) and by the `spec.config.__hash__()`-keyed cache it
  would have to replace (ad-hoc caching) — neither of which can be cleaned up
  before the layout and caching debts are.
- **Fork registration** can't collapse its `.gitignore` shotgun until the package
  moves out of `tests/`, and its `PREVIOUS_FORK_OF` genealogy, string-template
  builders, and hand-maintained CI fork lists all have to be reconciled together.
- **Splitting the kitchen-sink wheel into two distributions** is blocked by the
  layout move *and* the directory restructure *and* build orchestration — and by
  Hyrum's Law, since every reachable name is already a de-facto public contract for
  the downstream client teams.
- **Build orchestration** "encodes the physical package layout in `Makefile` string
  substitutions," so it cannot simplify until the layout and fork-registration
  debts move first.

### Retrofit-difficulty matrix

Cost of fixing each debt **in place in consensus-specs**, and the prerequisites
that must be unblocked first. Every corresponding leanSpec cell would read
*"n/a — free by construction."*

| Debt (audit topic) | Retrofit cost | Must be unblocked first |
|---|---|---|
| Spec authoring → Python (markdown-as-source) | Very high | rewrite/retire `pysetup/`; characterise the extractor; years of EIP / client / readability buy-in |
| Self-referential package layout | Very high | "v2.0" move out of `tests/`: `setup.py`, `pyproject.toml`, `pysetup` retarget, every import, editable install, `.gitignore`, CI |
| Directory structure (deep nesting) | Very high | shares the v2.0 move above; preserve on-disk vector paths for downstream clients |
| **Real typing** (the ~877-error backlog) | Very high | self-ref layout (mypy can't traverse) · the untyped `(spec, state)` god object / empty `Spec` Protocol (§1) · characterisation tests (absent) · scope widening |
| Helper layer (10k lines, `is_post_*`) | Very high | directory / self-ref move · working typing · characterisation tests · ad-hoc-cache collapse |
| Decorator stack → pytest plugin | Very high | self-ref (dual-module `conftest`) · ad-hoc caching (`spec.config.__hash__`) · typed result protocol · single entry point |
| Fork registration (11 sites, 3 formats) | Very high | string-template builders · `PREVIOUS_FORK_OF` genealogy · `.gitignore` (needs layout move) · CI fork lists |
| Packaging → two distributions | Very high | self-ref move · directory restructure · build orchestration · Hyrum's-law downstream migration |
| Static-analysis config | High | self-ref layout (can't widen `MYPY_SCOPE`) · third-party stubs / `py.typed` · characterisation |
| Test oracle / vector formats | High–very high | pytest-plugin rework · typed fixture models · markdown-as-source · multi-client format migration |
| Build orchestration | Very high | fork registration · markdown-as-source · layout · config (the `Makefile` encodes the layout) |
| Unit tests for test-support code | Prerequisite | the safe first step — but the extractor can't be characterised until the spec-as-code decision is made |

The cluster of "Very high" with overlapping prerequisites *is* the finding: these
are not twelve independent tickets but one entangled graph rooted in two
long-standing foundational decisions.

### The honest version

None of this means consensus-specs is badly run. The difficulty is real *because*
the spec is mainnet-critical and multi-client, and markdown-first authoring was a
deliberate readability choice — the entanglement is the compound interest on years
of accreted, interdependent decisions, not carelessness.

Two facts follow. The in-place path is a coordinated, multi-quarter refactor across
interlocking foundational changes — each blocked on the others — carried out on a
live artifact that secures the network, where a behaviour-preserving slip is a
consensus bug. And the same end-state already exists in leanSpec's framework — the
`src/` layout, the typed domain model, the plugin test framework, the fork registry
— built and exercised independently of mainnet (so far for a single fork). The next
section sets the two paths side by side, with what each would cost.

---

## Two ways to close the gap — and what each costs

There are two ways to reach the end-state the dimensions above describe, and they
are not equal in effort or risk. (A third shape — repairing consensus-specs *in
place* — is the subject of the section above: an interlocking, multi-quarter change
on a live artifact.) Both options below build on leanSpec's framework, which already
exists and is exercised independently of mainnet; they differ only in where the work
lands:

- **Greenfield — consensus-specs on leanSpec.** Build the consensus spec on
  leanSpec's framework (typed domain types, the pytest-plugin filler, the fork
  registry, `src/` layout, declarative build + quality gates) and migrate the spec
  content and forks onto it, deprecating the current repo over time. Cleanest
  result, no debt carried forward; cost is running two repos through a transition
  and a cutover.
- **In-repo — move the framework into consensus-specs.** Bring leanSpec's framework
  into the existing consensus-specs repo and port the content onto it in place.
  Keeps one repo plus its history and URLs; cost is doing the work amid the existing
  structure — but still *adopting* a built framework rather than inventing one.

On effort, risk, and payoff the paths differ — and a framework-based path is not
cost-free; the comparison turns on *where* each path's cost falls:

- **Effort.** A framework-based path inherits the framework layers already built —
  the largest, most interlocking part of an in-place repair (see the retrofit
  matrix). Its main remaining cost is porting the *spec* — years of forks, the SSZ
  containers, the state-transition logic — onto that base. The *test suite* need not
  be re-authored: the existing consensus-specs vectors can stay in their current
  form as a frozen conformance corpus the new implementation must reproduce
  (leanSpec already runs fixtures against an implementation), with only *new* tests
  written in the leanSpec form. That narrows the content port to the spec plus a
  reader for the legacy vector formats — bounded, and continuously checkable against
  the existing vectors, where an in-place repair must instead design the framework
  *and* keep the live generator working throughout.
- **Risk.** This is where the paths diverge most, and in a framework-based path's
  favour. An in-place repair has no safe intermediate state — it restructures the
  live, mainnet-critical generator. A framework-based path can run the existing
  generator untouched until the new one produces *bit-identical* vectors, so mainnet
  is never exposed to an unvalidated generator; the cutover, not the rebuild, is the
  risk to manage. (Caveat: leanSpec's framework is itself validated only for a single
  fork so far; its multi-fork machinery is built but unexercised at consensus-specs'
  fork count.)
- **Payoff.** A framework-based path yields a typed, tool-legible foundation the
  dimensions above show consensus-specs lacks, shared across the consensus and lean
  efforts — and it inherits leanSpec's runnable, interop-tested reference node and
  its post-quantum cryptography ([Beyond the comparison](#beyond-the-comparison--leanspec-capabilities-with-no-consensus-specs-counterpart)),
  i.e. an executable reference implementation, not just spec tooling. The flip side is coupling: a shared framework needs an owner and a release
  discipline, and leanSpec's *no-backward-compatibility* rule would have to be
  reconciled with a mainnet artifact's need for stability.
- **Not free either way.** The typing and tooling wins depend on authoring the spec
  as Python — i.e. giving up Markdown's prose readability (§12) — so the EIP-process
  and client buy-in the retrofit matrix charges to in-place applies to a
  framework-based path too. It is not an independent toggle.

Each per-dimension *"already in the framework"* note above marks a capability that
exists, ready, in leanSpec. An in-place repair rebuilds a dozen of them separately
and re-incurs the integration cost the audit documents; a framework-based path
inherits them as one piece.

**Bottom line.** The framework-based paths trade an open-ended, mainnet-exposed
in-place restructure for a *bounded, vector-checkable* content port plus a cutover.
On **risk** the difference is real and in their favour — the existing generator runs
untouched until the replacement is bit-identical. On **effort**, keeping the
existing vectors as the conformance corpus removes the test-suite re-authoring,
leaving the spec port plus a legacy-vector reader; whether that residual is smaller
than the in-place framework-rebuild is not measured here, but the framework layers —
the hard, interlocking part — already exist and would not be rebuilt. The costs are
stated above so the reader can weigh them; none of this is free.

---

## Provenance

- **Subjects:** consensus-specs `90cee983d` (2026-05-05); leanSpec `d80e55d`
  (2026-05-30). consensus-specs was unchanged from the original audit; leanSpec was
  pulled to current `main` for this comparison — recent refactors (notably the
  dissolution of `lean_spec.types`, nesting `forks/` under `spec/`, and the
  `tox.ini → Justfile` migration) moved many citation targets, so every leanSpec
  reference here was re-derived against the current tree.
- **Tooling:** [repomix](https://repomix.com) source maps; a FalkorDB code graph
  (freshly re-indexed for leanSpec; because the spec is Python it is fully
  represented in the graph, whereas consensus-specs' `state_transition` has no node);
  pyright/`ty` for symbol navigation.
- **Method:** evidence was gathered one dimension at a time, and **every leanSpec
  citation was adversarially re-verified against the live source** before
  inclusion; headline metrics (e.g. `is_post_*` 220 vs. 0) were confirmed by
  direct `grep`. Consensus-specs evidence reuses the line-cited findings in the
  [README](README.md) and deep-dives, which were hand-authored end-to-end. The
  static-analysis claims here were calibrated against the review discussion on the
  audit PR (ethereum/consensus-specs#5233): a maintainer noted that
  `ignore_missing_imports` does not *literally* disable the strict directives, and
  that `codespell`/`mdformat` run via the `Makefile`. Both are reflected above; the
  sharpened, defended point is that the gates' *reach* — scope, plus the
  self-referential-layout blocker that stops mypy running at all, and the
  ~21,000-violation distance to the broader ruff families — not their existence, is
  what separates them from leanSpec's.
- **Companion:** the rest of the audit lives in this directory — the
  [README](README.md), the deep-dives, and the [glossary](glossary.md).
