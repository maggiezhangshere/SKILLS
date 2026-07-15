# Goal-Driven Review — Detailed Checklist

Reference file for `goal-driven-review`. Read this when you want depth on a particular
review dimension, or when a finding sits in a tricky area (security-sensitive code,
async hot paths, public-API changes). You don't need it for a routine review — the
SKILL.md workflow is enough. Come here on demand.

## Table of contents

- [Goal-source heuristics](#goal-source-heuristics)
- [Correctness](#correctness)
- [Reliability](#reliability)
- [Maintainability](#maintainability)
- [Regression risks](#regression-risks)
- [Edge cases worth checking](#edge-cases-worth-checking)
- [Error handling](#error-handling)
- [Missing tests](#missing-tests)
- [Project-specific rules (agent-core style)](#project-specific-rules-agent-core-style)

---

## Goal-source heuristics

How to read a goal out of each source, and how honest to be about it.

| Source (priority) | How to extract | Honesty note |
|---|---|---|
| 1. Explicit user goal | Use verbatim. Don't paraphrase away specifics. | Strongest signal. |
| 2. Design / requirement doc | Read fully, summarize to 1–2 sentences. Pull out any concrete success criteria (numbers, "no new API", compat constraints) — those become checkable in Goal Evaluation. | Say "summarized from `<doc>`". |
| 3. PR title / commit message | The conventional-commit type (`feat`/`fix`/`refactor`/`perf`) and subject are the goal. Body may add constraints. | Say "inferred from PR description / commit message". |
| 4. The diff itself | Ask: what problem does this change *solve*? What invariant does it *establish*? The answer is the goal. | Say "inferred from the diff — a reconstruction, not a stated goal" and invite correction. Calibrate findings a touch more conservatively. |

When multiple signals conflict (the PR title says one thing, the diff does another),
surface that: "PR title claims X, but the diff also does Y — which was intended?" Don't
silently pick one.

---

## Correctness

Ask, against the goal:

- Does it actually do what the goal requires — not just the happy path?
- Are return values / types consistent across branches? (A goal of "return enriched
  results" is defeated if one branch returns `None`.)
- For numeric work: floating-point equality, division-by-zero, `Decimal` from float.
- For collections: empty input, single-element input, duplicate keys.
- For anything with ordering or iteration: off-by-one, stability, mutation during
  iteration.
- Does it match the contract callers depend on? If signatures changed, are all
  callers updated in the same change?

A correctness finding that contradicts the goal is almost always 🔴 Blocking.

---

## Reliability

- Failure under bad input: does it raise a meaningful exception, or silently produce
  wrong output?
- Partial failure: if step 3 of 5 fails, is state left consistent or corrupt?
- Concurrency / async: reentrancy, shared mutable state, blocking calls inside
  async paths, un-awaited tasks, deadlocks.
- Resource cleanup: are files/sockets/locks/sessions released on *both* the success
  and exception paths? (Pair acquire with release — this is a common real leak.)
- Timeouts and cancellation: does it hang forever, or propagate cancellation?
- Per-call cost of *new* code on a repeated path: not blast radius (see Regression), but
  whether the new code itself repeats O(n) scans, full-set rebuilds, or per-invocation
  `Path.resolve()`/`.is_file()` calls. Calibrate 🟡 unless the goal names a
  latency/throughput target (then 🟠/🔴). Skip when the change isn't on a repeated path.
- External calls (network, subprocess): are timeouts set? Is the failure mode
  graceful?

Reliability issues are usually 🟠 Important or 🔴 Blocking — silent wrongness or
resource leaks are merge-blockers.

---

## Maintainability

- Can the next reader understand *why* this exists, not just *what* it does? A
  one-line "why" comment on a non-obvious decision is worth more than describing the
  code.
- Naming: do names match what the code does? A `get_x` that mutates is a landmine.
- Coupling: does the change add a dependency the goal explicitly wanted to *reduce*?
  (See Example 3 — Controller/Harness decoupling.)
- Surface area: is new public API really needed? Could it be private/module-local?
  Public surface is permanent; keeping it small is a maintainability win.
- Orphaned new symbols: for each new public method/function/class the diff adds, grep
  the repo (`openjiuwen/` *and* `tests/`) for at least one caller. A new symbol with
  zero callers is unrequested surface — dead code the goal didn't ask for. ruff's
  F-rules catch unused *imports*, not unused *definitions*, so this needs a manual
  `grep -rn "<name>" <src> <tests> --include="*.py"` (or `vulture`) — don't expect the
  lint gate to surface it. Report as a 🟡 Suggestion (delete it, or anchor it with a
  test/comment if it's groundwork for a stated next step). Skip this when the diff adds
  no new public symbols.
- Complexity: if 50 lines could be 20 without losing the goal, that's a 🟡 Suggestion.
  (Mirrors karpathy-principles §2.)
- **Structural organization**: are responsibilities in appropriate modules, has a file or
  class accumulated multiple unrelated responsibilities, and is there logic that should be
  extracted into a separate function / class / module? Raise this only when the current
  organization *actually causes* one of: unclear responsibility boundaries, duplicated
  logic, difficult testing, or contradiction with the stated design goal. Do **not**
  recommend extraction merely to shrink a file or add abstractions — that's speculative
  refactoring (see SKILL.md "No speculative refactors"). Coupling that the goal wanted
  *reduced* is the highest-signal structural finding (Example 3 — Controller/Harness
  decoupling): a change that *adds* such coupling moves against the goal.

Pure style (indent, line width, import order) is out of scope unless it violates an
explicit project rule — see [Project-specific rules](#project-specific-rules-agent-core-style).

---

## Regression risks

- Does it change behavior the goal assumed would keep working?
- For public API: signature changes, default-value changes, removed params — these
  break callers. Prefer additive, keyword-only changes.
- For shared/global state (e.g. `Runner.resource_mgr`-style singletons): does the
  change collide with other registrations or leak across tests?
- Does it touch a hot path used by many callers? Wider blast radius → higher severity.
- Are there callers/tests that would now break, and are they updated in this change?
- Serialization / data-compat: if a spec/config claims JSON/persist round-trip (check the
  docstring), do field adds/renames/removals break already-stored data? Does
  `ConfigDict(extra="forbid")` reject legacy formats that carry now-unknown fields? Stored-data
  breakage = 🔴 (silent, hits production); a narrow forbid the loader compensates for = 🟠 to verify.
- Deprecation lifecycle: a new `DeprecationWarning` should name its replacement and
  ideally a removal window. A deprecated public API still referenced in `docs/`/`examples/`
  extends its life and misleads users → 🟠. Bare deprecation with no replacement path → 🟡.

A regression against the stated goal is 🔴 Blocking; an unrelated regression is still
🟠 Important at minimum.

---

## Edge cases worth checking

The ones that matter depend on the goal, but a short list that's worth a glance:

- Empty / None / null / missing key
- Single element, very large input
- Duplicate / conflicting input
- Off-by-one, inclusive vs exclusive bounds
- Unicode / encoding (paths, filenames, content)
- Timezone-aware vs naive datetimes
- Concurrent / re-entrant calls
- Cancellation mid-operation
- Retry / partial failure on an external call

Only report the ones the goal makes relevant — don't pad the review with hypotheticals.

---

## Error handling

- Are failures surfaced (raised/logged), or swallowed silently?
- Is the exception type meaningful (a domain error), not a bare `Exception`?
- Is the original stack trace preserved when re-raising/translating?
- Are sensitive values (tokens, paths, user data) excluded from error messages and logs?
- Is there a `finally` / context manager that cleans up on every path?

Swallowed errors and leaked-sensitive-data-in-logs are the two findings most worth
escalating here.

---

## Missing tests

List the behaviors the *goal* depends on that aren't covered. Tie each to the goal:

- The core behavior the goal promises.
- The edge cases the goal makes likely (see above).
- The regression the goal is supposed to *prevent* — write a test that would fail if
  the bug/leak came back.
- For security-sensitive paths: a test that the bad input is rejected.

Prefer targeted tests mirroring the source path
(`openjiuwen/harness/deep_agent.py` → `tests/unit_tests/harness/test_deep_agent.py`).
If none are missing, say "No important missing tests." Don't invent test ideas the goal
doesn't need.

---

## Project-specific rules (agent-core style)

When the reviewed repo carries its own conventions, those define "correct" and
"maintainable" here — not generic defaults. For an agent-core-style repo, the
relevant guardrails to weigh findings against are:

- **Architecture / public API** (`.claude/rules/architecture.md`): public API changes
  must be additive (preserve names, positional args; prefer keyword-only). Card/Config
  split — cards are static/serializable metadata, configs are runtime state. Putting
  `session_id`/`runner` in a Card is an anti-pattern.
- **Code style** (`.claude/rules/code-style.md`, `.claude/rules/python/coding-style.md`):
  Python 3.11+, Ruff line length 120, 4-space indent, `logging` over `print()` in
  library code, type hints on new public APIs. Library code must be async-safe — no
  blocking calls in async paths.
- **Security** (`.claude/rules/security.md`, `.claude/rules/python/security.md`): no
  hardcoded secrets (use `os.getenv`), `safe_path` validation on user paths,
  parameterized SQL, no `shell=True`, no `eval`/`exec` on untrusted input. `sys_operation`
  and sandbox code are security-sensitive — be strict.
- **Testing** (`.claude/rules/testing.md`): mirror source path in tests, mock
  credentials with `os.getenv("KEY", "mock-...")`, keep unit tests network-free.
- **Error codes** (`.claude/rules/error-codes.md`): StatusCode / BaseError conventions.
- **Logging** (`.claude/rules/logging.md`): lazy interpolation, no sensitive data in logs.
- **Git workflow** (`.claude/rules/git-workflow.md`): commit/PR conventions.
- **Language standard**: `.claude/skills/python-rules/` (or the global `python-rules`
  skill) — the full Python coding standard. If the change is `.py`, weigh it against
  those rules; a violation of a rule that protects correctness/security (e.g. bare
  `except`, mutable default arg, `assert` in production) is a real finding, not a
  style nit.

Run the project's own `make check` / lint where available before reporting style — if
the linter catches it, the finding is low-value noise. Report only what the linter
*can't* catch: goal failures, logic bugs, missing edge-case handling, design-level
coupling, missing tests.

### Running the gates (do, don't eyeball)

This is the operational checklist behind SKILL.md's "Project conventions" + Step 2.5.
The point isn't to re-derive style by hand — it's to run the project's gates and let
their output steer the review.

1. **Discover**: `ls .claude/rules/`, read `AGENTS.md`/`CLAUDE.md`, grep `pyproject.toml`
   and `Makefile` for the canonical commands. Read the rules whose `paths:` frontmatter
   covers the touched files.
2. **Lint**: `make check` (or `ruff check` with the project config). For Python, also run
   `ruff check --select E,F,W` — the project's `make check` often narrows to `--select I`
   (import order only), so the wider selection surfaces undefined-name (F821) /
   unused-import classes that the gate skips. A project-gate error → Blocking-or-near;
   a wider-selection-only warning → usually Suggestion.
3. **Type-check**: `make type-check` (often `mypy`). If `mypy`/`pyright` isn't installed
   in the environment, record "type-check: not run (reason)" — do not let silence imply
   it passed.
4. **Tests**: run the **diff-derived test set first**, then mirror tests.
   - Diff-derived (highest signal — the author changed these for a reason): `git diff --name-only HEAD`
     (or `--staged`/the branch's merge-base diff) and run **every** test file the diff touches,
     whatever directory it's in. A change that adds a public API often has its contract test outside
     the source-mirror dir (e.g. an integration test under `tests/unit_tests/auto_harness/` for code
     under `openjiuwen/harness/`). Mirror-only inference misses these — `git diff --name-only` does not.
   - Mirror (fallback for paths the diff didn't already cover): `make test TESTFLAGS="..."` or
     `pytest <mirror>` per testing.md's source-path mirroring
     (`openjiuwen/harness/deep_agent.py` → `tests/unit_tests/harness/test_deep_agent.py`).
   Broken tests → Blocking; new behavior with no diff-touched *and* no mirror test →
   Missing-Tests finding. Record which set was run; never claim "tests ✓" from mirror-only
   when changed test files exist outside the mirror dir.
5. **Doc/API sync**: if public API added/changed/removed, check `docs/` + `examples/`
   (testing.md: "update docs/examples alongside tests when behavior is user-visible").
   New public API with no doc → Suggestion; a doc still naming a removed/renamed API →
   Important.

### python-rules high-value spot-check

When the change is `.py`, a handful of `python-rules` entries protect correctness or
security rather than style — check these directly, because a linter configured to
`--select I` won't catch them. None of these is a "style nit"; each is a real finding if
violated:

- **Mutable default arg** (`def f(x=[])`/`={}`) → must use `field(default_factory=...)`
  / `None`.
- **Bare `except:`** or over-broad `except Exception` that swallows failures silently.
- **`assert` in production** (stripped under `-O`) — use real validation.
- **`eval` / `exec` on untrusted input**, **`shell=True`**, **`pickle.load` of external
  data**, **`yaml.load`** (must be `safe_load`) — security.
- **`print()` in library code** — must use project `logging`.
- **Blocking call in an async path** (architecture.md: library code must be async-safe).
- **Public-API signature change** — must be additive, keyword-only for new optional
  params (architecture.md).

A grep for these over the touched files takes seconds and catches what a narrowed lint
won't. Last-resort: if you can't run a gate, name it and the reason it was skipped, and
carry it as an explicit unverified item — never let a not-run gate read as "passed".
