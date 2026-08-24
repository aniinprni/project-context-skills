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

## Execution plan — read this before starting

These steps are a dependency graph, NOT a strict sequence. Steps at the same level
have no dependency on each other and MUST be dispatched together in one batch, not
run one after the other. Steps marked PARALLEL fan out to subagents internally.

```
Step -1 -> Step 0 -> Step 0.5 -> Step 1  strictly sequential
                                         (inputs, commit, language census, scope)
     |
     +-- Step 2  AST ground truth        PARALLEL: one agent per selected unit
     +-- Step 3  Graph inventory         PARALLEL: one agent per label/edge-type group
     |           (2 and 3 are independent - dispatch them together)
     v
   BARRIER (Step 4 needs both censuses complete)
     |
   Step 4  Diff and classify             single agent, whole-corpus reasoning
     |
     +-- Step 5  Construct matrix        PARALLEL: one agent per construct family
     +-- Step 6  Resolution audit        PARALLEL: one agent per indirect-construct class
     +-- Step 7  Hypothesis tests        PARALLEL: one agent per hypothesis
     |           (5, 6 and 7 all read the Step 4 output - dispatch them together)
     v
   BARRIER (Steps 8 and 9 rank across ALL findings)
     |
   Step 8 -> Step 9 -> Output            sequential synthesis, single agent
```

Steps 4, 8 and 9 are deliberately NOT parallel: they require reasoning over the
complete finding set, and splitting them produces inconsistent classification and
incomparable severity rankings.

### Subagent contract
Every fan-out subagent gets: the pinned commit, its assigned slice, the path to the
**shared** census JSON from Step 2, and the graph coordinates from Step -1.

- Each returns structured findings, one evidence triple per finding: source snippet
  with `file:line`, the exact graph query run, the observed result.
- **Reject any finding that arrives without a complete triple.** Do not launder an
  unsupported claim into the report by rewording it — send it back or drop it and
  note the drop.
- Subagents report counts **read from the shared census**; they must never re-derive
  counts independently, or the report's numbers will not reconcile.
- Independently re-verify a sample of at least two findings per subagent, plus
  **every** finding that drives an `unusable` trustworthiness verdict. A verdict that
  severe is worth confirming yourself before publishing it.
- Cap concurrent graph-querying agents at 4-6. The graph API is a shared service and
  an unthrottled fan-out will rate-limit, producing empty results that look exactly
  like genuine coverage gaps. **A failed or empty query is never evidence of a gap** -
  retry it, and if it still fails, mark the row UNVERIFIED.

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

## Step 0.5 — Language census and coverage declaration

Do this BEFORE scoping, because scoping is where languages get silently lost.

Count every language present in the repo by tracked file count — not just the one
you expect to audit. Use the version-control file list rather than a directory walk,
so vendored and ignored trees do not inflate the numbers:

```
git ls-files | sed -n 's/.*\.\([A-Za-z0-9_]\+\)$/\1/p' | sort | uniq -c | sort -rn
```

For each language above a trivial threshold, establish two facts with one query each:
1. Does the graph contain ANY node sourced from a file of that type?
2. If yes, is it a first-class representation (symbols, relationships) or only a file
   stub with no contained structure?

Then, for every language, make an explicit disposition — **scoped in** or **declared
out** — and record it as a table in Methodology AND as rows in the Coverage Baseline:

| Language | Files | Indexed? | Representation | Disposition |
|---|---:|---|---|---|

**A language may never be omitted silently.** Declaring a language out of scope is
acceptable and often correct; omitting it without saying so is not, because the
resulting report looks complete while a whole layer of the system is missing from it.

An **unindexed language is itself a finding, and frequently a larger one than any
missing construct within an indexed language** — a repo whose infrastructure-as-code,
schema, or pipeline definitions are absent from the graph cannot answer blast-radius
questions that cross into those layers, no matter how well its application code is
parsed. Report such a language as a single high-severity gap with its file count and
the questions it makes unanswerable. Do NOT spend the full evidence machinery of Steps
5 and 6 on it: once "not indexed at all" is established, per-construct auditing adds
nothing. Escalate to a detailed audit only if the language IS indexed and you need to
know how well.

Watch specifically for languages that live in their own top-level tree rather than
inside the application packages — infrastructure (Terraform/HCL, Bicep, CloudFormation),
pipeline and deployment definitions (CI YAML, Kubernetes manifests, Helm), and
architecture or diagram DSLs. These are the ones the Step 1 selection axes below will
never reach, because those axes are shaped around application code. Languages that live
*inside* the application packages — SQL migrations, schema definition files, generated
clients — travel along with whichever unit you select and need no special handling.

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

The axes above are shaped around application code, so they select application
packages. Reconcile the selection against the Step 0.5 census before continuing: any
language whose files sit outside every selected unit is carried by its Step 0.5
disposition, not by this step, and must already be marked scoped-in or declared-out
there. If a scoped-in language has no unit covering it, add a fifth unit for it or
change its disposition — do not let it fall between the two steps.

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
label any counts sourced that way as syntax-only. For other general-purpose languages use the
equivalent resolution-capable frontend (e.g. `jedi`/`libcst` + mypy for Python,
`go/packages` for Go, JDT for Java) and state which you used.

**Declarative and configuration languages have no equivalent, and you should not
pretend otherwise.** For anything scoped in from Step 0.5 that is not a
general-purpose language, use the native tool that resolves its own references and say
plainly that the resulting ground truth is coarser than the TS/JS census:
- Terraform / HCL — `terraform graph`, or `hcl2json` plus module/resource reference
  extraction. Ground truth is module, resource, variable and output wiring, NOT
  statements. Report coverage at that granularity and do not fabricate
  statement-level rows for it.
- SQL — parse DDL for tables, columns, constraints and indexes; treat migration
  ordering as a relationship. Note that dynamic or application-constructed SQL is
  out of reach of static parsing at all, and say so rather than counting it as covered.
- CI / Kubernetes / Helm YAML — resolve job, step and manifest references structurally.
- Schema and diagram DSLs (GraphQL SDL, Prisma, likec4) — use their own parsers where
  one exists; otherwise treat them as unindexed and report them as such.

When a language's ground truth is coarser, mark its Coverage Baseline rows with the
granularity used, so a reader never compares an HCL resource count against a TS
statement count as though they measured the same thing.

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

**Parallelism.** Write ONE extractor and share it — do not let each agent author its
own, or the per-unit numbers will not be comparable. Then fan out **one agent per
selected unit** to run it over that unit, reconcile its output against a hand-checked
sample of files, and report unit-specific blind spots (unresolved tsconfig paths,
excluded globs, generated directories). Merge the per-unit JSON into a single census
keyed by unit before Step 4 — every later step reads this one file.

Dispatch this step in the SAME batch as Step 3; they do not depend on each other.

## Step 3 — Inventory the graph
Enumerate every node label and edge type in the graph with counts, scoped to the
selected units. For each, identify the source construct it is supposed to
represent, and split the delta into **UNDERCOUNT** (source constructs with no
node) and **OVERCOUNT** (nodes with no supporting source construct). Report both
columns separately — never a net figure.

**Parallelism.** Enumerate the label and edge-type list first (cheap, single call),
then fan out **one agent per related group** — e.g. file/module nodes, symbol
declaration nodes, relationship edges, framework-specific labels — each pulling its
own counts and a sample of node payloads. Grouping by relatedness rather than by
arbitrary chunk keeps each agent's reasoning coherent when it decides which source
construct a label is meant to represent.

Dispatch this step in the SAME batch as Step 2.

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

**Parallelism.** This is the largest step by data volume — roughly 60+ constructs
across 3–4 units, each needing its own evidence triple. Fan out **one agent per
construct family** (Declarations / Imports-exports / Calls-and-references / Type
relationships / Control-and-data-flow / Framework idioms), each covering all selected
units so it can spot per-unit divergence — a construct parsed correctly in one service
and missed in another is itself a finding, and splitting by unit instead of by family
would hide it. Framework idioms is the heaviest family; split it further by stack
concern (DI, routing, API schema, ORM, UI, build/manifest) if one agent's slice is too
large to cover at evidence depth.

Dispatch this step in the SAME batch as Steps 6 and 7.

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

**Parallelism.** Fan out **one agent per bullet above**. Each must trace concrete
instances end to end — from the reference site, through the resolution chain, to what
the graph actually points at — rather than reasoning about the construct in general.
Chasing a multi-hop barrel chain is slow serial work, which is exactly why these run
concurrently.

Dispatch this step in the SAME batch as Steps 5 and 7.

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

**Parallelism.** Fan out **one agent per hypothesis**, each instructed to try to
REFUTE it rather than confirm it. Independent refutation attempts are what keep a
stale finding from being copied forward into a new report as though it were still
true.

Dispatch this step in the SAME batch as Steps 5 and 6.

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
- Respect the execution plan's batching. Running an independent step serially is not
  a safe default here — it multiplies wall-clock for no gain in accuracy, and on a
  step this wide it is the difference between a finished report and a truncated one.
- Do not fan out Steps 4, 8 or 9. Classification and severity ranking need one
  reasoner holding the whole finding set.

## Final step
Print a short summary: graph and commit audited, languages scoped in vs. declared
out (with file counts), units selected vs. excluded,
total gaps by class (A/B/C/D), the top 3 by downstream impact, how many findings
are UNVERIFIED, how many subagent findings were rejected for missing an evidence
triple, and the report paths written.
