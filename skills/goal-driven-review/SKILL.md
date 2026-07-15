---
name: goal-driven-review
description: >
  Implementation-aware code review that first recovers the intended design goal of a
  change, then judges whether the implementation actually achieves it — before drilling
  into correctness, reliability, and maintainability. Use this skill whenever the user
  asks to review code, review a git diff, review a pull request, review an implementation,
  or review current changes. Trigger on phrasings like "review this PR", "review the
  current git diff", "review this implementation", or "review these changes" — even when
  the user never says "goal-driven". Works well for Python projects but is language-agnostic;
  the focus is always on whether the change meets its design goal. Supports design goals,
  design docs, and requirement docs provided in either Chinese or English.
---

# Goal-Driven Review

A review that judges a change against what it set out to do — not against an abstract
ideal. Style nits and "I would have done it differently" take a back seat to the one
question that actually matters: **does this change achieve its intended design goal?**

This matters because a review without a goal drifts into personal preference. The
author wrote this code on purpose, toward some end. When you don't know that end you
can't tell a real bug from a deliberate tradeoff — so the first job is to recover the
goal (from the user, a design doc, a PR description, or the diff itself), then measure
the implementation against it.

This is the goal-driven execution mindset from `.claude/rules/karpathy-principles.md`
§4 ("define success criteria, loop until verified") applied to *reading* code rather
than writing it: the author's stated goal is the success criterion you verify against.

## When to use

Trigger whenever the user asks to review:

- code
- a git diff
- a pull request
- an implementation
- current changes

Phrasings like these — or anything close — should trigger:

- "Review this PR."
- "Review current git diff."
- "Review this implementation."
- "Review these changes."

The user may optionally supply a design goal or a design/requirement document. Both
are supported. A review with no explicit goal still works — you infer the goal first,
then review.

## Language

The user may write goals, design docs, requirements, or the review request itself in
either Chinese or English. Understand everything in its original language.

The final review **must** use the same language as the user's request. If they write in
Chinese the whole review is in Chinese; if in English, in English. This is non-negotiable:
handing back a Chinese review for an English request (or vice versa) breaks the basic
contract that the author can read what you wrote.

## Project conventions — the repo's rules are the real spec

A repo that ships its own conventions has already decided what "correct" and
"maintainable" mean here. Your generic defaults are a worse source of truth than what
the team wrote down. So before judging: **discover the conventions, run the gates,
weigh findings against the rules.** Doing this only half-way is the most common failure
mode of this skill — reading the architecture doc but skipping the testing rule, running
the linter but not the tests, and so quietly reporting on a standard you never actually
held the code to. The steps below exist to stop that from happening.

### Discover

Look for, in order of authority:

- A root `AGENTS.md` / `CLAUDE.md` — shared architecture and instruction-priority guidance.
- A `.claude/rules/` directory — topic-scoped rules (architecture, code-style, testing,
  security, logging, git-workflow, error-codes, plus a `python/` subtree where present).
- A `.claude/skills/python-rules/` (or the global `python-rules` skill) — the
  language-level coding standard, e.g. the list of forbidden patterns (`eval`/`exec`
  on untrusted input, `shell=True`, `assert` in prod, mutable default args, bare
  `except`, `pickle.load` of external data, `print()` in library code).
- `pyproject.toml` / `Makefile` — the canonical tooling: line length, lint, type-check,
  test, and fix commands.

Read the rules **relevant to the touched module**, not all of them. A change to
`harness/rails/` pulls in architecture + code-style + testing; a change to
`core/sys_operation/` additionally pulls in security. The `paths:` frontmatter on each
`.claude/rules/*.md` file tells you which source paths it governs — use it to scope.

### Run the gates (do not re-derive by hand)

Prefer the project's own gates over eyeballing. A finding a linter already catches is
low-value noise; the point is to find what the gates *can't* catch (goal failures,
logic bugs, missing edge cases, coupling, missing tests).

- **Lint**: `make check` (or the repo's equivalent). For Python, also run
  `ruff check` with the project's config — and once with `--select E,F,W` to surface
  undefined-name / unused-import classes of bugs that the project's narrower `make`
  selection (often `--select I`) skips. Distinguish: an error the project gate raises
  is Blocking-or-near; a warning only the wider `--select` raises is usually a
  Suggestion.
- **Type-check**: `make type-check` (often `mypy`). If the tool isn't installed in the
  environment, **do not pretend it passed** — record "type-check: not run (mypy
  unavailable)" and carry that as an explicit unverified item in the review (see
  "Honesty about unverified gates" below).
- **Tests**: run the **diff-derived test set first**, then mirror tests. The diff
  is the truth source — `git diff --name-only` and run every test file the change touches,
  whatever directory it's in. Mirror tests (`openjiuwen/harness/deep_agent.py` →
  `tests/unit_tests/harness/test_deep_agent.py`) are the fallback for paths the diff
  didn't already cover. Why both: a change that adds a public API often has its contract
  test outside the source-mirror dir (e.g. an integration test under
  `tests/unit_tests/auto_harness/` for code under `openjiuwen/harness/`) — mirror-only
  inference misses it. Broken tests → Blocking; new behavior with no diff-touched *and* no
  mirror test → Missing-Tests finding. Record which set was run; never claim "tests ✓"
  mirror-only when changed test files exist outside the mirror dir.

### Weigh findings against the rules

A change that follows the repo's conventions is almost never a "style" finding — leave
it alone. A change that **violates an explicit project rule** may be a real
maintainability or correctness issue: a blocking call in an async path (architecture.md),
a hardcoded secret or unvalidated path (security.md), a public-API signature change
(architecture.md "keep public API additive"), a missing mirror test (testing.md). This
is the exception to "skip style": project rules that protect correctness, security, or
the public-API surface are in scope, and the rule they violate should be cited in the
finding (e.g. "violates `.claude/rules/testing.md`: new public API has no mirror test").

### Honesty about unverified gates

Not every gate can be run every time — the tool may be missing, the sandbox may block
network tests, the diff may be too large to type-check in one shot. When a gate is not
run, **say so explicitly in the review**: name the gate, the reason it was skipped, and
what risk that leaves uncovered. "type-check: not run (mypy not installed in this
environment) — type errors in the hot-load path would not be caught here" is far more
useful than silence, because silence reads as "all green" to the author. Treat an
unverified gate as a first-class fact in the review, not a footnote to hide.

## Workflow

### Step 1 — Determine the design goal

Before implementation details, recover the intended design goal. Use this strict
priority — stop at the first that applies:

1. **Explicit goal.** The user states a Goal directly → use it verbatim.
2. **Design / requirement document.** The user points to a design doc or requirements
   → read it and summarize the design goal in one or two sentences.
3. **PR description or commit message.** No goal given, but a PR/commit message exists
   → infer the goal from it.
4. **The diff itself.** None of the above → infer the goal from the current git diff.

Why this order: the author's own words are the most reliable signal of intent; a doc is
next; a PR title is terse but intentional; the diff is the fallback, and intent read
from code is an *inference* — be honest that it is one.

**Always present the inferred design goal before reviewing.** This lets the author
correct you before you spend the rest of the review arguing against a goal they never
held. If you inferred from a diff (priority 4), say so explicitly so the author knows
the goal is a reconstruction, not a stated fact.

### Step 2 — Gather the change

Get the actual content under review. Depending on what the user asked:

- files the user named
- `git diff` (unstaged) / `git diff --staged`
- a commit: `git show <sha>`
- a branch/PR: `git diff <base>...HEAD` — prefer the merge-base diff
  (`git diff $(git merge-base HEAD <base>)...HEAD`) so you see only this branch's
  changes, not everything since the base diverged.

If the scope is ambiguous, ask which changes to review rather than guessing —
reviewing the wrong diff wastes everyone's time.

### Step 2.5 — Run the project's gates (the real standard, not your defaults)

This step is what stops the review from being "I read the architecture doc and eyeballed
the rest." Do it before reading for detail, because the gate results steer where you look:
a failing test or a mypy error names the exact lines to scrutinize.

1. **Discover** — confirm `AGENTS.md`/`CLAUDE.md`, list `.claude/rules/`, check for
   `python-rules`, read `pyproject.toml`/`Makefile` for the canonical commands. Read the
   rules whose `paths:` frontmatter covers the touched files (see "Discover" above).
2. **Lint** — run the project's lint (`make check` / `ruff check` with the project
   config). For Python, also run `ruff check --select E,F,W` to catch undefined-name /
   unused-import bugs the project's narrower selection may skip.
3. **Type-check** — run `make type-check` (often `mypy`). If the tool is unavailable,
   record "type-check: not run (reason)" — do not let silence imply it passed.
4. **Tests** — run the diff-derived test set first (`git diff --name-only` → run
   every touched test file, whatever dir it's in), then mirror tests for paths the diff
   didn't already cover (`openjiuwen/harness/deep_agent.py` →
   `tests/unit_tests/harness/test_deep_agent.py`). Mirror-only inference misses contract
   tests that live outside the source-mirror dir — `git diff --name-only` does not. Broken
   tests are Blocking; new behavior with no diff-touched *and* no mirror test is a
   Missing-Tests finding. Record which set was run.
5. **Doc/API sync** — if the change adds or changes public API, check whether `docs/`
   and `examples/` need updating (testing.md: "update docs/examples alongside tests
   when behavior is user-visible"). A missing doc update for new public API is a
   Suggestion; an out-of-date doc that still names a removed/renamed API is Important.

Record the gate results compactly (pass / fail with output / not-run with reason).
These results feed the Findings and Missing-Tests sections downstream: a failing gate is
a finding, a not-run gate is an explicit unverified item.

### Step 3 — Review goal-first, then details

Read the change with the goal in mind. The central question is always:

- **Does the implementation achieve the intended design goal?**

Only once you've formed a view on that, evaluate the supporting dimensions:

- Correctness — does it do what the goal needs, including awkward cases?
- Reliability — does it fail safely under bad input, partial failure, concurrency?
- Maintainability — can the next reader understand and change it without fear? This
  includes **structural organization**: are responsibilities placed in appropriate
  modules, has a file or class accumulated multiple unrelated responsibilities, and is
  there logic that should be extracted into a separate function / class / module? Only
  raise structure as a finding when the current organization *actually causes* unclear
  boundaries, duplicated logic, difficult testing, or contradicts the stated design goal
  (see "No speculative refactors" below) — don't extract merely to shrink files or add
  abstractions. **Orphaned new symbols**: if the diff adds public methods/functions,
  grep each across `openjiuwen/` + `tests/` for ≥1 caller — zero callers is dead code the
  goal didn't ask for (ruff catches unused imports, not unused definitions). See
  `references/review-checklist.md` → Maintainability. Skip when no new public symbols.
- Regression risks — does it break behavior the goal assumed would keep working?
- Important edge cases — the ones the goal makes likely to matter (empty input,
  large input, None/null, off-by-one, reentrancy, ordering, cancellation).
- Error handling — are failures surfaced, not swallowed?
- Missing tests — are the behaviors the goal depends on actually covered?

Prioritize findings that directly affect the stated design goal. A goal that promises
"async without blocking" makes a synchronous call on the hot path a Blocking issue; a
goal of "keep TaskLoop unchanged" makes any edit to TaskLoop a Blocking issue. Tie each
finding back to the goal when you can.

For the per-perspective prompts and the full goal-source heuristics, see
[`references/review-checklist.md`](references/review-checklist.md) — read it when you
want depth on a particular dimension, or when a finding sits in a tricky area
(security-sensitive sysop code, async hot paths, public-API changes).

### Step 4 — Write the review

Use the output format below, in the user's language.

## Review principles

These keep the review useful rather than noisy. The aim is high signal: real problems
that affect the goal, correctness, reliability, or maintainability — and nothing else.

- **Goal-first.** Judge against the stated goal before anything else. If the change
  meets the goal, that is the headline, even if the implementation isn't to your taste.
- **Assume intention.** The author made choices on purpose. Treat the implementation as
  an intentional design unless strong evidence says otherwise — a comment, a test, or
  behavior that contradicts the goal.
- **No speculative refactors.** "You could also do X" is not a finding. Flag an
  alternative only when the chosen one materially hurts the goal, correctness,
  reliability, or maintainability. This covers structural extraction too: do **not**
  recommend pulling logic into a separate function / class / module merely to make a
  file smaller or to create more abstractions — only when the current organization
  actually causes unclear boundaries, duplicated logic, difficult testing, or contradicts
  the goal. (Mirrors karpathy-principles §2, Simplicity First.)
- **Surgical expectations.** Prefer small, targeted diffs. Don't ding a change for
  *not* refactoring adjacent code — that's the author's call, and opportunistic
  refactors are themselves a regression risk. (Mirrors §3, Surgical Changes.)
- **Skip style and formatting** unless it genuinely impairs correctness or
  maintainability — or violates an explicit project rule (see "Project conventions"
  above). Linters exist for a reason; don't replay them by hand.
- **Material impact only.** If it doesn't change behavior, reliability, or the goal,
  leave it out. Reviewers who report everything report nothing — the real issues drown.
- **Tie to the goal.** When a finding relates to the design goal, say so. That tells
  the author why it matters and helps them prioritize.
- **Be concrete.** Every finding gets a location and a suggested fix, not just a
  description of the problem. A finding you can't act on is dead weight.
- **Give credit.** When the author made a genuinely good design decision, name it.
  Reviews that only list problems teach the author what to avoid; naming good
  decisions teaches them what to repeat.

## Output format

Produce the review in exactly this structure, written in the user's language.

```
### Overall Decision

Choose exactly one:

- ✅ Approve
- 🟡 Comment
- ❌ Request Changes

Provide an overall risk level:

- Low
- Medium
- High

Provide a short summary.

Severity → decision:
- Any 🔴 Blocking, or any required in-scope gate failure (unless explicitly proven
  non-blocking for merge) → ❌ Request Changes.
- Any 🟠 Important → 🟡 Comment at minimum, usually Medium risk.
- Only 🟡 Suggestions and passing gates → ✅ Approve or 🟡 Comment / Low.

---

### Goal Evaluation

State the intended design goal.

Then evaluate whether the implementation:

- Fully satisfies the goal
- Partially satisfies the goal
- Does not satisfy the goal

Explain why.

---

### Gates & Conventions

Report what the project's gates actually said — one line per gate. This is where
"compliance" stops being a vibe and becomes evidence. If a gate was not run, say so
with the reason; silence here reads as "all green" to the author.

For each: name → result (pass / fail with the offending line / not-run with reason).

Typical entries:
- lint: `make check` / `ruff` → pass | fail (`file:line: rule`)
- type-check: `mypy` → pass | fail (`file:line: error`) | not-run (reason)
- tests: `pytest <mirror path>` → pass | fail (`test::name`)
- conventions: list the `.claude/rules/*.md` read for the touched module; note any
  explicit rule violated (cited by file in the finding below).

---

### Findings

Group findings by severity.

#### 🔴 Blocking
- Required project gate fails on in-scope new/touched code.
- Tests fail.
- The implementation fails the stated design goal.
- Security, data-loss, public-API breakage, rollback failure, or runtime crash likely in normal use.

#### 🟠 Important
- New/touched code has type-check or lint errors, but the repo gate is known dirty or not CI-blocking.
- A reliability/type-safety gap could become a runtime bug under plausible boundary conditions.
- Public API has bad failure behavior, unclear contract, or missing guard, but the main goal still works.
- Goal-relevant tests are missing for important edge cases.

#### 🟡 Suggestion
- Cleanup, readability, small guardrails, or advisory sweep findings that do not fail required gates.
- Linter false positives, reported only when they do not fail the required project gate.
Do not include formatting or style suggestions unless they affect maintainability.

#### 🌟 Good Design
Highlight particularly good implementation or design decisions when appropriate.

---

For every finding include:

- Location
- Related design goal (or the project rule violated, cited by file)
- Explanation
- Potential impact
- Suggested fix

---

### Missing Tests

List important missing test cases.

If none, explicitly state:

"No important missing tests."

---

### Final Recommendation

Provide a concise recommendation for the author.

Examples:

- Ready to merge.
- Merge after addressing the blocking issues.
- Consider the suggestions for future improvements.

If no meaningful issues are found, explicitly state that the implementation
successfully satisfies the intended design goal.
```

Leave a section out only when it's genuinely empty — a review with no "Missing Tests"
section at all is not OK; say "No important missing tests." instead. Explicit absences
are clearer than silent ones. The one exception is "Gates & Conventions": even when
every gate passes, keep a one-line summary ("lint ✓, type-check ✓, tests ✓, rules read:
architecture.md + testing.md") rather than dropping the section — a present, all-pass
gates section is the evidence that the standard was actually held.

## Examples

Each example shows the goal source and a sketch of the review that follows. The paths
are drawn from a real agent-core-style repo (`openjiuwen/core/controller/`,
`openjiuwen/harness/task_loop/`, `openjiuwen/core/session/`) so the shape is concrete —
but the review applies anywhere; substitute the touched module's real path.

### Example 1 — Explicit goal, English request

```
Review current git diff.

Goal: Support asynchronous execution without blocking parent tasks.
```

Goal source: priority 1 — explicit goal.

Review opens by restating the goal, evaluates whether the diff achieves it (look for
blocking calls on the parent's hot path, fire-and-forget tasks that should be awaited,
work that should run in the background but runs inline), then reports findings grouped
by severity — with the async-blocking ones as 🔴 Blocking because they directly defeat
the stated goal. Whole review in English.

### Example 2 — Explicit goal, Chinese request

```
Review 这个 PR。

Goal: 保持 TaskLoop 的实现不变，仅新增 Plan Mode。
```

Goal source: priority 1 — explicit goal.

Review in Chinese throughout. Because the goal is to keep `openjiuwen/harness/task_loop/`
untouched, **any** edit there is a 🔴 Blocking finding, no matter how small or "clean" —
the goal defines correctness here. Suggestions that touch TaskLoop aren't suggestions,
they're regressions against the goal. The review makes that linkage explicit in each
finding's "Related design goal" line.

### Example 3 — Goal inferred from a PR description

```
Review this PR.（无显式 Goal，但 PR 描述为"减少 Controller 与 Harness 的耦合"）
```

Goal source: priority 3 — inferred from PR description.

Review states the inferred goal — "减少 Controller 与 Harness 的耦合" — and notes it
was taken from the PR description, so the author knows the provenance. Findings then
prioritize anything that *adds* coupling between `openjiuwen/core/controller/` and
`openjiuwen/harness/` (a new direct import where an interface would do) as 🟠 Important
or 🔴 Blocking by severity, since those move against the goal. Review in Chinese.

### Example 4 — Goal inferred from the diff alone

```
Review these changes.（无显式 Goal，无 PR 描述）
```

Goal source: priority 4 — inferred from the diff.

Review is explicit that the goal is reconstructed and invites correction: "Inferred
goal: … — if this isn't what you intended, tell me and I'll re-review against the right
goal." Findings are calibrated a touch more conservatively, because an inferred goal is
a hypothesis. Review in whatever language the request was in.

### Example 5 — Goal summarized from a design doc

```
Review this implementation. Design doc: docs/design.md
```

Goal source: priority 2 — summarized from the provided design doc.

Read the doc, distill the design goal to one or two sentences, present it, and note it
was summarized from `docs/design.md`. If the doc specifies concrete success criteria
(e.g. "latency under 50ms", "no new public API"), treat those as part of the goal and
check them explicitly in Goal Evaluation.

### Example 6 — Goal: fix a lifecycle leak

```
Review this PR.

Goal: 修复 Session 生命周期泄漏。
```

Goal source: priority 1 — explicit goal.

Review verifies whether the leak is actually fixed (e.g. `openjiuwen/core/session/`
objects are now released, close hooks / `__del__` / async teardown run, references
aren't retained in long-lived caches) and adds regression tests for the leak to Missing
Tests. Findings that would re-introduce a leak — e.g. storing a Session on a
module-level dict without a removal path — are 🔴 Blocking because they directly undo
the goal. Review in Chinese.
