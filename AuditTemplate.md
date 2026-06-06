# Code Audit & Review — Reusable Prompt

A general-purpose prompt for having Claude (in Claude Code, or chat with code execution)
do a rigorous, security-focused audit of any repo and produce a report a coding agent can
act on. Replace the placeholder, paste, run.

---

## The prompt (copy this)

```
You are performing a rigorous, security-focused audit and code review of the project at
<REPO URL OR LOCAL PATH>. Clone or open it and read it in full — every source file, the
build/CI config, launchers/scripts, and configuration. Read the code; do not skim. If the
repo is too large to read entirely, read all security-sensitive and core paths in full
(entry points, core logic, auth, parsing, native/FFI, I/O, persistence, build/CI) and
explicitly list what you read in full versus sampled.

AUDIT FOR:
- Correctness — logic errors, edge cases, off-by-ones, race conditions, error-handling
  gaps, incorrect assumptions.
- Security — injection (SQL/command/template/CSV), unsafe deserialization
  (pickle/yaml/etc.), path traversal, secrets committed in code or git history,
  authn/authz gaps, SSRF, unsafe or over-permissive defaults, privilege/elevation
  handling, native-memory/FFI/ctypes bounds, and dependency/supply-chain risk. Adapt this
  list to the actual stack and call out anything stack-specific.
- Performance & efficiency — hot-path waste, redundant allocations/IO, accidental O(n^2),
  blocking work on the wrong thread, scalability ceilings.
- Robustness — resource cleanup, crash/atomicity safety, thread-safety, validation of
  untrusted input, behavior under failure.
- Quality — dead code, duplication, doc/comment drift, missing types or tests.

VERIFY — DO NOT ASSERT:
- Run the test suite if present; report exact pass/fail counts.
- Build / typecheck / lint / byte-compile as the stack allows.
- Before claiming any non-trivial bug, reproduce it deterministically — a minimal repro or
  an isolated control-flow demonstration.
- Cite exact file:line for every finding.
- State plainly what you could NOT verify and why (needs a display / hardware /
  credentials / specific OS), and label those findings "static-only."
- For any performance claim, measure — profile or time the hot path on a representative
  workload and give numbers, not intuition. Do not flag (or optimize) cold paths.

CALIBRATION — THIS MATTERS MOST:
- Open with an honest one-line verdict: shippable as-is, or are there blockers? Do not
  inflate severity to seem thorough — if the code is clean, say so directly.
- Separate genuine findings from nitpicks. Give every finding a severity
  (Critical / High / Medium / Low / Info) with a one-line justification.
- Include a "Correct by design — do not change" section: things that look like bugs but
  are intentional, so the next agent doesn't break working code chasing phantoms.
- If you are unsure, say so. Never present a guess as fact.

FEATURES & IMPROVEMENTS:
- Don't stop at "is it correct" — also propose how to make the app better: rough edges in
  existing features, missing affordances/UX, weak defaults, observability gaps, and
  genuinely new capabilities it should plausibly have (gaps versus comparable tools, natural
  extensions of what's already built).
- Ground every idea in THIS app's purpose, architecture, and dependency posture — read the
  README/docs for stated scope and non-goals and respect them. No generic filler.
- Rank by value-to-effort, split quick wins from bigger bets, flag which are speculative,
  and keep features clearly separate from defects. Don't pad the list.

DELIVERABLE — write a Markdown report (e.g. AUDIT_REPORT.md at the repo root) that a coding
agent can act on directly, with these sections:
1. Verdict — 1-3 sentences plus a table scoring each dimension
   (security / correctness / performance / robustness / tests / quality).
2. Methodology / what I verified — commands run, test results, repro evidence, environment
   caveats.
3. Findings table — ID | Severity | file:line | one-liner, ordered by priority.
4. Detailed findings — per finding: what happens, impact, evidence/repro, the smallest
   correct fix (show the diff), and an acceptance test.
5. Test-coverage gaps — what's untested and how to test it without heavy infrastructure.
6. Correct by design — do not change.
7. Feature & improvement opportunities — grouped into Quick wins and Bigger bets; per item:
   what it is, the user need or gap it fills, where it would hook into the code, a rough
   effort estimate (S/M/L), and any risk/tradeoff. Ranked by value-to-effort and kept
   clearly separate from defects.
8. Suggested work order — numbered, highest-value first (fixes, plus any top features).
Also summarize the verdict and top findings in your reply.

CONSTRAINTS:
- This is a review pass: do NOT modify code or open PRs unless I explicitly ask — propose
  diffs in the report instead.
- Match the project's existing style and dependency posture in any suggested fix.
- Scale depth to repo size and never bail; if it's large, prioritize and state what you
  skipped.
```

---

## How to customize it (knobs)

- **Required:** replace `<REPO URL OR LOCAL PATH>` with the GitHub URL or the path Claude Code already has open.
- **Remediate instead of review:** append — *"Then implement fixes for everything Medium and above, run the test suite after each change, and open a PR. Keep each change focused and explained."*
- **Single-lens deep dive:** swap the AUDIT FOR list for one focus, e.g. *"Audit for security only, in depth — threat-model the trust boundaries first."* (or performance / correctness).
- **Report audience:** default targets a coding agent. For a human reviewer add — *"Write for a senior engineer who hasn't seen this code: more narrative, explain the why behind each finding."*
- **Benchmark against norms:** add — *"Compare against current best practices for <framework/language> and flag deviations."*
- **Big-repo budget:** add — *"Spend most of the effort on security-sensitive and concurrency code; treat UI/boilerplate lightly."*
- **Dependencies:** add — *"Check dependencies for known CVEs and unmaintained/abandoned packages; note pinning and lockfile hygiene."*

---

## Optimization pass (active, measured)

The base prompt finds and proposes performance fixes but won't change code. When you
actually want it to make the code faster, append this — it forces measure → optimize →
prove, so you get verified wins on real hot paths instead of cold-path guesswork:

```
Beyond reporting, optimize this project for speed and efficiency — but only where it's
warranted, and without changing behavior:

1. MEASURE FIRST. Profile or instrument to find where time, CPU, memory, and allocations
   actually go on a representative workload. Rank the hot paths. Optimize from data, not
   intuition — ignore cold paths no matter how ugly.
2. BASELINE. Capture a reproducible benchmark (inputs, iteration count, machine) and
   record the current numbers before touching anything.
3. OPTIMIZE, PRESERVING BEHAVIOR. Work the lenses that apply: algorithmic complexity and
   data-structure choice; redundant or duplicated work (cache/memoize); I/O and network
   batching and N+1 queries; concurrency/parallelism and not blocking the wrong thread;
   memory footprint, copies, and GC/allocation pressure; startup/import time; streaming or
   laziness for large data. The test suite must still pass and outputs must be unchanged
   (or within a tolerance you state explicitly).
4. PROVE THE WIN. Re-run the same benchmark and report before/after numbers with the exact
   method so I can reproduce it. Drop any change that doesn't measurably help.
5. TRADEOFFS. Don't sacrifice readability or maintainability for a cold-path micro-gain;
   call out every speed-vs-clarity and speed-vs-memory tradeoff.

Deliver the optimized diffs (or apply them and open a PR if I've said you can), plus a
table: change | hot path it targets | before | after | tradeoff.
```

---



## Suggest features & improvements (standalone)

When you want a forward-looking roadmap rather than a defect audit, use this on its own:

```
Study the project at <REPO URL OR LOCAL PATH> — read the code and the README/docs to
understand what it does, who it's for, and its stated scope and non-goals. Then propose how
to make it better:

- Improvements to what exists: rough edges, missing affordances/UX, weak defaults,
  reliability or observability gaps.
- New capabilities it should plausibly have: gaps versus comparable tools, and natural
  extensions of the existing architecture.

Ground every idea in this app's actual design and dependency posture — no generic filler,
and respect the project's non-goals. For each: what it is, the user need or gap it fills,
where it would hook into the code, a rough effort estimate (S/M/L), and any risk or
tradeoff. Group into Quick wins and Bigger bets, rank by value-to-effort, and flag which
are speculative. Don't pad — a short, honest set beats a long one.

DELIVERABLE — write a Markdown report.
```

---

## Compact variant (small repo or a single file)

```
Review <REPO/PATH or paste code> in full. Run the tests and build if present. Find real
bugs, security issues, and performance problems — cite file:line, reproduce anything
non-trivial, and assign each a severity. Be calibrated: lead with an honest verdict and
don't manufacture severity. List anything that looks wrong but is intentional. Give me a
prioritized punch-list with a concrete fix per item. Don't change code unless I ask.
```

---

### Why this version is better than a plain "review and audit it" ask

- **Calibration is enforced**, so you get an honest signal instead of a padded list — and the *"do not change"* section stops the next agent from breaking working code.
- **Evidence over assertion** — it runs the tests, reproduces bugs, and admits what it couldn't check, rather than guessing.
- **Structured, actionable output** — severity + file:line + diffs + acceptance tests + a work order hand off cleanly to Claude Code.
- **Stack-agnostic** — the security checklist adapts to any language/framework, not just one project.
- **Bounded** — it's a review pass by default (no surprise edits) and scales to repo size without bailing.
