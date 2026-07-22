---
description: Generate deep, multi-file engineering context for a frontend/UI repo
argument-hint: "[optional: repo path or focus area]"
---

# Generate deep, multi-file engineering context (FRONTEND)

You are documenting this frontend codebase so any future Claude Code session can
answer questions accurately WITHOUT re-reading the whole repo. Explore the app
exhaustively and write durable context split across multiple files under a
`context/` directory at the repo root.

Optional focus/scope from the user: $ARGUMENTS

## Mode selection — FIRST, decide full vs. incremental
- If **no `context/` directory exists** → FULL generation (everything below).
- If **`context/` already exists** → INCREMENTAL UPDATE (this section), unless the user asked for a full regenerate.

### Incremental update (diff-aware)
1. Read the baseline commit from `context/00-index.qmd` ("Generated from commit `<sha>`"). If absent, ask or fall back to FULL.
2. Changed set: `git diff --name-only <baseline>..HEAD` + `git status --porcelain`. If not git / unresolvable, ask which areas changed.
3. Empty changed set → report "already up to date at `<HEAD>`", stop — UNLESS the existing set is missing files this command now requires (`14-symbol-reference.qmd`, `15-diagnosis-playbook.qmd`) or predates the depth requirements. That is a SCHEMA UPGRADE: (re)generate the missing/shallow files against current HEAD even without source changes.
4. Re-explore ONLY affected areas and rewrite just those `.qmd` files. Change→file map:
   - routing/router config, route files, `App`/shell → `05-routing-and-navigation.qmd` (+ `02`/`04` if bootstrap/structure changed)
   - store/state slices, effects/thunks/sagas, query hooks → `06-state-and-data-flow.qmd`
   - api client, services, sdk wrappers → `07-api-and-integration.qmd`
   - components / design-system dirs → `08-design-system-and-components.qmd`
   - styles, theme, tokens, tailwind/config → `09-styling-and-theming.qmd`
   - auth, i18n, a11y, feature flags, error/notify, forms → `10-cross-cutting.qmd`
   - bundler/build config, SSR/SSG config → `11-build-and-performance.qmd`
   - env/config injection → `03-configuration-and-env.qmd`
   - `package.json`/scripts → `02-setup-and-run.qmd`
   - test files/config → `12-testing.qmd`
   - **ANY changed source file → ALSO refresh the affected sections of
     `14-symbol-reference.qmd` and `15-diagnosis-playbook.qmd`.** These two deep
     files must never lag the source they describe.
5. Always: bump `date:` on rewritten files; re-check `13-glossary-and-open-questions.qmd`; update `00-index.qmd` `date:` + "Generated from commit" sha + coverage checklist if files were added/removed.
6. Report baseline→HEAD, changed files, updated context files, and those left unchanged.
7. Touch memory only if the file SET changed or a recorded high-level fact is now wrong.

## Operating rules
- EXPLORE FIRST, WRITE SECOND. Never guess. Every non-obvious claim must be
  grounded in a real file. Cite sources as `path/to/file.tsx:line`.
- Verify before you write: read the router config, component tree, state stores,
  API client layer, styling system, build config, env handling, and tests.
- Be precise over verbose. Prefer tables, route maps, and component hierarchies.
  No filler, no marketing language.
- Facts (routes, component names, store slices, env vars, design tokens) must
  match code exactly. Flag anything unverifiable as "UNVERIFIED".
- Use subagents (Explore/general-purpose) to fan out across feature areas in
  parallel; you synthesize their findings.
- Prefer dedicated search/read tools over shell cat/grep where possible.

## Output: write these files under `context/` (Quarto `.qmd`, HTML-renderable)
Each file starts with YAML frontmatter (`title`, `date`) and a one-line "Scope"
note, and is self-contained.

1. `context/00-index.qmd` — Master index: one-paragraph app description, TOC
   linking every other file with a one-line hook, "read me first",
   "how to regenerate", glossary, and a COVERAGE CHECKLIST mapping every
   top-level source directory to the file(s) documenting it.
2. `context/01-overview.qmd` — What the app does, primary user personas/roles,
   framework + exact versions (React/Vue/Angular/Next/etc.), language, package
   manager, and a mermaid diagram of the high-level app architecture
   (shell → routing → features → data layer → backend).
3. `context/02-setup-and-run.qmd` — Prereqs, install, every npm script, dev
   server, ports, environment modes, and local-dev gotchas.
4. `context/03-configuration-and-env.qmd` — Every env/config var (name, purpose,
   default, build-time vs runtime, which feature uses it), feature flags, and
   how config is injected (`.env`, vite/webpack define, runtime config).
5. `context/04-architecture-and-conventions.qmd` — Folder/feature structure,
   module boundaries, naming conventions, code-splitting strategy, error
   boundaries, and "how to add a new page/feature/component/store slice"
   recipes.
6. `context/05-routing-and-navigation.qmd` — Complete route map: path, page/
   component, layout, guards (auth/role), lazy-loading, params/query, and
   redirects. Include a mermaid nav diagram. Note persona-conditional routes.
7. `context/06-state-and-data-flow.qmd` — State management (Redux/Zustand/
   Context/Signals/etc.): stores/slices, what each owns, selectors, side-effect
   handling (thunks/sagas/effects), and server-state caching (React Query/SWR).
8. `context/07-api-and-integration.qmd` — API client layer: base client,
   endpoints consumed (method + path + calling module), auth token handling,
   interceptors, error handling, websockets/SSE, and third-party SDKs
   (analytics, auth, maps, payments, etc.).
9. `context/08-design-system-and-components.qmd` — Component inventory by atomic
   level (atoms/molecules/organisms/templates/pages) or by feature, shared/UI
   library components, reuse patterns, and notable prop contracts.
10. `context/09-styling-and-theming.qmd` — Styling approach (CSS modules/
    Tailwind/styled-components/etc.), design tokens, theme(s), dark mode,
    responsive breakpoints, and global styles.
11. `context/10-cross-cutting.qmd` — Auth flow (login/refresh/logout), i18n/
    localization, accessibility posture, feature flags, logging/telemetry,
    error/notification systems, and forms/validation approach.
12. `context/11-build-and-performance.qmd` — Bundler/build config, output
    artifacts, code-splitting/lazy boundaries, asset handling, performance
    budgets, SSR/SSG/ISR (if any), and known perf considerations.
13. `context/12-testing.qmd` — Unit/component/e2e frameworks, structure, how to
    run, mocking, visual/snapshot tests, coverage posture, and gaps.
14. `context/13-glossary-and-open-questions.qmd` — Domain/UI glossary + list of
    unclear/risky/undocumented items ("UNVERIFIED"/"TODO") to confirm.
15. `context/14-symbol-reference.qmd` — The **answer-completeness backbone**. A
    per-module reference so questions can be answered WITHOUT opening source.
    Group by feature/dir; for EVERY significant component, hook, store slice,
    util, and api function record: name, `file:line`, props/params + return/
    emitted, a one-to-three-line description of WHAT IT DOES and HOW (key
    conditional rendering, effects, state it reads/writes, side effects), and
    collaborators. Include a **render/data-flow** note per route/page
    (route → page → components → hooks → api). Terse rows over prose; large is fine.
16. `context/15-diagnosis-playbook.qmd` — The **diagnosis-completeness layer**.
    Per subsystem: likely failure modes (render errors, stale state, failed
    requests, hydration/SSR issues), the broken assumption, the origin
    (`file:line`), blast radius, and the fix approach — from context alone. Add a
    "symptom → suspect location(s)" table and note foot-guns / effect-ordering /
    race conditions.

## Depth requirements (answer-complete & diagnosis-complete)
The goal: a future session can ANSWER any question and DIAGNOSE any issue from
`context/` alone, opening source only to EXECUTE a change. Beyond structure,
every file must capture BEHAVIOR: control flow & conditional rendering,
state/data flow across components and to the backend, invariants/assumptions,
error & edge-case handling (loading/empty/error states, retries, guards), and
precise `file:line` anchors on every claim. Completeness over brevity — better a
long context file than needing to read source to answer or troubleshoot.

## Coverage requirement
Before finishing, verify you accounted for EVERY top-level source directory and
EVERY route, top-level feature, store, and shared component group — and that
`14-symbol-reference.qmd` lists every significant component/hook/store/util. The
coverage checklist in `00-index.qmd` must show nothing silently omitted.

## Finalize — one-command setup (ALWAYS do this, full OR incremental run)
After the `context/` files are written, wire the repo to USE them automatically.
Do this idempotently (update in place; never duplicate or clobber other content):

1. **Repo `CLAUDE.md`** (create at repo root, or MERGE a "## Deep context"
   section into an existing one):
   - When answering ANY question / explaining / troubleshooting this repo: read
     `context/00-index.qmd` first, automatically, WITHOUT asking permission; then
     read the relevant `context/*.qmd` and answer/diagnose FROM those files.
   - Route component/hook/behavior questions → `context/14-symbol-reference.qmd`;
     bug/error/"why is X failing"/fix questions → `context/15-diagnosis-playbook.qmd`;
     routes → `05-routing-and-navigation.qmd`; state → `06-...`; api → `07-...`.
   - Do NOT open source for Q&A or diagnosis. Read/edit source ONLY when the user
     explicitly asks to implement/execute a change. If context is insufficient or
     marked UNVERIFIED, say so and ask before falling back to source.
   - Reading `context/` is pre-approved — never pause to confirm.
   - Add a "keeping context fresh" line: re-run `/deep-context-frontend` to update.

2. **Project memory** (per your memory instructions; memory dir derived from this
   repo's launch directory). MERGE, do not clobber:
   - A `reference` memory: "deep `context/` set exists for this repo; read
     `context/00-index.qmd` first" + the baseline commit.
   - A `feedback` memory for the context-first workflow: answer/diagnose from
     context files, read/edit source only when explicitly asked to execute.
   - Add matching one-line pointers to `MEMORY.md`.

## Final step
Print a short summary: files created/updated, totals (routes/features/components/
stores/symbols documented), whether `CLAUDE.md` + memory were created or updated,
and any gaps flagged UNVERIFIED.
