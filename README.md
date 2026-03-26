# Getting Started with SonarQube Agentic Analysis and Claude Code

> A hands-on guide for SonarQube Cloud customers on using **Agentic Analysis**
> (the Verify phase) and **Context Augmentation** (the Guide phase) together
> with Claude Code to write cleaner code from the very first draft.
>
> All examples use the real [clipper2-java](https://github.com/jon-lipsky-sonarsource/clipper2-java)
> project with actual analysis output from four structured experiments.

---

## Table of Contents

1. [Overview — The Verification Gap](#overview)
2. [How These Tools Actually Work](#how-it-works)
3. [Prerequisites and Setup](#setup)
4. [Phase 1 — Crawl: Manual Verification](#phase-1)
5. [Phase 2 — Walk: Skills and Standards](#phase-2)
6. [Phase 3 — Run: Autonomous Guide-Generate-Verify Loop](#phase-3)
7. [Insights: Getting the Most From These Tools](#insights)
8. [Measuring Impact](#measuring-impact)
9. [Appendix](#appendix)

---

## Overview — The Verification Gap

Every team that adopts a static analysis tool like SonarQube quickly runs into
the same frustration: the tool catches issues *after* the fact. A developer
writes code, pushes a PR, and then — minutes or hours later — the CI pipeline
returns a list of issues that have to be fixed before merge. The code has
already been written, reviewed, and mentally filed away. Now it must be
reopened.

This feedback lag is the **verification gap**: the distance between when code
is written and when quality issues are detected.

AI coding assistants like Claude can close this gap — but only if they have
access to the right context. By default, an AI assistant knows about general
Java best practices and common patterns, but it does not know:

- Which specific Sonar rules are configured for your project
- Which rules your team has elevated to blockers
- How your current code base is structured (what extends what, who calls what)
- Where in the architecture a new class should sit

**SonarQube's MCP server** gives Claude Code two capabilities that address
this directly:

| Capability | Tool category | What it does |
|------------|---------------|--------------|
| **Context Augmentation** (CAG) | `get_guidelines`, `get_current_architecture`, `get_type_hierarchy`, `get_references`, `get_downstream_call_flow`, `get_upstream_call_flow`, `search_by_signature_patterns`, `search_by_body_patterns`, `get_source_code`, `get_intended_architecture` | Injects Sonar rule knowledge and code structure *before* the AI writes code |
| **Agentic Analysis** | `run_advanced_code_analysis` | Runs a full CI-level Sonar analysis on a file *in the editor*, with cross-file context |

Used together, these capabilities enable a **Guide → Generate → Verify** loop
that catches and fixes quality issues in the same session the code is written,
before a line is ever pushed.

This guide shows you how to set it up, how to use it at three levels of
sophistication, and — through real experiments on the clipper2-java project —
what measurable difference it makes.

---

## How These Tools Actually Work

### Context Augmentation: What Gets Injected and Why It Matters

CAG tools fall into two categories: **rule context** (what Sonar will flag)
and **structural context** (how your codebase is organised). Used together
before code generation, they turn the AI from a general-purpose generator
into a context-aware collaborator that understands *your* project's rules
and structure.

#### Rule context: `get_guidelines`

`get_guidelines` returns the specific Sonar rules enabled for your project,
with rule descriptions, compliant and non-compliant code examples, and
configured severity — not generic advice.

It supports three `mode` values:

| Mode | What it returns | When to use |
|------|-----------------|-------------|
| `"project_based"` | Guidelines derived from real open issues in your SonarQube project | When you want to focus on problems already present |
| `"category_based"` | Guidelines from a pre-selected set of rule categories | When starting a new file with no existing issues to draw from |
| `"combined"` | Merges both of the above | Default for most sessions — broadest coverage |

**Key parameters:**

- `languages` (required for category-based and combined modes) — SonarQube
  repository key format: `"java"`, `"python"`, `"typescript"`, `"javascript"`,
  `"csharp"`, `"cpp"`, `"php"`, `"xml"`, `"html"`, `"css"`.
- `categories` (required for category-based and combined modes) — one or more
  of 32 pre-defined categories. The most impactful for Java algorithm code:
  - `"Code Complexity & Maintainability"` — cognitive complexity, loop
    structure, break/continue
  - `"Naming Conventions & Code Style"` — `java:S1659` (grouped declarations),
    `java:S120` (package naming), `java:S117` (local variable naming)
  - `"Exception/Error Handling"` — improper throws, empty catch blocks
  - `"Type System & Generics"` — generic method overloads, bounded wildcards
  - `"Spring Framework"`, `"JPA/Hibernate & Enterprise Java"`,
    `"Modern Java Features"`, `"Serialization & Collections API"` — framework-
    specific Java categories
- `file_paths` (optional) — restrict guideline generation to issues found in
  specific files, useful for targeted pre-generation context.

#### Structural context tools

These tools expose the dependency graph and type hierarchy that SonarQube
builds from your project. They are available for **Java, JavaScript,
TypeScript, Python, and C#** unless noted otherwise.

**`get_current_architecture`** — shows the module/package dependency graph.
Returns nodes with their FQNs, kind (package, module, class), depth, and
dependency lists (`depends_on` / `depended_on_by`).

Recommended workflow: start with `depth=0` to see top-level structure, then
pass a specific FQN as `path_prefix` with a higher depth to drill into a
subsystem. Note that FQN format varies by language — JavaScript/TypeScript
modules use `:` as a separator (e.g., `my-module:com.example.service`),
while Java uses `.`.

**`get_intended_architecture`** — returns the user-defined allowed module
relationships (if any are configured). The `meta.policy` field indicates
whether unlisted dependencies are denied by default. If `constraints` is
empty, all relationships are implicitly allowed.

**`get_type_hierarchy`** — shows the full inheritance and implementation
graph for a class, interface, enum, record, or struct. Returns parent and
child nodes with their FQNs, file paths, and relationship kind
(`EXTENDS`, `IMPLEMENTS`). Available for Java, JS, TS, Python, and C#.

**`get_references`** — shows all **direct** inbound and outbound references
for a given class, interface, or module FQN (not method FQNs — for method-
level analysis use the call flow tools). Returns each referencing node's FQN,
file path, and the `dependency_kinds` that describe the relationship. Use this
to understand the blast radius of any change.

**`get_downstream_call_flow`** / **`get_upstream_call_flow`** — traces the
call graph from a method FQN forward (callees) or backward (callers) to a
configurable depth. Useful for understanding what a method calls, or which
entry points reach it. **Java only.**

- `depth=0` returns only the method itself; `depth=1` (default) returns its
  direct callees/callers; higher values recurse further.
- If you pass a class or module FQN instead of a method FQN, the tool
  automatically reroutes to `get_references`.
- External library nodes appear in the graph with `null` for `file_path`,
  `signature`, and line numbers.

**`search_by_signature_patterns`** / **`search_by_body_patterns`** — search
the UDG by regex against method/class declarations (signature) or their
implementations (body). Each pattern must be single-line (no newlines).
**Java only.**

- `include_code_regex_list`: list of regex patterns to match (OR by default;
  pass `regex_lists_operator: "AND"` to require all patterns)
- `include_glob` / `exclude_glob`: file path filters (e.g., `**/model/**/*.java`)
- `limit` (default 10): max results; response includes `truncated: true` if
  more exist
- Results grouped by pattern (OR mode) or under `"ALL_PATTERNS"` (AND mode)

**`get_source_code`** — retrieves the full source code of a method or class
given its FQN. Use after a search to inspect an actual implementation.
**Java only.**

All structural tools accept an optional `fields` parameter — a comma-
separated list of field names to include in the response — useful for
reducing response size when only specific data is needed.

### Agentic Analysis: What CI-Level Precision Means in Practice

The `run_advanced_code_analysis` tool is fundamentally different from other
"AI code review" approaches. It does not ask Claude to review its own code
using its training knowledge. It submits the actual source file to the
SonarQube analysis engine — the same engine that runs in CI — and returns the
results.

This means:

- **Cross-file context**: the analysis has access to the full project context
  (types, call graphs, data flows) that single-file tools lack
- **Rule fidelity**: the results are identical to what CI would return — no
  false confidence from an AI that "thinks" the code is fine
- **Fast feedback**: the analysis runs in a Docker container on your local
  machine and returns results in seconds

### The Guide → Generate → Verify Loop Explained

The three phases described in this guide are different points on a spectrum
of how much the SonarQube tools are integrated into the development flow:

```
Guide phase   →   Generate phase   →   Verify phase
(get_guidelines,    (Claude writes        (run_advanced_code_analysis
 architecture        the code)             confirms it is clean)
 tools)
```

- **Phase 1 (Crawl)**: Only the Verify phase is used. Code is written first,
  then checked. Issues are found and fixed reactively.

- **Phase 2 (Walk)**: Guide + Verify. Guidelines are fetched before writing.
  The AI generates code that is already aware of the rules, then confirms
  cleanliness. Fewer iterations needed.

- **Phase 3 (Run)**: Full autonomous loop. The AI calls all three phases
  automatically, iterating until the file is clean before declaring the task
  complete.

Each phase builds on the previous one. Most teams will start at Phase 1 and
progressively adopt more automation as they build confidence.

---

## Prerequisites and Setup


### What You Need

**SonarQube Cloud Enterprise account** — The MCP server connects to
SonarQube Cloud to retrieve your project's configured rules and to submit
files for analysis. The Agentic Analysis feature (`run_advanced_code_analysis`)
requires a SonarQube Cloud Enterprise subscription. You will also need your
**organization ID** and a **project key** for an existing project analysed in
SonarQube Cloud using a supported language (Java, Python, JavaScript/TypeScript,
CSS, HTML, or XML).

**Docker** — The SonarQube MCP server is distributed as a Docker image
(`mcp/sonarqube`). Claude Code starts the container automatically when the
MCP server is needed. The container mounts your project directory read-write
so the analysis engine can access your source files.

**A SonarQube user token** — The MCP server uses this token to authenticate
against SonarQube Cloud. It must be a **user token**, not an organization-
scoped token, because organization tokens do not have the permissions needed
for analysis submission. Generate one in SonarQube Cloud under
**My Account → Security → Generate Tokens**.

### Step 1: Set Your Token in the Environment

Add your user token to your shell profile for persistent access across sessions:

```bash
echo 'export SONARQUBE_TOKEN="your_token_here"' >> ~/.zshrc && source ~/.zshrc
```

Or set it for the current terminal session only:

```bash
export SONARQUBE_TOKEN="your_token_here"
```

### Step 2: Register the MCP Server with Claude Code

Change to your project directory, then run this command — replacing
`your_org_id` and `your_project_key` with your actual values:

```bash
claude mcp add sonarqube -s user \
  -e SONARQUBE_URL=https://sonarcloud.io \
  -e SONARQUBE_ORG=your_org_id \
  -e SONARQUBE_PROJECT_KEY=your_project_key \
  -e SONARQUBE_TOOLSETS=cag,projects,analysis \
  -e SONARQUBE_ADVANCED_ANALYSIS_ENABLED=true \
  -- docker run -i --rm --pull=always \
  -e SONARQUBE_URL -e SONARQUBE_TOKEN \
  -e SONARQUBE_ORG -e SONARQUBE_PROJECT_KEY \
  -e SONARQUBE_TOOLSETS -e SONARQUBE_ADVANCED_ANALYSIS_ENABLED \
  -v "$(pwd):/app/mcp-workspace:rw" mcp/sonarqube
```

This registers the server at user scope (`-s user`) so it is available in any
Claude Code session for this project. The key environment variables:

| Variable | Purpose |
|----------|---------|
| `SONARQUBE_TOOLSETS=cag,projects,analysis` | Enables Context Augmentation and Agentic Analysis tools |
| `SONARQUBE_ADVANCED_ANALYSIS_ENABLED=true` | Required for `run_advanced_code_analysis` |

Additional toolsets (`issues`, `quality-gates`, `rules`, `duplications`,
`measures`, `security-hotspots`, `dependency-risks`) can be added to
`SONARQUBE_TOOLSETS` as needed — see the
[SonarQube MCP Server docs](https://docs.sonarsource.com/sonarqube-mcp-server/build-and-configure/environment-variables#tool-enablement).

### `.mcp.json` Configuration

After running `claude mcp add`, the configuration is stored at
`~/.claude/mcp.json`. See the [Appendix](#appendix) for the full reference
format, including how `SONARQUBE_TOKEN` is read from your shell environment
at runtime rather than stored in the file.

### CLAUDE.md Agent Directives

A `CLAUDE.md` file in your project root (or `~/.claude/CLAUDE.md` for global
directives) tells Claude Code how to behave. The SonarQube integration is most
effective when CLAUDE.md instructs Claude to use the tools automatically.

See the [Appendix](#appendix) for ready-to-use CLAUDE.md templates, or copy
the templates from `examples/CLAUDE.md.template`.

### Verifying Connectivity

After setup, open Claude Code in your project directory and run:

```
What SonarQube tools do you have available? List them.
```

Claude should respond with a list of tools including `get_guidelines`,
`get_current_architecture`, `run_advanced_code_analysis`, and others.
If it does not, check that Docker is running and that the MCP server
registered correctly (`claude mcp list`).

### Troubleshooting

**MCP server fails to start?** Make sure Docker is running on your machine.
The MCP server is distributed as a Docker container — if Docker is not running,
Claude Code cannot start the server.

---

## Phase 1 — Crawl: Manual Verification

### When to Use This Phase

Phase 1 is the right starting point when:
- You want to try the tools with minimal setup (no CLAUDE.md changes needed)
- You are working on an existing codebase and want to check code you have
  already written
- You want to understand what Sonar would flag before pushing a PR

Phase 1 adds one step to your existing workflow: after writing code, you ask
Claude to analyse the file. That is it.

### Example Prompts

The simplest Phase 1 prompt:

```
I just added methods to src/main/java/clipper2/Clipper.java.
Can you run a Sonar analysis on it and tell me what issues it found?
```

A slightly more guided prompt that produces actionable output:

```
Please analyse src/main/java/clipper2/Clipper.java with
run_advanced_code_analysis on branch experiment/phase1/feat-05.
For each issue found in code I added (after line 1180), explain
what rule was violated and suggest a fix.
```

### What to Expect — Real Output

In the Phase 1 experiment (Feature 5 — Geometric Utilities), implementing
`centroid()`, `perimeter()`, `convexHull()`, `isConvex()`, and
`hausdorffDistance()` in `Clipper.java` without any Sonar guidance produced
**8 new issues** in the first draft, all of the same type:

```
java:S1659 — Declare "sumY" on a separate line
  Line 1198:  double sumX = 0, sumY = 0;

java:S1659 — Declare "cx" and all following declarations on a separate line
  Line 1202:  double area = 0, cx = 0, cy = 0;

java:S1659 — Declare "dy" on a separate line
  Line 1467:  double dx = pa.x - pb.x, dy = pa.y - pb.y;
```
*(plus 5 more of the same rule across the two overloads)*

All 8 were fixed in **one round of edits** by splitting each grouped
declaration onto its own line. The analysis was re-run and confirmed clean
for all new code.

**Key observation**: Without guidelines, the AI consistently grouped related
variable declarations on one line for compactness — a natural writing style
that violates `java:S1659`. The fix is mechanical once the rule is known.

---

## Phase 2 — Walk: Skills and Standards

### Creating Skills for Consistent Behaviour

Phase 2 introduces two ideas:

1. **Fetch guidelines before writing code** — calling `get_guidelines` first
   means the AI generates code that is already aware of your project's rules
2. **Skills** — reusable slash commands that encode the right tool-calling
   sequence, so you do not have to remember to do it manually

#### The `/sonar-scan` Skill

`/sonar-scan` is designed for mid-task use. It fetches guidelines for the
files you are working on, runs analysis, and reports a summary. Use it after
you finish a feature to see whether any issues crept in.

See `examples/sonar-scan.md` for the full skill definition.

#### The `/sonar-verify` Skill

`/sonar-verify` is a lightweight "did I break anything" check. It runs
`run_advanced_code_analysis` on the files you have modified and returns
a pass/fail result. Use it frequently — after each logical unit of work.

See `examples/sonar-verify.md` for the full skill definition.

### CLAUDE.md Directives for Consistent Behaviour

Add the minimal CLAUDE.md template from `examples/CLAUDE.md.template` to
your project. This instructs Claude to:
- Run analysis before finalising any file
- Fix all issues above INFO severity before completing a task

### Example Session — Real Output

The Phase 2 experiment implemented Feature 6 (Visvalingam-Whyatt simplification).
Before writing any code, the following `get_guidelines` call was made:

```
get_guidelines(
  mode: "combined",
  categories: ["Code Complexity & Maintainability",
               "Naming Conventions & Code Style",
               "Exception/Error Handling"],
  languages: ["java"]
)
```

The top rules returned (relevant to algorithm code):

```
5. Java: Cognitive Complexity of methods should not be too high
17. Java: Package names should comply with a naming convention
19. Java: Standard outputs should not be used directly to log anything
```

With this context loaded, the first-draft VW implementation produced **2 new issues**
(compared to 8 in the Phase 1 experiment for a similar-complexity feature):

```
java:S135 — Reduce the total number of break and continue statements
  Line 1316: while (remaining > minVertices && !heap.isEmpty()) {
    → had both 'continue' (stale entry check) and 'break' (area threshold)

java:S107 — Method has 11 parameters, which is greater than 7 authorized
  Line 1335: private static void vwUpdateVertex(...)
```

Both issues required structural refactoring rather than trivial line splits:
- `java:S135`: restructured the loop to use if-else instead of continue
- `java:S107`: introduced a `VwState` inner class to bundle mutable arrays

After one fix round, a second analysis found the restructuring had raised
cognitive complexity in `vwRun` to 19. Extracting four helper methods
(`vwInitLinks`, `vwSeedHeap`, `vwRemoveVertex`, `vwUpdateNeighbour`) resolved
this in a third iteration.

**Total: 2 issues → 0 in 3 iterations.**

Crucially: **zero `java:S1659` issues** (the 8 issues that appeared in Phase 1).
The guidelines call explicitly prevented the grouped-declaration pattern.

---

## Phase 3 — Run: Autonomous Guide-Generate-Verify Loop

### The CLAUDE.md Rules for Autonomous Operation

The full CLAUDE.md template (see `examples/CLAUDE.md.template`) instructs
Claude to run the complete loop automatically:

1. Call `get_guidelines` with the relevant categories and files before writing
2. Consult architecture tools when adding to or modifying existing classes
3. After every file modification, call `run_advanced_code_analysis`
4. If issues are found, look up the rule documentation and self-correct
5. Re-run analysis until the file is clean
6. Never declare the task complete until all modified files pass

With these directives in place, the entire Guide → Generate → Verify loop
runs without any explicit prompting from the developer. You describe the
feature; Claude handles the rest.

### What the Loop Looks Like End to End

A Phase 3 session for implementing Feature 4 (AffineTransform) looked like this:

```
1. Developer: "Implement Feature 4 per specs/04-affine-transformations.md"

2. Claude calls:
   get_guidelines(combined, ["Code Complexity & Maintainability",
   "Naming Conventions & Code Style", "Exception/Error Handling",
   "Type System & Generics"], java)

3. Claude calls:
   get_current_architecture(depth=0)
   → sees top-level package structure
   get_current_architecture(path_prefix="clipper2", depth=1)
   → confirms clipper2.transform is a sensible new package
   → no existing transform package to conflict with

4. Claude writes AffineTransform.java (separate field declarations,
   simple method structure, no System.out, follows naming conventions)

5. Claude calls:
   run_advanced_code_analysis("src/.../AffineTransform.java")
   → 1 issue: java:S120 package naming (pre-existing project-wide issue)
   → 0 fixable issues

6. Claude writes Clipper.java additions (scalePath, rotatePath, etc.)

7. Claude calls:
   run_advanced_code_analysis("src/.../Clipper.java")
   → 0 new issues in new code

8. Claude reports: "AffineTransform.java: 1 pre-existing S120 (unfixable).
   All new code in both files passes with no new issues. Feature 4 complete."
```

### Real Results

| Metric | Phase 3 (AffineTransform) |
|--------|--------------------------|
| Tool calls (total) | 5 (1 guidelines + 2 architecture + 2 analysis) |
| First-draft fixable issues | 0 |
| Fix iterations needed | 0 |
| Remaining issues in new code | 0 |

---

## Insights: Getting the Most From These Tools

These insights emerged directly from running the four experiments on clipper2-java.

### 1. Call `get_guidelines` Before Writing, Not After

The data is unambiguous: guideline retrieval has far more impact when it
happens *before* code generation.

In the Phase 1 experiment (no guidelines), 8 `java:S1659` issues appeared
in the first draft — all grouped variable declarations on one line. In Phase 2
(guidelines first), there were 0 `java:S1659` issues. The rule is simple and
mechanical, but the AI only applies it consistently when it knows the rule
exists before it starts writing.

**For Java projects**, the most impactful categories to request:
- `"Code Complexity & Maintainability"` — catches cognitive complexity, loop
  structure, and break/continue issues before they appear
- `"Naming Conventions & Code Style"` — covers `java:S1659`, `java:S120`,
  `java:S117` (the most common issues in this project)
- `"Exception/Error Handling"` — catches improper throws and empty catch blocks
- `"Type System & Generics"` — relevant for utility methods with overloads

For non-Java languages, the language-agnostic categories (`"Code Complexity &
Maintainability"`, `"Naming Conventions & Code Style"`, `"Exception/Error
Handling"`) apply broadly. Language-specific categories exist for Python
(`"Django/Flask Frameworks"`, `"Data Science (Pandas, NumPy)"`,
`"Type Hints & Dynamic Features"`) and JavaScript/TypeScript projects can
benefit from `"REST API Development"`, `"Web Security (XSS, CSRF, Injection)"`,
and `"Authentication & Authorization"`.

### 2. Use Architecture Tools Before Writing — Not Just for New Packages

Before creating the `clipper2.transform` package in the Phase 3 experiment,
`get_current_architecture(depth=0)` confirmed the top-level package structure
in one call. A second call with `path_prefix="clipper2"` and `depth=1`
confirmed:
- No existing `transform` package in the project
- The new package would only need to depend on `clipper2.core`
- No circular dependency risk

This took 2 tool calls and a few seconds. Without them, the AI might have
placed the class in an existing package that already had too many
responsibilities, or introduced a dependency that violated the intended
architecture.

**When to use each structural tool:**

| Situation | Tool to use |
|-----------|-------------|
| Creating a new package or module | `get_current_architecture` (depth=0 first, then drill in) |
| Adding a new class to an existing hierarchy | `get_type_hierarchy` on the base class |
| Changing a class's public API | `get_references` — see every caller |
| Understanding what a method delegates to | `get_downstream_call_flow` (Java only) |
| Finding which entry points reach a method | `get_upstream_call_flow` (Java only) |
| Finding an existing pattern to replicate | `search_by_signature_patterns` (Java only) |
| Checking if a dependency is allowed | `get_intended_architecture` |

For modifications to existing classes, `get_references` returns only **direct**
references — not transitive ones. If you need the full impact, combine it
with `get_downstream_call_flow` on the specific method being changed.

### 3. Scope Analysis to the File Being Changed

`run_advanced_code_analysis` accepts a single file path. The experiments
consistently showed that the right workflow is:

1. Analyse the file you just wrote
2. Fix the issues
3. Analyse again
4. Move to the next file

Trying to analyse the whole project at once produces a list of pre-existing
issues that obscures the signal from your new code. In the Phase 1 experiment,
Clipper.java had 22 pre-existing issues — none from the new code. By scoping
analysis to new code and cross-referencing line numbers with your additions,
you can immediately distinguish new issues from old ones.

### 4. Some Issues Require Structural Fixes, Not Cosmetic Ones

The Phase 2 experiment showed that `java:S107` (too many parameters) and
`java:S135` (break and continue in same loop) require genuine refactoring.
The right response to `java:S107` was to introduce a `VwState` context class.
The right response to `java:S135` was to restructure the loop.

When an issue is returned that is not a simple style violation, look up the
rule key and read the full documentation before attempting a fix. The rule
documentation includes compliant and non-compliant examples that make the
fix pattern clear.

### 5. Fix Interactions Are Caught by the Iterative Loop

In the Phase 2 experiment, fixing `java:S135` (by using if-else instead of
continue) increased the nesting depth of the while loop body, which raised
the cognitive complexity of `vwRun` from below 15 to 19 — triggering a
new `java:S3776` issue.

This is a common real-world pattern: one fix creates a secondary issue. The
iterative verify loop catches this automatically. Without the loop, the
secondary issue would have been pushed to CI.

### 6. The Package Naming Issue (`java:S120`) Is a Project-Wide Constant

In all four experiments, `java:S120` appeared in every file analysed. This
is because the project's root package is `clipper2` — containing the digit
`2` — which violates the naming convention. This is a pre-existing project-
wide issue and is not fixable without renaming the entire package hierarchy.

When you see a consistent issue across all files that you did not introduce,
check whether it is pre-existing before attempting to fix it.

---

## Measuring Impact

The following data comes from implementing three features under four different
tool-usage conditions, using clipper2-java as a test bed.

### Experiment Design

| Branch | Condition | Feature | Tools Used |
|--------|-----------|---------|------------|
| `experiment/no-tools/feat-04` | Baseline | AffineTransform | None |
| `experiment/phase1/feat-05` | Phase 1 | Geometric Utilities | Verify only |
| `experiment/phase2/feat-06` | Phase 2 | Visvalingam–Whyatt | Guide + Verify |
| `experiment/phase3/feat-04b` | Phase 3 | AffineTransform (repeat) | Full loop |

Feature 4 (AffineTransform) is implemented twice to provide a direct
before/after comparison on identical work.

### Results

| Metric | Baseline | Phase 1 | Phase 2 | Phase 3 |
|--------|----------|---------|---------|---------|
| First-draft fixable issues (new code only) | 2 | 8 | 2 | **0** |
| Issues after self-correction | 1* | 0 | 0 | 0 |
| Verify iterations | 2 | 2 | 3 | 2 |
| Tool calls | 0 | 2 | 4 | 4 |
| S1659 (grouped declarations) issues | 2 | 8 | **0** | **0** |
| S107/S135 (structural) issues | 0 | 0 | 2 | 0 |

*\* 1 unfixable S120 package naming issue (pre-existing project convention)*

### Observations

**Guidelines pre-loading eliminates the most common issue class.** The
`java:S1659` pattern (multiple declarations per line) appears 8 times in Phase 1
(no guidelines) and 0 times in Phase 2 and Phase 3. The AI naturally reaches
for grouped declarations; the guidelines call prevents this.

**More complex algorithms produce harder-to-predict issues.** Feature 6 (VW
simplification — a complex algorithm with a priority queue) produced structural
issues (`java:S107`, `java:S135`) that required genuine refactoring. Feature 4
(AffineTransform — pure math, immutable class) produced only style issues.
The fix effort scales with algorithmic complexity.

**The Phase 3 full loop is the only condition that reaches zero fixable issues
in the first draft.** This is the target state: the developer gets clean code
without any manual iteration.

**Tool call overhead is modest.** The full Phase 3 loop used 4 tool calls
(1 guidelines + 1 architecture + 2 analysis). This is roughly 10-15 seconds
of additional latency for a feature that would otherwise require multiple
CI pipeline runs to clean up.

---

## Appendix

### Full `.mcp.json` Reference

When you run `claude mcp add sonarqube ...`, Claude Code stores the server
configuration. A minimal SonarQube MCP configuration:

```json
{
  "mcpServers": {
    "sonarqube": {
      "command": "docker",
      "args": [
        "run", "-i", "--rm", "--pull=always",
        "-e", "SONARQUBE_URL",
        "-e", "SONARQUBE_TOKEN",
        "-e", "SONARQUBE_ORG",
        "-e", "SONARQUBE_PROJECT_KEY",
        "-e", "SONARQUBE_TOOLSETS",
        "-e", "SONARQUBE_ADVANCED_ANALYSIS_ENABLED",
        "-v", "${PWD}:/app/mcp-workspace:rw",
        "mcp/sonarqube"
      ],
      "env": {
        "SONARQUBE_URL": "https://sonarcloud.io",
        "SONARQUBE_TOKEN": "${SONARQUBE_TOKEN}",
        "SONARQUBE_ORG": "your_org_id",
        "SONARQUBE_PROJECT_KEY": "your_project_key",
        "SONARQUBE_TOOLSETS": "cag,projects,analysis",
        "SONARQUBE_ADVANCED_ANALYSIS_ENABLED": "true"
      }
    }
  }
}
```

**Note**: `SONARQUBE_TOKEN` is read from your shell environment at runtime.
Set it in `~/.zshrc` (or `~/.bashrc`) as described in the Setup section above
rather than hard-coding it in the JSON file.

### Full CLAUDE.md Template

See `examples/CLAUDE.md.template` for the ready-to-use templates.
Two variants are provided:

- **Minimal** — for Phase 1/2 use: instructs Claude to run analysis before
  finalising files
- **Full** — for Phase 3 autonomous loop: complete Guide → Generate → Verify
  directives

### Skill Definitions

See the `examples/` directory for ready-to-install skill files:

- `sonar-scan.md` — `/sonar-scan` skill (guidelines + analysis, mid-task)
- `sonar-verify.md` — `/sonar-verify` skill (analysis only, lightweight)
- `sonar-review.md` — `/sonar-review` skill (full pre-PR review)

To install a skill globally:

```bash
cp examples/sonar-scan.md ~/.claude/commands/sonar-scan.md
cp examples/sonar-verify.md ~/.claude/commands/sonar-verify.md
cp examples/sonar-review.md ~/.claude/commands/sonar-review.md
```

Or include them in a project-local `.claude/commands/` directory to share
them with your team via version control.
