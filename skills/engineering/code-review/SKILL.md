---
name: code-review
description: |
  Three-axis review of the changes since a fixed point (commit, branch, tag, or merge-base):
  Standards (does the code follow this repo's documented conventions plus a code-smell baseline?),
  Spec (does the diff implement what the originating issue/PRD asked?), and Bugs (defect hunt with
  git-history context, where every candidate is re-judged by an independent verifier on a 0-100
  confidence rubric — only findings scoring ≥80 survive). Use whenever the user wants to review a
  branch, a PR, or work-in-progress changes, says "review since X" or "review this diff", or wants
  a second pair of eyes before committing, merging, or opening a PR.
---

# code-review

Review the diff between `HEAD` and a fixed point along **three separate axes**, then close with a
single actionable recommendation:

- **Standards** — does the code conform to this repo's documented conventions? (plus a fixed
  code-smell baseline that applies even when the repo documents nothing)
- **Spec** — does the diff faithfully implement the originating issue / PRD / spec?
- **Bugs** — are there real defects in the change, judged against the code *and* its git history?

The axes run as **parallel sub-agents** so they don't pollute each other's context. Bugs is the
only axis where hallucinated findings can poison the report — Standards and Spec findings must
quote their source (a standard file, a spec line), which makes them cheap to trust. So Bugs gets
a second pass: every candidate is re-judged by an independent verifier against a strict 0-100
confidence rubric, and anything below **80** is discarded.

## Economy rules — read before spawning anything

The full pipeline costs 3 sub-agents plus 1 verifier per bug candidate. Spend that only when the
diff deserves it:

- **Fail fast, spawn late.** Resolve the ref and confirm the diff is non-empty *before* spawning.
  A bad ref should cost one `git rev-parse`, not three sub-agents.
- **Small-diff fast path.** Under ~50 changed lines across 1-2 files: skip sub-agents entirely.
  Run all three axes yourself inline, then apply the verification rubric
  ([`references/rubric.md`](references/rubric.md)) to your own bug candidates before reporting.
  Same output format, a fraction of the cost.
- **No spec → no Spec agent.** Spawn two agents and report "no spec available" for that axis.
- **Zero bug candidates → skip the verification phase** entirely.
- **Never run builds, typecheckers, or tests yourself.** Gates run separately, and anything a
  linter/compiler would catch is a false positive by definition (the rubric says so).
- Verifiers get the **relevant diff excerpt**, never the whole diff.

## Process

### 1. Pin the fixed point

Whatever the user supplied — SHA, branch, tag, `main`, `HEAD~5`. If they didn't supply one,
inspect the branch first (`git log --oneline -5`, current branch vs default branch) and
**recommend** the obvious candidate (usually the merge-base with `main`) instead of asking cold.

Capture once: `git diff <fixed-point>...HEAD` (three-dot — compares against the merge-base) and
`git log <fixed-point>..HEAD --oneline`. Confirm the ref resolves (`git rev-parse`) and the diff
is non-empty. Failures stop here.

### 2. Identify the spec source

In order: issue references in the commit messages (`#123`, `Closes #45` — fetch with
`gh issue view` when the repo has a GitHub remote); a path the user passed; a PRD/spec/plan under
`docs/`, `specs/`, `plans/`, or `.scratch/` matching the branch or feature name; otherwise ask.
If the user says there is no spec, the Spec axis is skipped and the report says so.

One degenerate case: when the range is a single commit whose PR/commit body merely restates the
diff, that body is not an independent requirements source — use the linked issue as the spec, and
if there's only the commit message, report "no independent spec" rather than verifying the diff
against a paraphrase of itself.

### 3. Identify the standards sources

Anything in the repo that documents how code should be written: `CLAUDE.md`,
`CODING_STANDARDS.md`, `CONTRIBUTING.md`, `docs/` conventions. On top of whatever the repo
documents, the Standards axis always carries the **smell baseline** — read
[`references/smells.md`](references/smells.md) now; you will paste it into the Standards prompt
(the sub-agent has no other access to it). Two rules bind the baseline: **the repo overrides**
(a documented standard wins over the baseline), and **every smell is a judgement call**, never a
hard violation. Skip anything tooling already enforces.

### 4. Spawn the sub-agents — one message, all in parallel

Use `general-purpose` sub-agents. Every prompt includes the diff command and the commit list.

**Standards** — plus the standards sources (paths, plus a condensed summary of the key rules —
saves the sub-agent from re-reading everything) and the smell baseline pasted in full. Brief:
"Report, per file/hunk: (a) every place the diff violates a documented standard — cite the
standard (file + rule); (b) any baseline smell — name it and quote the hunk. Distinguish hard
violations (documented) from judgement calls (smells). Skip anything tooling enforces.
Under 400 words."

**Spec** — plus the spec contents or path. Brief: "Report: (a) requirements the spec asked for
that are missing or partial; (b) behaviour the spec didn't ask for (scope creep); (c) requirements
that look implemented but wrong. Quote the spec line for each finding. Under 400 words."

**Bugs** — plus the false-positive catalog pasted from
[`references/rubric.md`](references/rubric.md). Brief: "Read the changed hunks — not the whole
repo — and hunt for real defects: logic errors, broken invariants, mishandled edge cases, wrong
API usage. Then add the history lens on the modified regions: `git log --follow -p` and
`git blame` — look for (a) hunks that re-introduce something a past commit deliberately removed
or fixed, (b) invariants stated in old commit messages or nearby code comments that the change
violates. Return numbered candidates: `file:line`, one-line description, category
(logic / history / comment-contract), and the evidence quote. Skip everything in the
false-positive catalog. Under 400 words."

### 5. Verify the bug candidates

For each candidate, spawn an independent verifier — all in one message. Each verifier receives
the candidate, the relevant diff excerpt, and the rubric from
[`references/rubric.md`](references/rubric.md) **verbatim**. It returns a 0-100 score plus a
one-line justification. Discard anything under 80 from the findings. Tell the report how many
candidates were filtered — transparency about the kill rate is part of the value. A near-miss
(75-79 with solid evidence) may appear as a single "noted, below threshold (score N)" line in the
Bugs section; it never enters the counts or the recommendation.

The verifier is independent on purpose: the finder is rewarded for recall and will over-report;
a fresh agent judging one claim against the rubric has no stake in keeping it alive.

### 6. Aggregate and recommend

Present the axes under `## Standards`, `## Spec`, and `## Bugs (verified)` — verbatim or lightly
cleaned, each with a one-line summary (finding count + worst issue *within that axis*). Do **not**
merge or rerank findings across axes — the separation exists so one axis can't mask another.

An adjacent discovery that fails the false-positive catalog (a pre-existing problem, say) is not
a finding — but when it's genuinely valuable, surface it as a clearly-labelled *"out of scope,
not a finding"* aside. Transparency, not inflation: it stays out of the counts and the
recommendation.

End with a mandatory **Recommendation** block: one of **ship** / **fix first** (with a short
ranked list) / **needs rework**, plus the one-sentence why. The recommendation answers "what
should the author do next" — it ranks the surviving findings by consequence, which is a different
question from "which axis is worse" and doesn't violate the no-rerank rule.

## Why three axes

A change can pass any two and fail the third: perfectly conventional code that implements the
wrong thing (Standards ✓, Spec ✗); code that does exactly what the issue asked while breaking the
project's conventions (Spec ✓, Standards ✗); spec-true, convention-true code with an off-by-one
(Bugs ✗). Reporting them separately stops one axis from masking another — and only Bugs needs
adversarial verification, because it's the only axis that *generates* claims instead of citing
them.
