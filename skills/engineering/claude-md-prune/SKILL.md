---
name: claude-md-prune
description: Audit and aggressively prune existing CLAUDE.md files following the Boris Cherny + Anthropic canonical filter ("would removing this cause Claude to make mistakes? If not, cut it"). Detects drift between docs and code (paths, commands, enums, ADRs no longer matching reality). Use whenever the user mentions CLAUDE.md being too long, bloated, stale, outdated, or wants to refactor/audit/trim/refine/prune/shrink CLAUDE.md files. Also triggers when the user says Claude is ignoring CLAUDE.md instructions, when CLAUDE.md exceeds ~200 lines, when there is evidence of drift between docs and code, or when the user asks for "Boris-style", "mistakes-only", or "minimal" CLAUDE.md. Subtractive audit (cuts derivable content + flags factual errors); complementary to claude-md-improver (which is additive). Works on any project, any language, any stack.
---

# CLAUDE.md Prune — Subtractive Audit

This skill aggressively removes content from CLAUDE.md files that Claude can already infer from the code, and flags claims in the doc that no longer match reality. It is the **opposite axis** of `claude-md-improver` — that one adds what's missing; this one removes what shouldn't be there.

## Why this skill exists

Two sources converge on the same operational philosophy:

**Anthropic best-practices** ([code.claude.com/docs/en/best-practices](https://code.claude.com/docs/en/best-practices)):

> Treat CLAUDE.md like code: review it when things go wrong, prune it regularly, and test changes by observing whether Claude's behavior actually shifts.
>
> For each line, ask: "Would removing this cause Claude to make mistakes?" If not, cut it.
>
> Bloated CLAUDE.md files cause Claude to ignore your actual instructions.

**Boris Cherny** (creator of Claude Code — [howborisusesclaudecode.com](https://howborisusesclaudecode.com)):

> Minimal configuration, zen-like restraint: the file should contain only rules that fix recurring mistakes. Not documentation. Not architecture explainers. Not coding standards Claude already knows.
>
> Ruthlessly edit over time — keep iterating until Claude's mistake rate measurably drops.

The CLAUDE.md is **advisory (~70-80% adherence)**. For rules that must execute 100% of the time, use a **hook** instead. The longer the file, the more individual rules dilute and adherence drops. This skill restores signal-to-noise.

## When to trigger

Trigger on any of these:

- User says "audit", "refine", "trim", "prune", "shrink", "refactor" CLAUDE.md
- User describes CLAUDE.md as "too long", "bloated", "stale", "outdated", "messy"
- User says Claude is "ignoring CLAUDE.md", "didn't follow the rule", "asked something that was in the docs"
- User mentions "Boris-style" / "mistakes-only" / "minimal" CLAUDE.md
- User asks for a project audit or documentation audit and CLAUDE.md files exist
- During unrelated work, you notice a CLAUDE.md file is > 200 lines AND has obvious derivable content (stack lists, directory trees, command listings) — proactively offer.

**Do NOT trigger** when the user wants to:

- ADD content to CLAUDE.md → use `claude-md-improver`
- Capture session learnings → use `/revise-claude-md` command
- Generate a CLAUDE.md from scratch → use `claude-md-improver` templates

## What this skill does vs sibling skills

| Skill | Direction | When |
|---|---|---|
| `claude-md-improver` | **ADD** | Audit quality, propose additions, A-F grade |
| `/revise-claude-md` (command) | **CAPTURE** | End-of-session learnings into CLAUDE.md |
| `claude-md-prune` (this) | **REMOVE + FACT-CHECK** | Cut derivable content; flag drift between docs and code |

The three are complementary. Run `claude-md-prune` periodically (every 1-3 months, or whenever a CLAUDE.md grows beyond ~200 lines).

## Workflow

Five phases. Always present the report BEFORE applying changes.

### Phase 1: Discovery

Find every CLAUDE.md in scope. Default scope is the current project.

Locations to check:

- `./CLAUDE.md` (project root)
- `./CLAUDE.local.md` (personal, gitignored — may or may not exist)
- Any `./**/CLAUDE.md` in monorepos (workspace/package-level)
- `~/.claude/CLAUDE.md` (global) — **only if user explicitly asks for global scope**
- `~/.claude/memory/*.md` (auto-memory) — **only if user explicitly asks**. Same Boris filter applies: each memory file should answer "would removing this make Claude make mistakes / lose non-obvious knowledge?" Cut derivable filesystem listings, memories that point back to CLAUDE.md without adding info, generic philosophy already known to Claude. KEEP specific gotchas, citations of real incidents, decision rationales, and project context not in the code.

For each file: record line count and approximate size. **Files > 200 lines are prime candidates** for aggressive pruning. Files < 50 lines might still benefit from drift detection but rarely need cuts. Memory files are typically 10-50 lines each — apply per-file: does the description claim non-obvious knowledge that the body actually delivers, or is the body just paraphrasing the description? If paraphrasing, cut.

Ask the user to confirm the scope before proceeding. Default: project only.

### Phase 2: Drift detection (fact-checking against code)

For each CLAUDE.md, verify that the **claims in the doc still match the code**. This is the most valuable phase — silent drift is a more common failure mode than bloat.

Detect the project's primary manifest file(s) so checks adapt to the stack:

| Manifest | Stack |
|---|---|
| `package.json` | JS / TS / Node / Bun |
| `Cargo.toml` | Rust |
| `pyproject.toml` / `requirements.txt` / `uv.lock` | Python |
| `go.mod` | Go |
| `Gemfile` | Ruby |
| `pom.xml` / `build.gradle` | Java / Kotlin |
| `composer.json` | PHP |
| `mix.exs` | Elixir |
| `Package.swift` | Swift |
| `pubspec.yaml` | Dart / Flutter |

For each factual claim in the CLAUDE.md, verify:

- **Paths** mentioned exist on the filesystem
- **Commands** referenced (e.g., `bun db:sync`, `cargo run`) appear in the manifest's scripts/tasks
- **Functions, types, classes, enums, constants** cited via Grep — values still match what's documented
- **ADRs / decision documents** referenced still exist; check if a newer ADR has superseded the workflow described
- **Environment variables** documented still appear in the env validation source (zod schema, dotenv-vault, T3 env, etc.)
- **External dependencies** mentioned (e.g., "uses XYZ library") still appear in the manifest

Use parallel tool calls (multiple Read/Grep in one message) since these checks are independent.

See `references/drift-checks.md` for the full checklist per stack and example evidence patterns.

### Phase 3: Categorize content (the Boris filter)

For each section or bullet in the CLAUDE.md, apply the canonical test:

> **Would Claude make mistakes if I removed this line?**

If "no" → it's noise. If "yes" → preserve. When unclear, default to **ASK the user** rather than cut blindly; the user may know history (past bugs, specific incidents) that explains why an apparently generic rule exists.

Classify each section/bullet into one of five buckets:

#### CUT (derivable from code or generic knowledge)

Common patterns to remove:

- **Stack / dependency lists** (read the manifest)
- **Directory structure trees** (read the filesystem)
- **Command listings** (read manifest scripts)
- **Environment variable lists** (read env validation source)
- **Architecture descriptions / explainers** (Claude reads the code)
- **Per-folder descriptions** ("what each folder does")
- **Tutorials, onboarding instructions** (belongs in README/docs/)
- **Generic best-practices** Claude already knows ("use TypeScript", "write clean code", "follow REST conventions", "use proper error handling")
- **Generic linter rules** (the linter config exists)
- **Long verbose explanations** when a one-liner would do
- **Code style** that mirrors what the language/framework defaults already enforce

See `references/cut-criteria.md` for detailed examples per category.

#### KEEP (mistakes-recurrent or genuinely non-obvious)

These survive the filter:

- **P0 invariants** (e.g., "never import X from Y", "never set cookie domain to Z")
- **Gotchas with specific symptom + workaround** (framework bugs, version-specific issues, library quirks)
- **Anti-patterns banned with reason** (when language/framework default would be a bug here)
- **Non-obvious architectural decisions** (server-wins on conflict, ownership rules, isolation boundaries)
- **Specific bugs Claude has made before** (the compounding-engineering log)
- **Cross-cutting rules** that span multiple files

See `references/keep-criteria.md` for examples and rationale per category.

#### MOVE TO `CLAUDE.local.md` (personal/gitignored)

- Personal preferences ("I prefer succinct responses", "I like Portuguese explanations")
- Per-user paths or aliases
- Anything that should not be shared with the team

#### MOVE TO WORKSPACE-LOCAL CLAUDE.md

In a monorepo, push down. Root `CLAUDE.md` should only carry **cross-cutting** items. A mistake that only applies to one package or workspace lives in that package's CLAUDE.md, where it's automatically loaded when Claude works in that directory tree.

#### CONVERT TO HOOK

If a rule requires 100% adherence (formatter, linter trigger, block on certain file writes), convert to a hook. Examples:

- "Always run `bun fix` after writes" → PostToolUse hook
- "Never commit without running tests" → PreToolUse hook on commit
- "Block writes to `.env`" → PreToolUse hook

Flag candidates here — actual hook creation is out of scope for this skill (suggest `update-config` skill).

### Phase 4: Quality report

ALWAYS output the full report before applying any changes. The user reviews and approves per-file.

Format:

```
## CLAUDE.md Prune Report

### Summary
- Files audited: <N>
- Total lines: <before> → <estimated after> (<−X%>)
- Drift issues found: <N>
- Files > 200 lines (prime candidates): <list>

### File: ./CLAUDE.md (root) — <before> → <estimated after> lines

#### Drift (factual errors detected)
1. References path `X/Y/Z.md` which does not exist (renamed? deleted?)
2. Cites command `bun db:generate` but `package.json` does not contain it
3. Enum declared as `['a','b','c']` but `schema/foo.ts` shows `['a','b']`
4. ADR-0004 referenced as authoritative but ADR-0009 (2026-05-18) supersedes that flow

#### Sections to CUT
- Lines 14-30 "Stack": derivable from `package.json`
- Lines 35-71 "Topology": derivable from `ls`
- Lines 75-110 "Daily Commands": derivable from `package.json` scripts
- Lines 217-228 "MCP Servers": derivable from `.mcp.json`

#### Sections to KEEP
- "Auth invariants P0" (lines 120-138): each bullet is a real isolation rule — KEEP
- "Anti-patterns banidos" (lines 250-266): each is a specific bug pattern — KEEP
- "Drizzle-kit push + TTY" (line 276): gotcha with specific symptom — KEEP

#### Sections to MOVE TO HOOK
- "Always run `bun fix` after writes" (line 282): 100% enforcement → already a PostToolUse hook (no action needed)

#### Suggested rewording
- "Schema is push-only" (line 133): ADR-0009 made this more nuanced. Suggest: "..."

### File: ./packages/api/CLAUDE.md (workspace) — <before> → <after> lines

[... per-file breakdown ...]
```

### Phase 5: Apply with approval

For each file the user approves, write the new version. Show the new version inline (or a diff for small changes) before writing — the user should see exactly what will replace the file.

After applying:

- Print the final line count reduction (this validates the work and reinforces the habit)
- Suggest the user re-run this skill periodically (every 1-3 months, or whenever a CLAUDE.md grows above ~200 lines)
- If hook candidates were identified, suggest invoking `update-config` skill to wire them up

## Anti-instructions

These are the easy ways to misuse the skill — avoid each:

- **Do not add new content.** This skill is subtractive. If you spot a missing rule, flag it in the report but do not write it. Let the user invoke `claude-md-improver` for additions.
- **Do not invent gotchas.** Only KEEP content that's already in the file. The skill does not generate new project-specific rules from imagination.
- **Do not cut without asking when unclear.** A rule that looks generic might cover an actual past incident only the user remembers. When in doubt, ask: "Did this line ever fix a real bug?"
- **Do not touch `~/.claude/CLAUDE.md` (global)** unless the user explicitly opts in. Default scope is the current project.
- **Do not delete CLAUDE.md files entirely** during prune. Even a tiny CLAUDE.md is acceptable if it carries one real gotcha. Empty files are fine; non-existence is suspicious.
- **Do not preserve content "just in case".** The whole point is that bloat reduces adherence. Aggressive is the goal — but always with the user's approval per file.

## References

- `references/cut-criteria.md` — categories of removable content with examples per stack
- `references/keep-criteria.md` — what survives the filter, with rationale
- `references/drift-checks.md` — fact-checking checklist universal across languages
