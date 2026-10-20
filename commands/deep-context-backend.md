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
3. If the changed set is empty → report "context is already up to date at `<HEAD sha>`", touch nothing, stop — UNLESS the existing set is missing files this command now requires (e.g. `14-symbol-reference.qmd`, `15-diagnosis-playbook.qmd`) or predates the depth requirements below, **or violates any "Non-negotiable output invariants" below** (stacked per-cycle sections, dated update banners, an over-budget `00-index.qmd`, inline `🆕 <sha>` delta narration, stale anchors). That is a SCHEMA UPGRADE: (re)generate the missing/shallow files against current HEAD even though source did not change — and NORMALIZE an offending set by collapsing every per-cycle section into the single current-state entry for that symbol/subsystem, deleting the history (keep at most a 10-line entry per past cycle in `context/CHANGELOG.qmd`), then update `00-index` accordingly.
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
     the symbol reference (file 14 / its shards) — add/update/remove the symbols in
     those files — and the diagnosis playbook (file 15 / its shards): revisit the
     failure modes for that subsystem. These two deep files must never lag the
     source they describe.**
   - **MERGE IN PLACE, NEVER APPEND.** Rewrite the entry that is already there so it
     describes HEAD. Do not add a "new/changed this cycle" section, a dated banner,
     or a second entry for a symbol you already documented. One symbol → exactly one
     entry, always current. Delete entries whose source is gone.
   - **RE-ANCHOR THE WHOLE FILE.** When a `src/` file changes at all, re-verify the
     `file:line` anchor of EVERY symbol documented from that file — not just the
     symbols that changed. Inserting or deleting any line shifts every anchor below
     it. Skipping this is the single most common way this context set rots.
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

## Non-negotiable output invariants
These override everything else below. A run that breaks one of these has failed,
even if every sentence it wrote is accurate.

1. **CURRENT STATE ONLY — the context set is not a changelog.** Every `.qmd`
   describes the code as it is at HEAD, not the path it took to get there.
   - NEVER append a per-cycle section, under ANY wording. Not `## New & changed
     symbols this cycle`, not `## What changed at this baseline`, not `## Changes
     at <sha>`, not `## … N baselines ago`, not a dated "incremental update to
     `<sha>`" banner, not a `*(new at <sha>)*` suffix on a heading. Renaming the
     section does not make it allowed. Edit the existing entry instead.
   - **STRUCTURAL TEST — no heading may name a commit, a baseline or a cycle.**
     `grep -rnE '^#+.*([0-9a-f]{7,40}|baseline|cycle)' context/ --include=*.qmd`
     must come back empty. That is the phrasing-proof form of the rule above.
   - NEVER narrate deltas inside a durable entry. Write "calls
     `gitService.checkUpdate`" — not "🆕 `bd20239`: now calls X (was Y)". Commit
     shas, "new this cycle", "previously", "reverted by" appear nowhere in files
     01–15.
   - If a reader greps a symbol name they must find exactly ONE description of it,
     and it must be true right now. Two entries for one symbol is a defect, because
     the stale one is usually found first.
   - If a history trail is genuinely wanted it lives in exactly ONE file,
     `context/CHANGELOG.qmd`: newest first, **one entry per run, max 10 lines**,
     whole file capped at **200 lines** — trim the oldest. No other file carries
     history.
2. **"READ FIRST" appears exactly once** in the whole `context/` set — in
   `00-index.qmd`. If everything is marked read-first, nothing is.
3. **Size budgets — measured in BYTES as well as lines.** Line counts do not
   bound read cost: a 350-line file with 230-byte lines is an 80 KB read. A file
   over budget gets split or tightened, never shipped:
   - `00-index.qmd` — **≤ 300 lines AND ≤ 25 KB**, with the routing table/TOC
     inside the first 50 lines. It is read on EVERY question; keep it cheap.
   - any other single `.qmd` — **≤ 1,500 lines AND ≤ 120 KB**. Shard rather than
     overflow (see files 14 and 15).
   - **no single line over ~400 bytes.** A multi-thousand-byte line is an
     unreadable wall AND it hides from every line-based budget — break it up.
   - the whole `context/` set should land well under the byte size of the source it
     documents (`find src -name '*.ts' -exec cat {} + | wc -c` vs
     `cat context/**/*.qmd | wc -c`). If it approaches or exceeds it, the set is
     narrating history or repeating itself — compress, do not rationalize.
     Context bigger than the code defeats the entire purpose of this command.
4. **Anchors must be true at HEAD** — see the next section.
5. **Do not number section headings.** Write `## Contexts` / `## Custom hooks`,
   never `## 12. Contexts`. Numbering corrupts the moment a section is inserted or
   removed — observed result in a real run: a playbook running §19 → §21 → … →
   §26 → §28 → §31 with `## 27. Where to start` sitting last, after §31. Order
   sections logically and let the heading text carry the meaning.

## Anchor hygiene (`file:line` must not rot)
Cited anchors are where the execution step jumps, and the `CLAUDE.md` this command
writes tells future sessions to trust them WITHOUT opening source. A confidently
wrong anchor is worse than no anchor.

- Anchor format is `path/to/file.ts:LINE` pointing at the symbol's **declaration**
  line, and the symbol's own name must appear in the same row so it stays
  greppable after drift.
- **Never carry an anchor forward unverified.** Before writing any `file:line`,
  confirm it against a fresh read of that file at HEAD. An anchor copied from the
  previous version of a context file is presumed stale.
- When a file changes, re-anchor every symbol in it (see the incremental rules).
- List symbols in source order within a module, so an out-of-order or duplicated
  anchor is visible on sight.
- **NEVER publish an anchor-drift table.** Do not write a section recording how
  far anchors moved, or telling the reader to apply a `+717` / `−12` delta to an
  older section's line numbers, or declaring an earlier section's anchors "void".
  That is knowingly shipping wrong anchors with homework attached — and after two
  updates the reader must apply the arithmetic twice, in order. If you worked out
  the drift, you already know the correct line: **write it into the entry and
  delete the old one.** The drift table is the bug, not the fix.
- The verification gate below checks this mechanically. Run it.

## Output: write these files under `context/` (Quarto `.qmd`, HTML-renderable)
Each file starts with YAML frontmatter (`title`, `date`) and a one-line "Scope"
note, and is self-contained.

1. `context/00-index.qmd` — Master index, **≤ 300 lines**: one-paragraph app
   description, TOC linking every other file with a one-line hook (inside the
   first 50 lines), "read me first" note, a SHORT "how to regenerate" note (the
   baseline commit line + the command — not a log), glossary, and a COVERAGE
   CHECKLIST mapping every top-level source directory to the file(s) that document
   it. It also records headline counts (routes/entities/services/…), which the
   verification gate re-derives from source every run. This file is read on every
   single question: no changelog, no update banners, no per-cycle history.
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
    may be large; that is expected and correct — but **if it would exceed 1,500
    lines, shard it**: move the per-module sections into `context/14-symbols/<module>.qmd`
    (e.g. `controllers.qmd`, `services-graph.qmd`, `services-ai.qmd`, `agents-workers.qmd`,
    `utils-middleware.qmd`) and keep `14-symbol-reference.qmd` as a **≤ 150-line
    router**: one line per shard saying which modules/symbols live in it, so a
    session loads one shard instead of the whole reference. Shard by module, never
    by date.
16. `context/15-diagnosis-playbook.qmd` — The **diagnosis-completeness layer** so
    Claude can propose fixes from context alone. For each subsystem list: likely
    failure modes / error signatures, the invariant or assumption that breaks,
    the exact origin site (`file:line`), the blast radius (other places
    affected), and the fix approach — WITHOUT needing to open source. Include a
    "symptom → suspect location(s)" lookup table and note any known foot-guns,
    race conditions, ordering constraints, and silent-failure paths. There is **ONE**
    symptom table for the whole repo, kept current — never one per cycle. If the
    file would exceed 1,500 lines, shard it the same way as file 14 into
    `context/15-diagnosis/<subsystem>.qmd` with `15-diagnosis-playbook.qmd` reduced
    to a ≤ 150-line router plus the single global symptom table.

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

## Verification gate — RUN these checks, do not assert them
Before finishing (full OR incremental), actually execute the checks below and print
each result in the final summary. Do not claim completeness — demonstrate it. A
check you skipped is a check that failed.

1. **Anchor check.** For every `file:line` you cite in a file you wrote this run,
   read that line and confirm the symbol named alongside it is declared there
   (±3 lines tolerance for a decorator or overload). Fix every miss.
   Report `anchors checked: N, corrected: M`.
   ```bash
   grep -ohE '[A-Za-z0-9_./-]+\.(ts|tsx|js):[0-9]+' context/*.qmd context/*/*.qmd \
     | sort -u > /tmp/anchors.txt
   # for each path:line — sed -n "${line}p" "$path" — and compare to the entry
   ```
2. **Count check.** Re-derive every count stated in `00-index.qmd` from source and
   correct the prose to match reality:
   ```bash
   grep -rhoE '@(Get|Post|Put|Patch|Delete)\(' src/controllers | wc -l   # routes
   grep -rlE '@Entity\(' src/entities | wc -l                            # entities
   ls src/migrations/*.ts 2>/dev/null | wc -l                            # migrations
   ```
3. **Symbol coverage diff.** Per module, diff source symbols against documented
   symbols — do not eyeball this:
   ```bash
   grep -oE '^    (public |private )?async [A-Za-z0-9_]+' src/controllers/X.controller.ts \
     | sed -E 's/.* //' | sort > /tmp/src.txt
   grep -oE '^- X\.[A-Za-z0-9_]+' context/14-symbols/controllers.qmd \
     | sed 's/^- X\.//' | sort > /tmp/doc.txt
   comm -13 /tmp/doc.txt /tmp/src.txt   # in source, undocumented  → MUST be empty
   comm -23 /tmp/doc.txt /tmp/src.txt   # documented, gone from source → delete these
   ```
   Repeat per controller/service/entity/worker module. Both sides must come back
   empty.
4. **Invariant check.** Confirm nothing violates the invariants:
   ```bash
   wc -l context/*.qmd context/*/*.qmd 2>/dev/null        # vs the budgets above
   grep -rc 'READ FIRST' context/ | grep -v ':0$'         # exactly ONE hit (00-index)
   # no heading names a commit / baseline / cycle — phrasing-proof, MUST be empty:
   grep -rnE '^#+.*([0-9a-f]{7,40}|baseline|cycle|🆕)' context/ --include=*.qmd \
     | grep -v CHANGELOG
   grep -rnE '^#+ *[0-9]+[.)] ' context/ --include=*.qmd   # numbered headings: MUST be empty
   awk 'length>400 {print FILENAME": line "FNR" ("length" bytes)"}' \
     context/*.qmd context/*/*.qmd 2>/dev/null              # mega-lines: MUST be empty
   wc -c context/00-index.qmd ; du -sk context/             # vs the byte budgets
   find src -name '*.ts' -exec cat {} + | wc -c           # source bytes
   cat context/*.qmd context/*/*.qmd 2>/dev/null | wc -c  # context bytes — must be well under
   ```
5. **Coverage checklist.** Every top-level source directory maps to a documenting
   file in `00-index.qmd`, with nothing silently omitted.

If a check fails and you cannot fix it this run, say so explicitly and flag it
UNVERIFIED in the summary — never silently.

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
     If 14/15 are sharded, read the router file, then ONLY the one shard that
     covers the question — never the whole reference.
   - Anchors (`file:line`) are accurate as of the recorded baseline commit. When
     executing a change, locate the symbol BY NAME in that file rather than
     trusting the line number blindly — it may have drifted since.
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
symbols documented, each re-derived from source by the verification gate), the
result of EVERY verification-gate check (anchors checked/corrected, count
mismatches fixed, coverage diffs empty or not, invariant/budget check, context
bytes vs source bytes), whether `CLAUDE.md` + memory were created or updated, and
any gaps flagged UNVERIFIED.
