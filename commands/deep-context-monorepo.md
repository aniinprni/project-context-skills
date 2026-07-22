---
description: Generate deep, multi-file engineering context for a monorepo (root + per-package)
argument-hint: "[optional: package name to focus, or 'all']"
---

# Generate deep, multi-file engineering context (MONOREPO)

You are documenting a MONOREPO so any future Claude Code session can answer
questions accurately WITHOUT re-reading everything. Produce TWO layers of
context: (A) a repo-wide layer at the root `context/`, and (B) a per-package
layer inside each package. Explore exhaustively.

Optional focus from the user (a package name, or `all`): $ARGUMENTS
If a package is named, still generate the root layer, then go deep only on that
package. If `all` or empty, cover every package.

## Mode selection — FIRST, decide full vs. incremental
- If **no root `context/` directory exists** → FULL generation (everything below).
- If **root `context/` already exists** → INCREMENTAL UPDATE (this section), unless the user asked for a full regenerate.

### Incremental update (diff-aware)
1. Read the baseline commit from the root `context/00-index.qmd` ("Generated from commit `<sha>`"). If absent, ask or fall back to FULL.
2. Changed set: `git diff --name-only <baseline>..HEAD` + `git status --porcelain`. If not git / unresolvable, ask which packages changed.
3. Empty changed set → report "already up to date at `<HEAD>`", stop — UNLESS a package's per-package set is missing files this command now requires (symbol-reference / diagnosis-playbook / api-reference) or predates the depth requirements. That is a SCHEMA UPGRADE: (re)generate the missing/shallow files against current HEAD even without source changes.
4. Bucket each changed path by owning package (use the authoritative workspace list). Then:
   - **Root layer:** if `package.json`/workspace config/task-runner config or cross-package contracts changed → update the affected root files (`02-workspace-and-tooling`, `03-package-inventory`, `04-dependency-graph`, `05-shared-contracts`, `06-build-release-and-ops`). If a package was added/removed → always update `03-package-inventory` + `04-dependency-graph`.
   - **Per-package layer:** for each package with changes, run its per-package update using the BACKEND or FRONTEND change→file map (by package type) against that package's own `context/`.
5. Always: bump `date:` on rewritten files; update root `00-index.qmd` `date:` + "Generated from commit" sha + the coverage checklist (packages ↔ per-package context).
6. Report baseline→HEAD, changed packages, root + per-package files updated, and what was left unchanged.
7. Touch memory only if the package/file SET changed or a recorded high-level fact is now wrong.

## Operating rules
- EXPLORE FIRST, WRITE SECOND. Never guess. Cite sources as `path:line`.
- Detect the workspace tooling first: pnpm/npm/yarn workspaces, Nx, Turborepo,
  Lerna, Rush, Bazel, Gradle, Cargo, Go workspaces. Read the root manifest(s)
  and workspace config to enumerate packages authoritatively.
- Classify each package: app | service | shared-library | tooling | config |
  e2e/tests. Determine its type (backend / frontend / cli / infra) to decide
  which per-package template to apply.
- Facts (package names, scripts, task pipelines, internal deps) must match
  config exactly. Flag anything unverifiable as "UNVERIFIED".
- Use subagents to process packages IN PARALLEL — one subagent per package that
  returns a structured summary; you synthesize the root layer and write files.
  For a large monorepo, prefer running this as a workflow (fan out per package,
  then synthesize).
- Be precise over verbose. Tables and diagrams over prose.

## Layer A — Repo-wide context under root `context/`
1. `context/00-index.qmd` — Master index: what the monorepo is, TOC of all
   root files AND links to each package's `context/00-index`, "read me first",
   "how to regenerate", glossary, and a COVERAGE CHECKLIST listing every package
   and whether it has per-package context.
2. `context/01-overview.qmd` — Purpose, the product(s) the monorepo builds,
   who owns what, and a system-level mermaid diagram showing apps, services,
   shared libs, and external systems.
3. `context/02-workspace-and-tooling.qmd` — Workspace manager + version, task
   runner/pipeline config (Nx/Turbo targets, caching, affected builds), root
   scripts, node/tool version pinning, linting/formatting, and CI overview.
4. `context/03-package-inventory.qmd` — TABLE of every package: name, path,
   type, purpose, public entry points, owning team (if known), and build/test
   commands.
5. `context/04-dependency-graph.qmd` — INTERNAL dependency graph between
   packages (mermaid graph), build order, shared libraries and their consumers,
   and any circular-dependency risks. Also summarize notable third-party deps
   shared across packages and version-alignment concerns.
6. `context/05-shared-contracts.qmd` — Cross-package contracts: shared types/
   schemas/DTOs, API contracts between apps and services, event/message
   contracts, design-system/component libraries, and config packages. Note the
   source of truth for each contract.
7. `context/06-build-release-and-ops.qmd` — Build/deploy artifacts per
   deployable, versioning/release strategy (changesets/lerna/independent), CI/CD
   pipelines, environments, and cross-cutting ops (Docker, k8s, infra-as-code).
8. `context/07-cross-cutting.qmd` — Repo-wide conventions: auth, logging,
   observability, config/env strategy, testing standards, and code-sharing
   rules that apply across packages.
9. `context/08-glossary-and-open-questions.qmd` — Glossary + "UNVERIFIED"/"TODO"
   items spanning the repo.

## Layer B — Per-package context under `<package-path>/context/`
For EACH in-scope package, create a per-package `context/` folder. Choose the
template by package type:
- Backend/service package → apply the BACKEND deep-context structure (overview,
  setup, config, architecture, data-model, api-surface, services,
  integrations/async, cross-cutting, ops, testing, glossary, **symbol-reference,
  diagnosis-playbook**).
- Frontend/app package → apply the FRONTEND deep-context structure (overview,
  setup, config, architecture, routing, state, api/integration, design-system,
  styling, cross-cutting, build/perf, testing, glossary, **symbol-reference,
  diagnosis-playbook**).
- Shared library / tooling package → a lighter set:
  `00-index.qmd` (purpose, public API surface, consumers, how to extend),
  `01-api-reference.qmd` (EVERY exported module/function/type with signature +
  `file:line` + behavior — the answer-completeness backbone),
  `02-usage-and-conventions.qmd` (patterns, do/don't, versioning), and
  `03-diagnosis-and-open-questions.qmd` (failure modes → origin `file:line` → fix,
  plus open questions).
Every per-package `00-index.qmd` must link BACK to the root
`../../context/00-index.qmd` and note this package's internal dependencies.

## Depth requirements (answer-complete & diagnosis-complete)
The goal: a future session can ANSWER any question and DIAGNOSE any issue about
any package from its `context/` alone, opening source only to EXECUTE a change.
Every per-package file must capture BEHAVIOR (control flow, invariants, error &
edge-case handling, data flow) with precise `file:line` anchors — not just
structure. The per-package `14-symbol-reference.qmd` / `01-api-reference.qmd`
must list every significant symbol. Completeness over brevity.

## Coverage requirement
Every package from the authoritative workspace list must appear in
`context/03-package-inventory.qmd`, and every in-scope package must have a
per-package `context/` folder. The root coverage checklist must show nothing
silently omitted; explicitly list any package intentionally skipped and why.

## Finalize — one-command setup (ALWAYS do this, full OR incremental run)
After the root + per-package `context/` files are written, wire the monorepo to
USE them automatically. Idempotent (update in place; never clobber other content):

1. **Root repo `CLAUDE.md`** (create at monorepo root, or MERGE a "## Deep
   context" section):
   - When answering ANY question / explaining / troubleshooting: read the ROOT
     `context/00-index.qmd` first, automatically, WITHOUT asking; then, for a
     package-specific question, read that package's `context/00-index.qmd` and its
     relevant files (incl. its `14-symbol-reference.qmd` / `01-api-reference.qmd`
     for behavior and diagnosis file for fixes). Answer/diagnose FROM context.
   - Do NOT open source for Q&A or diagnosis. Read/edit source ONLY when the user
     explicitly asks to implement/execute a change. If context is insufficient or
     marked UNVERIFIED, say so and ask before falling back to source.
   - Reading `context/` (root and per-package) is pre-approved — never confirm.
   - Add a "keeping context fresh" line: re-run `/deep-context-monorepo` to update.

2. **Project memory** (per your memory instructions; memory dir derived from the
   monorepo's launch directory). MERGE, do not clobber:
   - A `reference` memory: "monorepo has a root + per-package `context/` set; read
     root `context/00-index.qmd` first" + the baseline commit.
   - A `feedback` memory for the context-first workflow: answer/diagnose from
     context files, read/edit source only when explicitly asked to execute.
   - Add matching one-line pointers to `MEMORY.md`.

## Final step
Print a short summary: workspace tooling detected, packages found vs. documented,
root + per-package files created/updated, whether `CLAUDE.md` + memory were
created or updated, and any gaps flagged UNVERIFIED.
