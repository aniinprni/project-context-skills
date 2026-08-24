---
description: Audit a code-graph parser against source AST ground truth and report which code constructs it fails to represent
argument-hint: "[optional: --api <host> --project <uuid> --services <a,b,c> --hypotheses <file>]"
---

# Code Graph Parser Gap Analysis

You are auditing a code-graph parser by diffing the graph it produced against
ground truth extracted from the source it was built from. The goal is NOT to
score the graph — it is to enumerate, with evidence, WHICH CODE CONSTRUCTS THE
PARSER FAILS TO REPRESENT, so the parser can be improved.

Optional arguments from the user: $ARGUMENTS

## Step -1 — Resolve inputs before doing anything else
Resolve, in this order of precedence: explicit `$ARGUMENTS` → repo config file
(e.g. `.breeze.json`, `.cog/`, or equivalent graph-tool config at the repo root)
→ ask the user. You need:
- **Source repo path** — defaults to the current working directory.
- **Graph API host** and **project / graph identifier**.
- **Graph access method** — MCP tools if a graph MCP server is connected
  (prefer it), otherwise the REST API.

A config file's default identifier is NOT automatically the right one: a repo may
be indexed into several graphs. State which graph you selected and why. If you
cannot resolve host + identifier, stop and ask — do not guess.

## Step 0 — Pin the comparison
Determine the exact commit the graph was indexed at, and check the working tree
out to (or diff against) that commit. Report the drift if the tree has moved.
Any gap you report must be attributable to the parser, not to commits landed
after indexing. If the indexed commit is unrecoverable, say so and treat every
finding as provisional.

## Step 1 — Scope: 3 to 4 representative units, not the whole repo
Depth of evidence matters more than breadth. Detect the stack first, then select
3–4 packages/services/apps that between them exercise the distinct parsing
problems present in this repo. For a typical TypeScript service+UI codebase that
means one unit per axis:
1. a **DI- and decorator-heavy backend** unit (modules, providers, guards,
   interceptors, custom tokens)
2. a unit with a substantial **ORM footprint** (entities, relations, migrations,
   query builder / raw SQL)
3. an **API-schema-exposing** unit (GraphQL resolvers and field decorators, SDL
   files, generated types — or OpenAPI/tRPC/protobuf equivalents)
4. the **UI app** (component edges, hooks, routing, i18n keys, data-fetching keys)

Adapt these axes to whatever stack you actually find. One unit may satisfy two
axes — say so and pick a fourth to cover what is left. Confirm each selected path
exists before committing to it; never assume package names.

Record in Methodology: the selection, the rationale, the packages deliberately
excluded, and which constructs are consequently unexamined. **Every count in the
report must be scoped to these units — never present a sampled figure as a
repo-wide total.**

## Step 2 — Build independent ground truth (AST, not regex)
Do NOT use grep/regex as your primary census. It systematically miscounts typed
languages (enums, overload signatures, type-only syntax, decorators).

**For TypeScript/JavaScript, use ts-morph loaded against the real workspace
tsconfig(s), so module resolution and the type checker are live.** This is a hard
requirement, not a preference: Step 6 audits whether the parser RESOLVES symbols,
and a syntax-only parser cannot produce ground truth for resolution — it can
confirm a construct exists but not what it points at. Use tree-sitter only as a
fallback for files the tsconfig does not cover (loose scripts, config files), and
label any counts sourced that way as syntax-only. For other languages use the
equivalent resolution-capable frontend (e.g. `jedi`/`libcst` + mypy for Python,
`go/packages` for Go, JDT for Java) and state which you used.

Write a throwaway extractor to a scratch directory (never into the audited repo)
emitting a JSON census over the selected units:
- files, classes, interfaces, type aliases, enums
- functions by kind (declaration / expression / arrow / method / getter / setter)
- decorators and their arguments
- imports and exports by kind
- extends / implements
- call sites, with resolved target where available
- component/template usages (JSX or equivalent)
- **resolution outcome per reference**: resolved to file+symbol / unresolved / ambient

Report the extractor's own blind spots before using its numbers.

## Step 3 — Inventory the graph
Enumerate every node label and edge type in the graph with counts, scoped to the
selected units. For each, identify the source construct it is supposed to
represent, and split the delta into **UNDERCOUNT** (source constructs with no
node) and **OVERCOUNT** (nodes with no supporting source construct). Report both
columns separately — never a net figure.

## Step 4 — Diff and classify
Produce four gap classes, never merged:
- **A. MISSING NODES** — in the AST census, absent from the graph
- **B. MISSING EDGES** — both endpoints exist as nodes, relationship absent
- **C. WRONG ATTRIBUTES** — node present, properties empty / default / false / incorrect
- **D. SPURIOUS or DUPLICATED** — graph asserts what source does not support

## Step 5 — Construct-level coverage matrix (primary deliverable)
For EACH construct below: `parsed / partial / missed`, occurrence count within the
selected units, one concrete `file:line` example, the graph query proving presence
or absence, and the downstream consequence. Mark anything you could not verify as
**UNVERIFIED** — do not guess. Drop rows for constructs the language does not have,
and add rows for language features not listed here.

**Declarations** — class, abstract class, interface, type alias, enum, const enum,
function declaration, function expression, arrow function, async/generator
function, method, getter/setter, static member, class property initializer,
constructor parameter properties, namespace/module declaration, declaration
merging, ambient declaration files, overload signatures.

**Imports / exports** — named, default, `export *`, `export * as ns`, barrel
re-export chains, type-only imports, dynamic `import()`, `require()`,
side-effect-only imports, tsconfig/workspace path aliases, circular imports.

**Calls and references** — direct call, method call, optional-chaining call
(`a?.b()`), computed member call (`a[k]()`), call through a destructured alias,
callback / higher-order passing, `new`, `super()`, decorator invocation, tagged
template, IIFE, spread arguments.

**Type relationships** — extends, implements, mixin patterns
(`class X extends Mixin(Base)`), generic type arguments and constraints.

**Control and data flow** — conditionals, loops, try/catch, throw sites, await
points, promise chains, event emitters and subscribers. State clearly whether the
parser models these at all rather than listing them as missed.

**Framework idioms** — DI container wiring (module imports/providers/exports
arrays, injectable classes, custom provider tokens including string and Symbol
tokens); HTTP route declarations, guards, interceptors, pipes; API schema
(resolvers, query/mutation/field decorators, SDL files, generated types); ORM
entities, column and relation decorators, repositories, query builder and raw SQL,
migrations; search/index client usage; UI components (function components,
memo/forwardRef wrappers), custom hooks, element→component edges, route
definitions, i18n key usage, data-fetching query keys; monorepo project config
targets, tags, implicit dependencies; manifest workspace dependency edges; env
var reads.

Also state explicitly: **are test files indexed, and do test→subject edges exist?**

## Step 6 — Resolution-capability audit (separate from syntax coverage)
Syntax coverage and symbol resolution are different parser problems, so analyze
them separately. For each indirect construct below, determine whether the parser
resolves it to a real target, resolves it to a placeholder, or drops it — and
quantify occurrences:
- barrel re-export chains (`export *` through index files, multi-hop)
- path aliases resolving to the correct file
- dynamic `import()` / `require()` with a static string vs. a computed argument
- string and Symbol DI tokens resolved to their provider
- computed member access `a[k]()` and destructured-alias calls
- mixin patterns and generic type arguments
- re-exported types crossing package boundaries

For each: does the miss produce a **missing** edge, or an edge to the **wrong**
node? Wrong-target edges are more damaging than missing ones — flag them
separately and rank them above missing edges in the backlog.

## Step 7 — Test prior hypotheses
If the user supplied a hypotheses file or named known-suspect findings from an
earlier audit, confirm or refute EACH with evidence. Treat them as hypotheses,
never as established facts — a previously-real gap may have been fixed, and a
previously-clean area may have regressed.

Regardless of what was supplied, explicitly check these commonly-suspect areas
and report the result even when clean:
- import / dependency edge count is zero or implausibly low
- boolean or enum node attributes uniformly populated with a default value
- schema-derived fields double-counted (hand-written source plus generated types)
- generated, vendored, or build-output directories indexed as first-class source
- files present as nodes but with zero contained symbols

## Step 8 — Downstream impact
For each confirmed gap, name the concrete question it breaks: single-file impact
analysis, cross-service blast radius, module boundary enforcement (check the repo
for documented architecture rules or lint boundary configs and cite them),
dead-code detection, test coverage tracing, ownership routing. A gap with no
named downstream consequence gets deprioritized, not deleted.

## Step 9 — Remediation backlog
Rank by (severity × expected node/edge yield) ÷ implementation effort. For each:
the proposed parser rule, the resolution capability it requires (symbol
resolution, type checker, module resolution, decorator evaluation), and the
estimated yield in nodes/edges.

## Output
Write a Quarto report at the repo root (or alongside existing reports if the repo
has a convention for them):

`code-graph-parser-gap-report.qmd` → render to `.html` and `.pdf`

Sections in this exact order:
1. **Executive Summary**
2. **Methodology and Scope**
3. **Coverage Baseline**
4. **Gap Taxonomy (A–D)**
5. **Construct Coverage Matrix**
6. **Framework-Idiom Gaps**
7. **Indirect and Dynamic Constructs**
8. **Downstream Impact**
9. **Remediation Backlog**
10. **Appendices** — reproducible query/AST pairs, false-negative catalog,
    false-positive catalog, raw counts

## Operating rules
- Every gap needs the **evidence triple**: source snippet with `file:line`, the
  exact graph query run, and the observed result. No triple, no row.
- **Quantify.** "Imports are missing" is useless; "0 of 1,207 import statements
  across the 3 audited services produced an edge" is actionable.
- Report absence of evidence as **UNVERIFIED**, distinctly from confirmed absence.
  These are different findings and must never be merged into one number.
- If a sampling or top-N cap limits coverage, say so explicitly and give the count
  of what went unexamined. Silent truncation reads as full coverage.
- Do not propose parser fixes for gaps you have not confirmed against source.
- Maintain two catalogs while you work and emit both as appendices:
  **FALSE NEGATIVES** (source construct confirmed present, graph has nothing) and
  **FALSE POSITIVES** (graph asserts a node/edge/attribute source does not
  support). Each entry: construct, `file:line`, graph query, observed result.
- Close the Executive Summary with a **trustworthiness verdict per question type**
  — single-file impact analysis, cross-service blast radius, boundary enforcement,
  dead-code detection — each rated `trustworthy / partial / unusable`, with the
  specific gap driving the rating.
- Never write scratch extractors, census JSON, or intermediate output into the
  audited repo.

## Final step
Print a short summary: graph and commit audited, units selected vs. excluded,
total gaps by class (A/B/C/D), the top 3 by downstream impact, how many findings
are UNVERIFIED, and the report paths written.
