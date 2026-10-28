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
3. Empty changed set → report "already up to date at `<HEAD>`", stop — UNLESS the existing set is missing files this command now requires (`14-symbol-reference.qmd`, `15-diagnosis-playbook.qmd`) or predates the depth requirements. That is a SCHEMA UPGRADE: (re)generate the missing/shallow files against current HEAD even without source changes. **Also a SCHEMA UPGRADE: an existing set that violates any "Non-negotiable output invariants" below** (stacked per-cycle sections under any wording, dated/baseline banners, an over-budget `00-index.qmd`, anchor-drift tables, numbered headings, stale anchors) — NORMALIZE it by collapsing every per-cycle section into the single current-state entry for that symbol/subsystem and deleting the history (at most a 10-line entry per past cycle in `context/CHANGELOG.qmd`).
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
   - **ANY changed source file → ALSO refresh the affected sections of the symbol
     reference (file 14 / its shards) and the diagnosis playbook (file 15 / its
     shards).** These two deep files must never lag the source they describe.
   - **MERGE IN PLACE, NEVER APPEND.** Rewrite the entry that is already there so
     it describes HEAD. Do not add a "what changed at this baseline" section, a
     "changes at `<sha>`" section, a dated banner, or a second entry for a symbol
     you already documented. One symbol → exactly one entry, always current.
     Delete entries whose source is gone.
   - **RE-ANCHOR THE WHOLE FILE.** When a source file changes at all, re-verify the
     `file:line` anchor of EVERY symbol documented from that file — not just the
     symbols that changed. Inserting or deleting any line shifts every anchor below
     it. Skipping this is the single most common way this context set rots, and a
     3,000-line page component drifts by hundreds of lines in one sprint.
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
- **Dispatch every fan-out subagent with `model: "sonnet"`.** The feature-area
  passes are read-and-extract work; the judgment lives in your synthesis, which
  stays on the session model. Set effort explicitly on each — the subagent does not
  inherit the session's effort setting, so an unset effort is whatever the agent
  definition defaults to, not what you are running at:
  - **`effort: "high"`** for feature-area exploration (the agent chooses what to
    read). Do not economize here — shallow exploration is the one failure this
    command exists to prevent, and it is invisible in the output until someone
    relies on it.
  - **`effort: "medium"`** for the symbol-reference anchor re-verification pass on
    an incremental update: re-checking `file:line` against a known file list is
    lookup, not search, and it is the highest-volume work in the run.
- Prefer dedicated search/read tools over shell cat/grep where possible.

## Non-negotiable output invariants
These override everything else below. A run that breaks one of these has failed,
even if every sentence it wrote is accurate.

1. **CURRENT STATE ONLY — the context set is not a changelog.** Every `.qmd`
   describes the app as it is at HEAD, not the path it took to get there.
   - NEVER append a per-cycle section, under ANY wording. Not `## New & changed
     symbols this cycle`, not `## What changed at this baseline`, not `## Changes
     at <sha>`, not `## … N baselines ago`, not a dated "incremental update to
     `<sha>`" banner, not a `*(new at <sha>)*` suffix on a heading. Renaming the
     section does not make it allowed. Edit the existing entry instead.
   - **STRUCTURAL TEST — no heading may name a commit, a baseline or a cycle.**
     `grep -rnE '^#+.*([0-9a-f]{7,40}|baseline|cycle)' context/ --include=*.qmd`
     must come back empty. That is the phrasing-proof form of the rule above.
   - NEVER narrate deltas inside a durable entry. Write "renders `<DataSourceCard>`
     per item" — not "🆕 `cdb7436`: now renders X (was Y)". Commit shas, "new at",
     "previously", "reverted by" appear nowhere in files 01–15.
   - If a reader greps a component/hook name they must find exactly ONE
     description of it, and it must be true right now. Two entries for one symbol
     is a defect, because the stale one is usually found first.
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
   - the whole `context/` set should stay well under the byte size of the source
     it documents (`find src -type f \( -name '*.ts*' -o -name '*.js*' \) -exec cat {} + | wc -c`
     vs `cat context/*.qmd context/*/*.qmd | wc -c`). If it approaches that, the
     set is narrating history or repeating itself — compress, do not rationalize.
4. **Anchors must be true at HEAD** — see the next section.
5. **Do not number section headings.** Write `## Contexts` / `## Custom hooks`,
   never `## 12. Contexts`. Numbering corrupts the moment a section is inserted or
   removed — observed result in a real run: a playbook running §19 → §21 → … →
   §26 → §28 → §31 with `## 27. Where to start` sitting last, after §31. Order
   sections logically and let the heading text carry the meaning.

## Anchor hygiene (`file:line` must not rot)
Cited anchors are where the execution step jumps, and the `CLAUDE.md` this command
writes tells future sessions to trust them WITHOUT opening source. A confidently
wrong anchor is worse than no anchor — and in a frontend repo, where a single page
component can be 3,000+ lines and gets edited every sprint, anchors rot fastest.

- Anchor format is `path/to/file.tsx:LINE` pointing at the symbol's **declaration**
  line, and the symbol's own name must appear in the same row so it stays
  greppable after drift.
- **Never carry an anchor forward unverified.** Before writing any `file:line`,
  confirm it against a fresh read of that file at HEAD. An anchor copied from the
  previous version of a context file is presumed stale.
- **Re-verify the described FACT, not only the line.** A moved symbol is a drifted
  anchor; a symbol that left the file is a wrong entry. If an entry says a provider
  chain lives in `layout/chat-layout.tsx` and that provider is no longer imported
  there, fix the claim, not just the number.
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

1. `context/00-index.qmd` — Master index, **≤ 300 lines and ≤ 25 KB**:
   one-paragraph app description, TOC linking every other file with a one-line
   hook (inside the first 50 lines), "read me first", a SHORT "how to regenerate"
   note (baseline commit line + the command — not a log), glossary, and a COVERAGE
   CHECKLIST mapping every top-level source directory to the file(s) documenting
   it. It also records headline counts (routes/components/hooks/…), which the
   verification gate re-derives from source every run. This file is read on every
   single question: no changelog, no "what changed" blocks, no per-baseline
   history.
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
    (route → page → components → hooks → api). Terse rows over prose; large is
    fine — but **if it would exceed 1,500 lines or 120 KB, shard it**: move the
    per-module sections into `context/14-symbols/<module>.qmd` (e.g. `routes.qmd`,
    `components-layout.qmd`, `components-elements.qmd`, `hooks.qmd`, `context.qmd`,
    `lib.qmd`) and keep `14-symbol-reference.qmd` as a **≤ 150-line router**: one
    line per shard saying which modules/symbols live in it, so a session loads one
    shard instead of the whole reference. Shard by module, NEVER by date.
16. `context/15-diagnosis-playbook.qmd` — The **diagnosis-completeness layer**.
    Per subsystem: likely failure modes (render errors, stale state, failed
    requests, hydration/SSR issues), the broken assumption, the origin
    (`file:line`), blast radius, and the fix approach — from context alone. Add a
    "symptom → suspect location(s)" table and note foot-guns / effect-ordering /
    race conditions. There is **ONE** symptom table for the whole app, kept
    current — never one per baseline. If the file would exceed 1,500 lines or
    120 KB, shard it the same way as file 14 into
    `context/15-diagnosis/<subsystem>.qmd`, with `15-diagnosis-playbook.qmd`
    reduced to a ≤ 150-line router plus the single global symptom table.

## Depth requirements (answer-complete & diagnosis-complete)
The goal: a future session can ANSWER any question and DIAGNOSE any issue from
`context/` alone, opening source only to EXECUTE a change. Beyond structure,
every file must capture BEHAVIOR: control flow & conditional rendering,
state/data flow across components and to the backend, invariants/assumptions,
error & edge-case handling (loading/empty/error states, retries, guards), and
precise `file:line` anchors on every claim. Completeness over brevity — better a
long context file than needing to read source to answer or troubleshoot.

## Verification gate — RUN these checks, do not assert them
Before finishing (full OR incremental), actually execute the checks below and print
each result in the final summary. Do not claim completeness — demonstrate it. A
check you skipped is a check that failed.

1. **Anchor check.** For every `file:line` you cite in a file you wrote this run,
   read that line and confirm the symbol named alongside it is declared there
   (±3 lines tolerance for a decorator/JSX wrapper). Fix every miss. Report
   `anchors checked: N, corrected: M`.
   ```bash
   grep -ohE '[A-Za-z0-9_./-]+\.(tsx?|jsx?):[0-9]+' context/*.qmd context/*/*.qmd \
     | sort -u > /tmp/anchors.txt
   # for each path:line — sed -n "${line}p" "$path" — and compare to the entry
   ```
2. **Count check.** Re-derive every count stated in `00-index.qmd` from source:
   ```bash
   grep -rhoE '<Route\b|path: *["\x27]' src/routes src/ 2>/dev/null | wc -l  # routes
   find src/components -name '*.tsx' | wc -l                                  # components
   find src/hooks src/service-hooks -name '*.ts*' 2>/dev/null | wc -l         # hooks
   ```
3. **Symbol coverage diff.** Per module, diff source exports against documented
   symbols — do not eyeball this:
   ```bash
   grep -rhoE '^export (default )?(function|const) [A-Za-z0-9_]+' src/components/layout \
     | sed -E 's/.* //' | sort -u > /tmp/src.txt
   grep -ohE '`[A-Za-z0-9_]+`' context/14-symbols/layout.qmd \
     | tr -d '`' | sort -u > /tmp/doc.txt
   comm -13 /tmp/doc.txt /tmp/src.txt   # in source, undocumented  → MUST be empty
   comm -23 /tmp/doc.txt /tmp/src.txt   # documented, gone from source → delete these
   ```
   Repeat per module (components, hooks, context, lib, routes, service-hooks).
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
   # CLAUDE.md "Hard rules" section: no anchors / dates / shas — MUST be empty:
   sed -n '/^### Hard rules/,/^##/p' CLAUDE.md \
     | grep -nE '\.(tsx?|jsx?|css|json|sh):[0-9]+|[0-9]{4}-[0-9]{2}-[0-9]{2}|\b[0-9a-f]{7,40}\b'
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
   - Route component/hook/behavior questions → `context/14-symbol-reference.qmd`;
     bug/error/"why is X failing"/fix questions → `context/15-diagnosis-playbook.qmd`;
     routes → `05-routing-and-navigation.qmd`; state → `06-...`; api → `07-...`.
     If 14/15 are sharded, read the router file, then ONLY the one shard that
     covers the question — never the whole reference.
   - Anchors (`file:line`) are accurate as of the recorded baseline commit. When
     executing a change, locate the symbol BY NAME in that file rather than
     trusting the line number blindly — it may have drifted since.
   - Do NOT open source for Q&A or diagnosis. Read/edit source ONLY when the user
     explicitly asks to implement/execute a change. If context is insufficient or
     marked UNVERIFIED, say so and ask before falling back to source.
   - Reading `context/` is pre-approved — never pause to confirm.
   - Add a short **`### Hard rules (no lookup needed)`** section: at most ~10
     one-line bullets a session must know BEFORE it would think to consult
     `context/` — rules that cause silent damage when broken. Typical picks: build
     strictness / the only working verification gate, PR title or branch rules CI
     enforces, theming traps (e.g. dead `dark:` utilities, remapped colours),
     global layout traps (non-scrolling `body`), the required API wrapper, names
     that must match across repos, banned comment/code patterns. Rules:
     - **No `file:line` anchors, no dates, no commit shas** in this section — it is
       outside the verification gate, so anything that can drift will rot here.
       Name the symbol or file; the anchor lives in `context/`.
     - **Each bullet must be backed by a `context/` entry** that states the same
       fact; re-verify it every run (full OR incremental) and delete bullets that
       are no longer true at HEAD. Do not duplicate content already in another
       CLAUDE.md section.
     - Rules only — no explanations, recipes, or subsystem walkthroughs. Those
       belong in `context/`; a CLAUDE.md that reads as "enough for most tasks"
       defeats the context-first rule above.
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
stores/symbols documented, each re-derived from source by the verification gate),
the result of EVERY verification-gate check (anchors checked/corrected, count
mismatches fixed, coverage diffs empty or not, invariant/budget check, context
bytes vs source bytes), whether `CLAUDE.md` + memory were created or updated, and
any gaps flagged UNVERIFIED.
