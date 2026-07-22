---
description: Generate deep, multi-file engineering context for a backend/API repo
argument-hint: "[optional: repo path or focus area]"
---

# Generate deep, multi-file engineering context (BACKEND)

You are documenting this backend codebase so any future Claude Code session can
answer questions accurately WITHOUT re-reading the whole repo. Explore the
application exhaustively and write durable context split across multiple files
under a `context/` directory at the repo root.

Optional focus/scope from the user: $ARGUMENTS

## Mode selection — FIRST, decide full vs. incremental
- If **no `context/` directory exists** at the repo root → FULL generation (do everything below).
- If **`context/` already exists** → INCREMENTAL UPDATE (this section), unless the user explicitly asked for a full regenerate.

### Incremental update (diff-aware)
1. Read the baseline commit recorded in `context/00-index.qmd` (the "Generated from commit `<sha>`" line). If absent, ask the user for a baseline or fall back to FULL.
2. Compute the changed source set:
   - Committed: `git diff --name-only <baseline>..HEAD`
   - Uncommitted: `git status --porcelain` (working tree + staged).
   - If the repo isn't git or the baseline is unresolvable, ask the user which areas changed instead of guessing.
3. If the changed set is empty → report "context is already up to date at `<HEAD sha>`", touch nothing, stop — UNLESS the existing set is missing files this command now requires (e.g. `14-symbol-reference.qmd`, `15-diagnosis-playbook.qmd`) or predates the depth requirements below. That is a SCHEMA UPGRADE: (re)generate the missing/shallow files against current HEAD even though source did not change, then update `00-index` accordingly.
4. Map each changed path to the affected context file(s) and re-explore ONLY those subsystems (subagents scoped to the changed files), then rewrite just those `.qmd` files. Change→file map:
   - `src/controllers/**`, `src/routes.ts`, `src/core.ts` → `07-api-surface.qmd` (+ `04` if routing/auth mechanics changed)
   - `src/entities/**`, `src/migrations/**` → `05-data-model.qmd`
   - `src/graph-entities/**`, `src/services/graphs/**`, `src/configs/graph-db.ts`, `vector-index.config.ts` → `06-domain-graphs-or-core-model.qmd`
   - `src/services/**` (non-graph), `src/services/checkpoint/**` → `08-services-and-business-logic.qmd`
   - `src/services/aws|sqs|*consumer*|git|llm|keycloak|confluence|jira*`, `src/workers/**`, `src/agents/**` → `09-integrations-and-async.qmd`
   - `src/middlewares/**`, `src/utils/**`, `src/enums/**`, `src/validations/**` → `10-cross-cutting.qmd` (+ `13` enum list)
   - `.env.example`, `src/configs/**` → `03-configuration.qmd`
   - `package.json`, `Dockerfile`, `*deploy*.yaml`, `nodemon.json`, `tsconfig.json`, `src/scripts/**` → `02-setup-and-run.qmd` + `11-operations.qmd`
   - test files / config → `12-testing.qmd`
   - **ANY changed `src/**` file → ALSO refresh the affected module sections of
     `14-symbol-reference.qmd` (add/update/remove the symbols in those files) and
     `15-diagnosis-playbook.qmd` (revisit failure modes for that subsystem).**
     These two deep files must never lag the source they describe.
5. Always also: bump the `date:` of every file you rewrite; re-check `13-glossary-and-open-questions.qmd` (resolve/add open questions the change affects); and update `00-index.qmd` — its `date:`, the "Generated from commit" line to the new HEAD sha, and the coverage checklist if files were added/removed.
6. Report: baseline→HEAD, changed source files, context files updated, and context files intentionally left unchanged.
7. Do NOT touch memory unless the file SET changed (files added/renamed/removed) or a recorded high-level fact is now wrong.

## Operating rules
- EXPLORE FIRST, WRITE SECOND. Never guess. Every non-obvious claim must be
  grounded in a real file. Cite sources as `path/to/file.ts:line`.
- Verify before you write: open the actual code, entities, configs, routes,
  migrations, env files, and tests. If ambiguous, read more.
- Be precise over verbose. Prefer tables, hierarchies, and short explanations a
  new engineer can act on. No filler, no marketing language.
- Facts (routes, enums, queue names, roles, env vars) must match code exactly.
  Flag anything you could not verify as "UNVERIFIED".
- Use subagents (Explore/general-purpose) to fan out across subsystems in
  parallel; each returns findings, you synthesize into the files.
- Prefer dedicated search/read tools over shell cat/grep where possible.

## Output: write these files under `context/` (Quarto `.qmd`, HTML-renderable)
Each file starts with YAML frontmatter (`title`, `date`) and a one-line "Scope"
note, and is self-contained.

1. `context/00-index.qmd` — Master index: one-paragraph app description, TOC
   linking every other file with a one-line hook, "read me first" note,
   "how to regenerate", glossary, and a COVERAGE CHECKLIST mapping every
   top-level source directory to the file(s) that document it.
2. `context/01-overview.qmd` — Purpose, business domain, users/personas, runtime
   stack + exact versions, external dependencies, and a mermaid diagram of
   request flow and data stores.
3. `context/02-setup-and-run.qmd` — Prereqs, env setup, every script and what it
   does, entry-point boot chain (file by file), ports, health/metrics/docs
   endpoints, local-dev gotchas.
4. `context/03-configuration.qmd` — Every env var (name, purpose, default,
   required?, subsystem), grouped by area, derived from `.env.example` AND the
   config loader/validator. Note secrets and feature flags.
5. `context/04-architecture-and-conventions.qmd` — Layering rules, dependency
   direction, routing/DI mechanism (decorators/metadata/middleware chain),
   auth/authorization model and roles, error/response conventions, and
   "how to add a new entity/service/controller/DTO" recipes.
6. `context/05-data-model.qmd` — Relational schema: every entity/table, key
   columns, relationships (mermaid ER diagram), enums, migration story, base
   entity + shared columns.
7. `context/06-domain-graphs-or-core-model.qmd` — The core domain model
   (aggregates, graph node hierarchies if any, how they relate, which services
   build/query them, indexes, embeddings). One subsection per domain area.
8. `context/07-api-surface.qmd` — Complete endpoint inventory grouped by
   controller/router: method, path, auth/role requirements, request DTO,
   response shape, and the service method it calls. Derive from the actual
   handlers + route registration, not only swagger/openapi.
9. `context/08-services-and-business-logic.qmd` — Catalog of services grouped by
   domain: responsibility, key methods, collaborators. Call out where behavior
   actually lives.
10. `context/09-integrations-and-async.qmd` — External integrations (auth, cloud,
    third-party/git/LLM APIs) plus all async machinery: queues, workers,
    consumers, webhooks, schedulers, and the events/messages flowing through
    them (names + directions).
11. `context/10-cross-cutting.qmd` — Logging, observability/metrics/tracing,
    security/redaction, request context, rate limiting, file handling,
    caching, transactions, background jobs/recovery.
12. `context/11-operations.qmd` — Build/deploy artifacts (Docker, k8s/yaml, CI),
    runtime topology, scaling notes, and a runbook (recovery jobs, maintenance
    scripts, known failure modes).
13. `context/12-testing.qmd` — Test frameworks, structure, how to run, coverage
    posture, fixtures/mocks, and gaps.
14. `context/13-glossary-and-open-questions.qmd` — Domain glossary + list of
    unclear/risky/undocumented items ("UNVERIFIED"/"TODO") to confirm.
15. `context/14-symbol-reference.qmd` — The **answer-completeness backbone**. A
    per-module reference so questions can be answered WITHOUT opening `src/`.
    Group by directory/module; for EVERY significant class and exported/public
    function/method record: fully-qualified name, `file:line`, signature
    (params + return type), a one-to-three-line description of WHAT IT DOES and
    HOW (key branches, side effects, external calls, thrown errors), and its
    collaborators. Also include a **data-flow / call-chain** note for each
    important entry point (request → controller → service → data store, or
    consumer → handler → effect). Prefer many terse rows over prose. This file
    may be large; that is expected and correct.
16. `context/15-diagnosis-playbook.qmd` — The **diagnosis-completeness layer** so
    Claude can propose fixes from context alone. For each subsystem list: likely
    failure modes / error signatures, the invariant or assumption that breaks,
    the exact origin site (`file:line`), the blast radius (other places
    affected), and the fix approach — WITHOUT needing to open source. Include a
    "symptom → suspect location(s)" lookup table and note any known foot-guns,
    race conditions, ordering constraints, and silent-failure paths.

## Depth requirements (answer-complete & diagnosis-complete)
The goal is that a future session can ANSWER any question and DIAGNOSE any issue
about this repo from `context/` alone, opening `src/` only to EXECUTE a change.
Therefore, beyond structure, every file must capture BEHAVIOR:
- **Logic & control flow** — the meaningful branches, loops, guards, and ordering
  constraints of each important function (not just its name).
- **Invariants & assumptions** — what must hold true; what breaks it.
- **Error & edge-case handling** — thrown/caught errors, retries, fallbacks,
  null/empty/oversize/duplicate handling, and silent-failure paths.
- **Data flow** — how data moves across layers and process/queue boundaries.
- **Precise `file:line` anchors** on every claim so the later execution step is
  surgical.
Completeness over brevity here: it is better for a context file to be long than
for Claude to have to read source to answer or troubleshoot. Only genuinely
line-level edits (the execution step) should require opening `src/`.

## Coverage requirement
Before finishing, verify you accounted for EVERY top-level source directory and
EVERY controller/entity/service/worker/migration — and that
`14-symbol-reference.qmd` lists every significant class/function. The coverage
checklist in `00-index.qmd` must show nothing silently omitted.

## Finalize — one-command setup (ALWAYS do this, full OR incremental run)
After the `context/` files are written, wire the repo to USE them automatically.
Do this idempotently (update in place; never duplicate or clobber other content):

1. **Repo `CLAUDE.md`** (create at repo root, or MERGE a "## Deep context"
   section into an existing one):
   - When answering ANY question / explaining / troubleshooting this repo: read
     `context/00-index.qmd` first, automatically, WITHOUT asking permission; then
     read the relevant `context/*.qmd` and answer/diagnose FROM those files.
   - Route symbol/behavior questions → `context/14-symbol-reference.qmd`;
     bug/error/"why is X failing"/fix questions → `context/15-diagnosis-playbook.qmd`;
     endpoints → `07-api-surface.qmd`; data model → `05-...`; graphs → `06-...`.
   - Do NOT open `src/` for Q&A or diagnosis. Read/edit `src/` ONLY when the user
     explicitly asks to implement/execute a change. If context is insufficient or
     marked UNVERIFIED, say so and ask before falling back to source.
   - Reading `context/` is pre-approved — never pause to confirm.
   - Add a "keeping context fresh" line: re-run `/deep-context-backend` to update.

2. **Project memory** (per your memory instructions; memory dir derived from this
   repo's launch directory). MERGE, do not clobber:
   - A `reference` memory: "deep `context/` set exists for this repo; read
     `context/00-index.qmd` first" + the baseline commit.
   - A `feedback` memory for the context-first workflow: answer/diagnose from
     context files, read/edit source only when explicitly asked to execute.
   - Add matching one-line pointers to `MEMORY.md`.

## Final step
Print a short summary: files created/updated, totals (endpoints/entities/services/
symbols documented), whether `CLAUDE.md` + memory were created or updated, and any
gaps flagged UNVERIFIED.
