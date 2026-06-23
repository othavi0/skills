# noctua-skills

[![skills.sh](https://skills.sh/b/othavi0/skills)](https://skills.sh/othavi0/skills)

Claude Code skills I actually use, packaged to drop into any project. Curated,
not exhaustive: each one earns its place or it is cut.

A skill is a folder with a `SKILL.md` (YAML frontmatter + instructions). Claude
Code reads the `description` to decide when to trigger; the body guides the run.
Heavier material loads on demand from a `references/` folder, so the trigger
stays cheap.

## Install

The fast path is the [skills.sh](https://skills.sh) CLI — no clone, no copy:

```bash
# all of them, into the current project
npx skills@latest add othavi0/skills

# globally, for every project
npx skills@latest add othavi0/skills --global

# or just one
npx skills@latest add othavi0/skills -s dev-up
```

Prefer to do it by hand? Skills live in `~/.claude/skills` (global) or
`.claude/skills` (one project) — copy the ones you want:

```bash
git clone https://github.com/othavi0/skills noctua-skills
cp -r noctua-skills/skills/engineering/dev-up ~/.claude/skills/
```

Either way, run `/reload-skills` (or restart Claude Code) afterwards. Invoke a
skill by its slash command, or let the model-invocable ones trigger on their own.

## The skills

### Engineering

- **[dev-up](./skills/engineering/dev-up/SKILL.md)** — `/dev-up <port>` — starts the
  current folder's dev server on a chosen port, opens one browser tab pinned to that
  port, and arms an error watcher, then hands control back. Built for running several
  servers and tabs in parallel without disturbing them. First run on a machine
  bootstraps the claude-in-chrome connection itself.
- **[claude-md-prune](./skills/engineering/claude-md-prune/SKILL.md)** — *auto* — subtractive
  audit of `CLAUDE.md`: cuts derivable content and flags drift (paths, commands, ADRs no
  longer matching the code), following the Boris Cherny / Anthropic filter — *"would
  removing this cause Claude to make mistakes? If not, cut it."* Triggers when you mention
  trimming or auditing `CLAUDE.md`.

### Writing

- **[humanize-pt-br](./skills/writing/humanize-pt-br/SKILL.md)** — *auto* — removes AI tells
  from Brazilian-Portuguese prose: 30+ verified patterns (inflated vocabulary, formulaic
  triggers, impersonal passive, negative parallelism, sycophancy). Wikipedia's *Signs of AI
  writing* adapted to PT-BR, plus Strunk. Triggers when editing PT-BR prose.

## Structure

```
skills/
  engineering/
    claude-md-prune/   SKILL.md + references/
    dev-up/            SKILL.md + references/
  writing/
    humanize-pt-br/    SKILL.md + patterns-pt-br.md
```

## Notes

- `dev-up` needs the [claude-in-chrome](https://docs.claude.com/en/docs/claude-code) browser connection — on the first run on a machine it walks through the one-time setup itself (`references/setup.md`), no separate command.
- `humanize-pt-br` is scoped to Read / Write / Edit and is Portuguese-only by design.

---

Part of the **noctua** toolset, alongside [agent-bar](https://github.com/othavi0/agent-bar). More at [othavio.com](https://othavio.com).
