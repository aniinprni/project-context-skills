# deep-context-commands

Three [Claude Code](https://claude.com/claude-code) slash commands that generate
**durable, multi-file engineering context** for a repository, so any future Claude
Code session can answer questions and diagnose issues **without re-reading the
whole codebase**.

Each command explores the repo (fanning out with subagents), then writes a set of
self-contained, cross-referenced Quarto (`.qmd`) files under a `context/`
directory at the repo root — including a per-symbol reference and a
diagnosis playbook — and wires the repo to use them automatically (repo
`CLAUDE.md` + project memory). Re-running a command performs an **incremental,
diff-aware update**: it diffs the recorded baseline commit → `HEAD` (plus the
working tree) and rewrites only the affected files.

## Commands

| Command | For | Output |
|---|---|---|
| `/deep-context-backend` | Backend / API repos | `context/00-index.qmd` … `15-diagnosis-playbook.qmd` |
| `/deep-context-frontend` | Frontend / UI repos | `context/00-index.qmd` … deep files |
| `/deep-context-monorepo` | Monorepos (root + per-package) | root `context/` + per-package context |

Each accepts an optional argument (repo path or focus area), e.g.
`/deep-context-backend src/payments`.

## Install

These are user-level (or project-level) custom slash commands. Copy the files in
`commands/` into a Claude Code commands directory:

**Globally (all your projects):**

```bash
mkdir -p ~/.claude/commands
cp commands/*.md ~/.claude/commands/
```

**Per project (checked into a repo, shared with your team):**

```bash
mkdir -p .claude/commands
cp /path/to/deep-context-commands/commands/*.md .claude/commands/
```

Restart Claude Code (or start a new session) and the commands appear as
`/deep-context-backend`, `/deep-context-frontend`, and `/deep-context-monorepo`.

## How it works

- **First run (full):** explores every top-level source directory, writes the
  full `context/` set, and adds a "Deep context" section to the repo `CLAUDE.md`
  plus pointers in project memory.
- **Later runs (incremental):** reads the baseline commit recorded in
  `context/00-index.qmd`, computes the changed source set (`git diff` +
  `git status`), and rewrites only the `context/*.qmd` files those changes map to
  — then bumps the baseline to the new `HEAD`.
- Every non-obvious claim in the generated context is grounded with a
  `path/to/file:line` anchor, so later edits are surgical.

## License

MIT — see [LICENSE](LICENSE).
