# project-context-skills

[Claude Code](https://claude.com/claude-code) slash commands for understanding a
repository deeply — and for checking whether the tools that claim to understand it
actually do.

**Deep context** (`/deep-context-*`) generates **durable, multi-file engineering
context** for a repository, so any future Claude Code session can answer questions
and diagnose issues **without re-reading the whole codebase**.

**Graph audit** (`/code-graph-gap-analysis`) goes the other way: it diffs a code
graph against AST ground truth from the source it was built from, and reports which
code constructs the parser silently fails to represent.

Each `deep-context` command explores the repo (fanning out with subagents), then writes a set of
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
| `/code-graph-gap-analysis` | Auditing a code-graph parser | `code-graph-parser-gap-report.qmd` → `.html` / `.pdf` |

The `deep-context` commands accept an optional repo path or focus area, e.g.
`/deep-context-backend src/payments`. The audit command accepts graph coordinates,
e.g. `/code-graph-gap-analysis --api graph.example.com --project <uuid>`; it falls
back to a graph config file at the repo root, and asks if it still cannot resolve them.

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

### `/deep-context-*`

- **First run (full):** explores every top-level source directory, writes the
  full `context/` set, and adds a "Deep context" section to the repo `CLAUDE.md`
  plus pointers in project memory.
- **Later runs (incremental):** reads the baseline commit recorded in
  `context/00-index.qmd`, computes the changed source set (`git diff` +
  `git status`), and rewrites only the `context/*.qmd` files those changes map to
  — then bumps the baseline to the new `HEAD`.
- Every non-obvious claim in the generated context is grounded with a
  `path/to/file:line` anchor, so later edits are surgical.

### `/code-graph-gap-analysis`

Answers a narrower question: *what is this code graph missing, and does that
invalidate the answers I get from it?*

- Runs as a **dependency graph, not a sequence**: the AST census and the graph
  inventory are independent and dispatch together, as do the three audit steps that
  follow the diff. Wide steps fan out to subagents — one per construct family, per
  indirect-construct class, per hypothesis — while classification and severity
  ranking stay single-reasoner so findings remain comparable.
- Pins the comparison to the commit the graph was indexed at, and reports drift.
- Takes a **language census before scoping**, and forces an explicit scoped-in /
  declared-out disposition for every language in the repo. Polyglot repos usually
  keep infrastructure, pipeline and diagram definitions in their own top-level trees,
  where application-shaped scope axes never reach them — so a language absent from the
  graph is reported as a headline finding with its file count, not lost in the
  sampling.
- Narrows to **3–4 representative packages** chosen to exercise different parsing
  problems (DI/decorators, ORM, API schema, UI), because depth of evidence beats
  breadth of sampling.
- Builds independent ground truth with a **resolution-capable** AST frontend
  (ts-morph against the real tsconfig for TS/JS) — not regex, and not a
  syntax-only parser, because the highest-value finding is usually whether the
  parser *resolves* symbols across barrels, aliases, and DI tokens.
- Classifies every gap as missing node / missing edge / wrong attribute /
  spurious, and separates **confirmed absence** from **UNVERIFIED**.
- Every finding carries an evidence triple: source `file:line`, the graph query
  run, and the observed result.
- Ends with a per-question-type trustworthiness verdict and a remediation backlog
  ranked by (severity × yield) ÷ effort.

## License

MIT — see [LICENSE](LICENSE).
