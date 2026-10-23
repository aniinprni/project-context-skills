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
1. Read the **package baseline table** in the root `context/00-index.qmd` — each package's own `documented at` sha, plus the root sha for the root layer. If the table is absent (an older set that recorded only one root sha), treat every package as documented at that root sha, rebuild the table this run, and say so in the summary. If no baseline resolves at all, ask or fall back to FULL.
2. Changed set: `git diff --name-only <baseline>..HEAD` + `git status --porcelain`. If not git / unresolvable, ask which packages changed.
3. Empty changed set **for a package** (diffed from ITS OWN baseline, not the root's) → leave that package alone. Only when EVERY package and the root layer are clean do you report "already up to date at `<HEAD>`" and stop — UNLESS a package's per-package set is missing files this command now requires (symbol-reference / diagnosis-playbook / api-reference), predates the depth requirements, **or violates any "Non-negotiable output invariants" below** (stacked per-cycle sections under any wording, dated/baseline banners, an over-budget index, anchor-drift tables, numbered headings, package-relative anchors, stale anchors). That is a SCHEMA UPGRADE: (re)generate the missing/shallow files against current HEAD even without source changes, and NORMALIZE an offending set by collapsing every per-cycle section into the single current-state entry and deleting the history (at most a 10-line entry per past cycle in that layer's `CHANGELOG.qmd`).
4. Bucket each changed path by owning package (use the authoritative workspace list). Then:
   - **Root layer:** if `package.json`/workspace config/task-runner config or cross-package contracts changed → update the affected root files (`02-workspace-and-tooling`, `03-package-inventory`, `04-dependency-graph`, `05-shared-contracts`, `06-build-release-and-ops`). If a package was added/removed → always update `03-package-inventory` + `04-dependency-graph`.
   - **Per-package layer:** for each package with changes **since that package's own baseline**, run its per-package update using the BACKEND or FRONTEND change→file map (by package type) against that package's own `context/`, following that sibling command's incremental rules in full — including **MERGE IN PLACE, NEVER APPEND** (rewrite the existing entry to describe HEAD; never add a "changes at `<sha>`" section or a second entry for a symbol already documented; delete entries whose source is gone) and **RE-ANCHOR THE WHOLE FILE** (when a source file changes at all, re-verify the anchor of every symbol documented from it, not just the changed ones).
   - **A shared library changed → revisit its CONSUMERS.** Its `03-diagnosis-playbook.qmd` blast-radius list names them; a contract change there can invalidate a consumer's documented behavior even though the consumer's own files did not change. Also refresh root `05-shared-contracts.qmd` and `09-cross-package-diagnosis.qmd`.
5. Always: bump `date:` on rewritten files; set each regenerated package's OWN `Generated from commit` sha to HEAD, in its `00-index.qmd` and in its row of the root baseline table; and update the root `00-index.qmd` `date:` + baseline table.
   - 🔴 **NEVER advance the root "Generated from commit" sha past a package you did not regenerate.** A focused run (`$ARGUMENTS` named one package) or any run that skips packages leaves those packages documented at an older sha. If the root sha jumps to HEAD anyway, their un-documented changes fall BELOW the next run's diff window and become invisible permanently — the set then reports "already up to date" while silently lying about those packages.
   - The root sha advances to HEAD **only when every in-scope package was regenerated to HEAD**. Otherwise set the root sha to the **OLDEST** sha in the baseline table, so the next run's diff window still covers every lagging package, and list the lagging packages in the run summary.
6. Report baseline→HEAD, changed packages, root + per-package files updated, and what was left unchanged.
7. Touch memory only if the package/file SET changed or a recorded high-level fact is now wrong.

## Operating rules
- EXPLORE FIRST, WRITE SECOND. Never guess.
- **Anchors are WORKSPACE-ROOT-RELATIVE, always.** Cite
  `packages/api/src/index.ts:42`, never `src/index.ts:42`. Ten packages can each
  have a `src/index.ts`, so a package-relative anchor is ambiguous the moment it
  is read from anywhere else — including from the root layer, which cites across
  packages by definition. This holds in per-package files too.
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
- **Dispatch every per-package subagent with `model: "sonnet"`.** Per-package
  exploration is read-and-extract work; the cross-package judgment lives in the
  root layer you synthesize, which stays on the session model. Set effort
  explicitly on each — the subagent does not inherit the session's effort setting,
  so an unset effort is whatever the agent definition defaults to, not what you are
  running at:
  - **`effort: "high"`** for per-package exploration (the agent chooses what to
    read). Do not economize here — shallow exploration is the one failure this
    command exists to prevent, and in a monorepo it is multiplied by N packages.
  - **`effort: "medium"`** for the anchor re-verification pass on an incremental
    update: re-checking workspace-root-relative `file:line` against a known file
    list is lookup, not search, and it is the highest-volume work in the run.
- Be precise over verbose. Tables and diagrams over prose.

## Non-negotiable output invariants
These override everything else below, and they apply to the ROOT layer and to
EVERY per-package layer. A run that breaks one of these has failed, even if every
sentence it wrote is accurate. In a monorepo these matter more than anywhere else:
Layer B repeats per package, so an unchecked habit is multiplied by N.

1. **CURRENT STATE ONLY — a context set is not a changelog.** Every `.qmd`
   describes the code at HEAD, not the path it took to get there.
   - NEVER append a per-cycle section, under ANY wording. Not `## New & changed
     symbols this cycle`, not `## What changed at this baseline`, not `## Changes
     at <sha>`, not `## … N baselines ago`, not a dated "incremental update to
     `<sha>`" banner, not a `*(new at <sha>)*` suffix on a heading. Renaming the
     section does not make it allowed. Edit the existing entry instead.
   - **STRUCTURAL TEST — no heading may name a commit, a baseline or a cycle.**
     `grep -rnE '^#+.*([0-9a-f]{7,40}|baseline|cycle)' context/ --include=*.qmd`
     must come back empty, in the root set and in every package set. (The
     per-package baseline sha belongs in frontmatter or a table row, never in a
     heading.)
   - NEVER narrate deltas inside a durable entry. No commit shas, "new at",
     "previously", "reverted by" in the body of a reference or playbook file.
   - One symbol → exactly ONE description of it, true right now. Two entries is a
     defect, because the stale one is usually found first.
   - If a history trail is wanted it lives in ONE file per layer,
     `context/CHANGELOG.qmd`: newest first, one entry per run, max 10 lines, file
     capped at 200 lines. No other file carries history.
2. **"READ FIRST" appears exactly once per layer** — in that layer's
   `00-index.qmd`. If everything is marked read-first, nothing is.
3. **Size budgets — measured in BYTES as well as lines.** Line counts do not bound
   read cost: a 350-line file with 230-byte lines is an 80 KB read.
   - root `context/00-index.qmd` — **≤ 300 lines AND ≤ 25 KB**, TOC/routing inside
     the first 50 lines. It is read on EVERY question about any package.
   - each per-package `00-index.qmd` — **≤ 250 lines AND ≤ 20 KB**.
   - any other single `.qmd` — **≤ 1,500 lines AND ≤ 120 KB**; shard rather than
     overflow (shard by module, NEVER by date, and reduce the original to a
     ≤ 150-line router pointing at the shards).
   - **no single line over ~400 bytes** — a mega-line is unreadable AND hides from
     every line-based budget.
   - **per package**, the context set should stay well under the byte size of that
     package's source. Check the aggregate too: `du -sk` of all context folders vs
     the repo's source bytes. Context that rivals the code it replaces has failed
     at its one job.
4. **Anchors must be true at HEAD** — see the next section.
5. **Do not number section headings.** Write `## Contexts`, never `## 12. Contexts`.
   Numbering corrupts the moment a section is inserted or removed — observed in a
   real run: a playbook running §19 → §21 → … → §26 → §28 → §31 with
   `## 27. Where to start` sitting last.

## Anchor hygiene (`file:line` must not rot)
Cited anchors are where the execution step jumps, and the `CLAUDE.md` this command
writes tells future sessions to trust them WITHOUT opening source. A confidently
wrong anchor is worse than no anchor.

- Anchors are workspace-root-relative (see Operating rules) and point at the
  symbol's **declaration** line, with the symbol's own name in the same row so it
  stays greppable after drift.
- **Never carry an anchor forward unverified.** Before writing any `file:line`,
  confirm it against a fresh read of that file at HEAD. An anchor copied from the
  previous version of a context file is presumed stale.
- **Re-verify the described FACT, not only the line.** A moved symbol is a drifted
  anchor; a symbol that left the file is a wrong entry. Fix the claim, not just
  the number.
- When a file changes, re-anchor EVERY symbol documented from that file, not only
  the symbols that changed — inserting or deleting any line shifts every anchor
  below it.
- List symbols in source order within a module, so an out-of-order or duplicated
  anchor is visible on sight.
- **NEVER publish an anchor-drift table.** Do not write a section recording how far
  anchors moved, or telling the reader to apply a `+717` / `−12` delta to an older
  section's line numbers, or declaring an earlier section's anchors "void". That is
  knowingly shipping wrong anchors with homework attached. If you worked out the
  drift, you already know the correct line: write it into the entry and delete the
  old one. The drift table is the bug, not the fix.
- **Subagents are not exempt.** Layer B is produced by parallel per-package agents;
  their anchors are unverified until the verification gate checks them. Budget for
  that check — do not assume a subagent verified its own output.
- The verification gate below checks this mechanically. Run it.

## Layer A — Repo-wide context under root `context/`
1. `context/00-index.qmd` — Master index, **≤ 300 lines and ≤ 25 KB**: what the
   monorepo is, TOC of all root files AND links to each package's
   `context/00-index` (inside the first 50 lines), "read me first", a SHORT "how
   to regenerate" note, glossary, and the **PACKAGE BASELINE TABLE** — the
   coverage checklist, which is a table, not a yes/no list:

   | package | path | type | per-package context | documented at |
   |---|---|---|---|---|
   | `@acme/api` | `packages/api` | service | yes | `4c8bc88` |
   | `@acme/ui` | `packages/ui` | shared-library | yes | `129aa5b` |
   | `@acme/tools` | `tools/` | tooling | **NO — skipped, reason** | — |

   The **documented at** column is load-bearing: it is the per-package baseline
   the next incremental run diffs from, and it is the only thing that makes a
   stale package distinguishable from a fresh one. A package whose sha is behind
   the others is a package whose context is lying. No changelog, no "what changed"
   blocks, no per-baseline history in this file.
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
10. `context/09-cross-package-diagnosis.qmd` — The **cross-package diagnosis
    layer**, and the root layer's answer to "why is X failing". Per-package
    playbooks cannot cover this by construction: in a monorepo the failures that
    actually hurt live BETWEEN packages. Cover at least:
    - **Contract drift** — producer changed a shared type/DTO/event, consumers
      not rebuilt or not updated. Which contract, who produces, who consumes
      (name them), what the break looks like at runtime vs build time.
    - **Version skew** — two packages resolving different versions of the same
      dep; hoisting/peer-dep surprises; a shared lib built against one version
      and consumed under another.
    - **Build-order and cache failures** — a task-pipeline edge that is missing
      or wrong, stale task cache, "works clean / fails incremental", affected-
      detection missing a dependent.
    - **Circular dependencies** — where they exist, what they break, build order.
    - **Cross-package blast radius** — for each shared library, the list of
      consumers, so the cost of a change is visible before it is made.
    Same shape as a per-package playbook: symptom → suspect location
    (`workspace/relative/path.ts:line`) → broken assumption → fix approach, plus a
    single "symptom → suspect package(s)" table for the whole repo.

## Layer B — Per-package context under `<package-path>/context/`
For EACH in-scope package, create a per-package `context/` folder. Choose the
template by package type.

**READ THE SIBLING COMMAND FIRST.** Before writing a per-package set, read the
matching command file — `commands/deep-context-backend.md` or
`commands/deep-context-frontend.md` — and apply it IN FULL to that package: its
file list, its depth requirements, its invariants, its anchor hygiene, and its
verification gate. The summaries below name the files; they do not replace those
commands, and they are not the rules. A per-package set written from the summary
alone will miss every quality rule those files carry, which is how a monorepo ends
up with N copies of the same defect. If a sibling command is unavailable, apply
the invariants and anchor-hygiene sections above and say so in the summary.

- Backend/service package → apply the BACKEND deep-context structure (overview,
  setup, config, architecture, data-model, api-surface, services,
  integrations/async, cross-cutting, ops, testing, glossary, **symbol-reference,
  diagnosis-playbook**), per `commands/deep-context-backend.md`.
- Frontend/app package → apply the FRONTEND deep-context structure (overview,
  setup, config, architecture, routing, state, api/integration, design-system,
  styling, cross-cutting, build/perf, testing, glossary, **symbol-reference,
  diagnosis-playbook**), per `commands/deep-context-frontend.md`.
- Shared library / tooling package → a lighter set for STRUCTURE, but NOT a
  lighter diagnosis. A shared library is the package whose bugs have the widest
  blast radius in the repo, so it gets the fullest failure coverage, not the
  thinnest:
  `00-index.qmd` (purpose, public API surface, consumers, how to extend),
  `01-api-reference.qmd` (EVERY exported module/function/type with signature +
  workspace-relative `file:line` + behavior — the answer-completeness backbone),
  `02-usage-and-conventions.qmd` (patterns, do/don't, versioning + what counts as
  a breaking change for consumers), and
  `03-diagnosis-playbook.qmd` — a FULL playbook, same depth as a service's
  `15-diagnosis-playbook.qmd`: failure modes → broken assumption → origin
  `file:line` → fix approach, a symptom → suspect location table, and a
  **CONSUMER BLAST RADIUS** section naming every package that imports this one
  and which exports each consumer depends on, so the cost of changing an export
  is visible before it is changed. Open questions go in `00-index.qmd` under
  UNVERIFIED, not folded into the playbook.
Every per-package `00-index.qmd` must link BACK to the root
`../../context/00-index.qmd`, note this package's internal dependencies, and
record **its OWN baseline**: a `Generated from commit <sha>` line for the commit
THIS package was documented at. That sha must match the package's row in the root
baseline table. It is what the next incremental run diffs this package from — a
package is never diffed from the root sha, because the root sha may have advanced
during a run that did not touch this package.

**Tooling hygiene:** a per-package `context/` folder lives inside a buildable,
often publishable package. Check whether it would be swept into `tsconfig`
includes, test globs, the Docker build context, or npm packaging (`files` /
`.npmignore` — a shared library can otherwise publish its context to the
registry), and add the needed ignores. Report what you checked and changed.

## Depth requirements (answer-complete & diagnosis-complete)
The goal: a future session can ANSWER any question and DIAGNOSE any issue from
`context/` alone, opening source only to EXECUTE a change. That covers BOTH
scopes, and the second is the one monorepos get wrong: a question about one
package is answered from its own `context/`, and a question that spans packages —
contract drift, version skew, build order, "what breaks if I change this export?"
— is answered from the root layer's `09-cross-package-diagnosis.qmd` and
`05-shared-contracts.qmd`. Neither scope may require opening source.
Every per-package file must capture BEHAVIOR (control flow, invariants, error &
edge-case handling, data flow) with precise workspace-root-relative `file:line`
anchors — not just structure. The per-package `14-symbol-reference.qmd` / `01-api-reference.qmd`
must list every significant symbol. Completeness over brevity.

## Verification gate — RUN these checks, do not assert them
Before finishing (full OR incremental), actually execute the checks below and print
each result in the final summary. Do not claim completeness — demonstrate it. A
check you skipped is a check that failed. Run the per-layer checks for the ROOT
set and for EVERY package set you touched this run — a defect that appears once per
package is still N defects.

1. **Package coverage.** Every package from the authoritative workspace list
   appears in `context/03-package-inventory.qmd` AND has a row in the root
   baseline table; every in-scope package has a per-package `context/` folder.
   Any package intentionally skipped is listed with a reason. Diff the two lists
   mechanically — do not eyeball:
   ```bash
   # authoritative list (adapt to the workspace tool actually detected)
   pnpm -r list --depth -1 --json 2>/dev/null | grep -oE '"name": *"[^"]+"' | sed -E 's/.*: *"//;s/"//' | sort -u > /tmp/pkgs.txt
   grep -oE '^\| `[^`]+`' context/00-index.qmd | tr -d '|` ' | sort -u > /tmp/documented.txt
   comm -13 /tmp/documented.txt /tmp/pkgs.txt   # package exists, undocumented → MUST be empty
   comm -23 /tmp/documented.txt /tmp/pkgs.txt   # documented, package gone → remove the row
   ```
2. **Baseline consistency.** Every package's own `Generated from commit` sha
   equals its row in the root baseline table, and the root sha is not newer than
   the OLDEST package sha. Print the table and the root sha side by side; a root
   sha ahead of any package sha is the orphaning bug — fix it before finishing.
3. **Anchor check.** For every `file:line` cited in a file written this run, read
   that line and confirm the symbol named alongside it is declared there (±3 lines
   tolerance). Fix every miss. Confirm anchors are workspace-root-relative, not
   package-relative. Report `anchors checked: N, corrected: M`.
   ```bash
   grep -rohE '[A-Za-z0-9_./-]+\.(ts|tsx|js|jsx|py|go|rs):[0-9]+' \
     context/ */context/ */*/context/ 2>/dev/null | sort -u > /tmp/anchors.txt
   # each must resolve from the WORKSPACE ROOT:
   cut -d: -f1 /tmp/anchors.txt | sort -u | while read f; do [ -f "$f" ] || echo "UNRESOLVABLE: $f"; done
   ```
4. **Symbol coverage diff.** Per package, diff source exports against documented
   symbols, both directions, per the sibling command's gate. Both must be empty.
5. **Invariant check.** Run against the root set and each package set:
   ```bash
   wc -l context/*.qmd context/*/*.qmd 2>/dev/null         # vs the budgets above
   grep -rc 'READ FIRST' context/ | grep -v ':0$'          # exactly ONE hit per layer
   # no heading names a commit / baseline / cycle — phrasing-proof, MUST be empty:
   grep -rnE '^#+.*([0-9a-f]{7,40}|baseline|cycle|🆕)' context/ --include=*.qmd \
     | grep -v CHANGELOG
   grep -rnE '^#+ *[0-9]+[.)] ' context/ --include=*.qmd    # numbered headings: MUST be empty
   awk 'length>400 {print FILENAME": line "FNR" ("length" bytes)"}' \
     context/*.qmd context/*/*.qmd 2>/dev/null              # mega-lines: MUST be empty
   wc -c context/00-index.qmd                               # vs the byte budget
   du -sk context/ */context/ */*/context/ 2>/dev/null      # aggregate vs source bytes
   ```
6. **Tooling hygiene.** Confirm no per-package `context/` folder is swept into
   build/test/publish globs (tsconfig includes, test patterns, Docker context,
   npm `files`/`.npmignore`). Report what you checked and any ignores added.

If a check fails and you cannot fix it this run, say so explicitly and flag it
UNVERIFIED in the summary — never silently.

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
   - For a question that SPANS packages — a contract change, a version-skew or
     build-order failure, "who consumes this?", "what breaks if I change this
     export?" — read root `09-cross-package-diagnosis.qmd` and
     `05-shared-contracts.qmd`, plus the shared library's blast-radius list. Do
     not answer a cross-package question from one package's set alone.
   - If a per-package file is sharded, read its router, then ONLY the one shard
     that covers the question — never the whole reference.
   - Check the root baseline table before trusting a package's context: if that
     package's `documented at` sha is behind the others, its context may be stale
     — say so rather than answering confidently from it.
   - Anchors are workspace-root-relative and accurate as of each package's own
     baseline. When executing a change, locate the symbol BY NAME in that file
     rather than trusting the line number blindly.
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
root + per-package files created/updated, **the baseline table (package → sha) with
any lagging packages called out explicitly, and the root sha with the rule that set
it**, the result of EVERY verification-gate check (package-coverage diff, baseline
consistency, anchors checked/corrected + unresolvable anchors, symbol-coverage
diffs, invariant/budget check, tooling hygiene), context bytes vs source bytes in
aggregate, whether `CLAUDE.md` + memory were created or updated, and any gaps
flagged UNVERIFIED.
