# Token & Speed Optimization Report

**Date**: 2026-03-06  
**Scope**: All 6 agent prompts + coordinator (`commands/*.md`)  
**Constraint**: Maintain the 6-agent + coordinator architecture

---

## Executive Summary

The extension detects many issues effectively but suffers from high token consumption and slow execution. The root causes are:

1. **Massive prompt duplication** — identical boilerplate repeated across all 6 agents
2. **Redundant file reads** — every agent independently reads all changed files
3. **Sequential execution** — agents run one after another instead of in parallel
4. **Full-file context** — entire files are sent even when only a few lines changed
5. **Verbose persona prose** — lengthy personality descriptions that don't improve output

The proposals below are ordered by ROI and preserve the existing 6-agent + coordinator architecture.

---

## Proposal 1: Diff-Only Context Instead of Full Files

**Impact**: 50–80% input token savings, major speed improvement  
**Effort**: Medium

### Problem

Agents read and analyze entire files even when only a few lines changed. A 500-line file with a 5-line change still costs 500 lines of input tokens × N agents.

### Fix

For each changed file, provide:

1. **The diff hunks** (what actually changed)
2. **±30 lines of surrounding context** per hunk (for understanding the change)
3. **Full file** only when the agent explicitly needs it (progressive disclosure)

The `detect-changed-files.sh` script already knows the diff — extend it with a `--with-diff` flag that includes hunk content. The coordinator passes diffs as the primary context, with file paths available for agents that need full reads.

### Example

```
# Current: agent reads full 500-line file (500 tokens of input)
# Proposed: agent receives only the relevant hunks (~50 tokens)

@@ -42,6 +42,8 @@ function processOrder(order) {
   const total = calculateTotal(order.items);
+  if (total < 0) {
+    throw new Error('Negative total');
+  }
   return submitOrder(order, total);
 }
```

Each agent still has the option to request the full file if its analysis requires it (e.g., the **types** agent might need class-level context), but the default is diff + surrounding context.

---

## Proposal 2: Parallelize Agent Dispatch

**Impact**: ~3–5× wall-clock speedup, no token change  
**Effort**: Low  
**Status**: ✅ Implemented

### Problem

The coordinator in `commands/run.md` runs agents sequentially:

> "For each applicable agent execute the agent's command as a focused sub-task"

With 6 agents this means wall-clock time = sum of all agent times.

### Spec Kit Constraint

Spec Kit has **no runtime execution engine** — it is a prompt/template scaffolding system. All execution is performed by the AI agent reading the Markdown instructions. This means:
- There is no framework-level parallel dispatch API to call
- Parallelism is achieved by **instructing the AI agent** via natural language hints
- The existing spec-kit `[P]` marker pattern (used in `implement.md` for parallel tasks) is the established convention — it's a natural language hint, not an executable directive

### Fix

Rewrite `commands/run.md` Step 4 to use the `[P]` marker convention from `implement.md`, staying agent-agnostic. No framework changes needed.

### Affected File

- `commands/run.md` — Step 4: Agent Orchestration

---

### Implementation Plan

#### What Changes

Replace the current sequential Step 4 in `commands/run.md` with parallel dispatch instructions and a two-phase collect-then-deduplicate flow.

#### Current Step 4 (sequential)

```markdown
## Step 4: Agent Orchestration

For each applicable agent execute the agent's command as a focused sub-task. Provide it with:
- The **CHANGED_FILES** list
- The **GUIDELINES_PATH** (if present)
- The **CONFIDENCE_THRESHOLD** value

### Collect Results

After each agent completes, extract from its output:
- All findings (severity, file, line, description, recommendation)
- Files analyzed count
- Agent status (success, error, skipped)
```

#### Implemented Step 4

```markdown
## Step 4: Agent Orchestration

Execute all applicable agents, providing each with:
- The **CHANGED_FILES** list
- The **GUIDELINES_PATH** (if present)
- The **CONFIDENCE_THRESHOLD** value

All agents are independent, read-only, and marked **[P]** — they can run together in parallel. Agents do not depend on each other's output.

### Collect Results

Wait for all dispatched agents to complete. For each agent, extract from its output:
- All findings (severity, file, line, description, recommendation)
- Files analyzed count
- Agent status (success, error, skipped)

For parallel agents [P], continue with successful agents and report failed ones. Do not re-run failed agents.
```

#### Design Decisions

- **Uses `[P]` marker convention** from spec-kit's `implement.md` — consistent with framework patterns
- **Agent-agnostic** — does not reference Claude Code, Copilot, or specific tool APIs; trusts the AI agent to handle `[P]` natively
- **Error handling mirrors `implement.md`** — *"continue with successful agents, report failed ones"*
- **Brief, declarative style** — bullet points instead of prescriptive prose or blockquotes

#### Risks & Mitigations

| Risk | Likelihood | Mitigation |
|---|---|---|
| AI agent ignores `[P]` and runs sequentially | Medium | No correctness impact — just slower; same results either way |
| Agent timeout when running in parallel | Low | Explicit "continue with successful, report failed" instruction |
| Context window pressure from 6 concurrent sub-tasks | Low | Each sub-task has its own context window; only the coordinator needs space for all results |
| De-duplication receives results in unpredictable order | None | De-duplication logic is order-independent (compares all pairs regardless of arrival order) |

#### Validation

1. Run `/speckit.review` on a small changeset — verify all 6 agents still produce findings
2. Confirm the consolidated report contains de-duplicated findings from all agents
3. Measure wall-clock time before and after (expect ~2–4× improvement depending on AI agent)
4. Run `/speckit.review tests errors` (targeted) — verify partial dispatch still works

#### No Breaking Changes

- Direct agent invocation (`/speckit.review.code`, etc.) is unaffected — those don't go through the coordinator
- Targeted review (`/speckit.review tests errors`) works identically — fewer agents dispatched but still parallel
- The De-Duplicate step is order-independent — no logic change needed
- Config toggles (`agents.code: false`) are evaluated in Step 3 before dispatch — unaffected

---

## Proposal 3: Extract Shared Preamble (Eliminate Prompt Duplication)

**Impact**: ~40% prompt token savings  
**Effort**: Low

### Problem

Each of the 6 agent prompts repeats nearly identical sections verbatim:

| Repeated Section | Approx. Tokens | Copies |
|---|---|---|
| Operating Constraints | ~100 | 6 |
| Confidence Scoring scale | ~80 | 6 |
| Step 1: Determine Changed Files | ~120 | 6 |
| Step 2: Load Project Guidelines | ~80 | 6 |
| Operating Principles | ~150 | 6 |
| Report output template | ~100 | 6 |
| **Total duplicated** | **~630 × 5 extra** | **~3,150 wasted tokens/run** |

### Fix

Create a `commands/_shared-preamble.md` partial containing the common sections. The coordinator injects this once as system context before dispatching agents. Each agent prompt shrinks to **only its unique analysis logic**.

The coordinator pre-resolves and passes to each agent:
- `CHANGED_FILES` list
- `GUIDELINES_PATH`
- `CONFIDENCE_THRESHOLD`
- Shared constraints (read-only, confidence scoring, report format)

### Before (each agent, ~200 lines)

```
## Operating Constraints            ← duplicated
## Confidence Scoring               ← duplicated
## Step 1: Determine Changed Files  ← duplicated
## Step 2: Load Project Guidelines  ← duplicated
## Step 3: [Agent-Specific Logic]   ← UNIQUE
## Step 4: Output Report            ← duplicated
## Operating Principles             ← duplicated
```

### After (each agent, ~60 lines)

```
## Analysis   ← UNIQUE analysis logic only
```

The shared preamble, file list, guidelines, and report format are inherited from the coordinator context.

---

## Proposal 4: Route Files to Relevant Agents Only

**Impact**: ~20–30% input token savings  
**Effort**: Medium

### Problem

Every agent receives *all* changed files, but most agents only need a subset:

| Agent | Actually Needs |
|---|---|
| **code** | All source files |
| **errors** | All source files |
| **tests** | Test files + source files they test |
| **types** | Typed-language files only |
| **comments** | Code files with comments (not JSON, YAML, binary) |
| **simplify** | Source files only (not configs, markdown) |

### Fix

Add a lightweight file-classification step in the coordinator (Step 2.5) that tags each file:

```yaml
source_files:  [auth.ts, service.ts]
test_files:    [auth.test.ts]
typed_files:   [auth.ts, service.ts, auth.test.ts]
config_files:  [config.yml]
doc_files:     [README.md]
```

Then dispatch only the relevant subset to each agent. This avoids sending irrelevant files (e.g., JSON configs) to the **types** or **comments** agents.

### Affected File

- `commands/run.md` — add Step 2.5: File Classification

---

## Proposal 5: Early-Exit for Irrelevant Agents

**Impact**: Up to 100% savings per skipped agent  
**Effort**: Low

### Problem

The **types** agent already has an applicability check for dynamically-typed languages, but it happens *inside the agent* (after the full prompt is already loaded). Other agents have no such checks at all.

### Fix

Add lightweight applicability checks at the **coordinator level** (before dispatching). This avoids the full agent prompt + context window cost for clearly inapplicable agents:

| Agent | Skip Condition |
|---|---|
| **types** | All files are dynamically-typed (`.py`, `.js`, `.rb`, `.sh`) |
| **tests** | No test files AND no source files with testable logic |
| **comments** | All files are non-code (JSON, YAML, binary, images) |
| **simplify** | Changeset is trivially small (< 20 LOC changed) |
| **code** | Never skip (always applicable) |
| **errors** | Never skip (always applicable) |

### Affected File

- `commands/run.md` — add early-exit logic to Step 3: Determine Applicable Agents

---

## Proposal 6: Trim Verbose Agent Personas

**Impact**: ~15% prompt token savings  
**Effort**: Low

### Problem

Several agents have lengthy personality descriptions that consume tokens on every invocation without improving output quality:

| Agent | Verbose Section | Approx. Tokens |
|---|---|---|
| `errors.md` | 5-paragraph "Core Principles" + "Your Tone" (6 bullets) | ~250 |
| `comments.md` | 4-sentence character description | ~80 |
| `types.md` | "Key Principles" (7 bullets) + "Anti-patterns" (7 bullets) + "When Suggesting" (5 bullets) | ~300 |
| `simplify.md` | Multi-paragraph persona | ~100 |
| **Total excess** | | **~730 tokens/run** |

### Fix

Compress each agent persona to 1–2 sentences. Move detailed checklists into the structured analysis steps (which are actionable) rather than personality prose (which is not).

#### Example — `errors.md`

**Before** (~250 tokens):
```markdown
You are an elite error handling auditor with zero tolerance for silent failures 
and inadequate error handling. Your mission is to protect users from obscure, 
hard-to-debug issues...

### Core Principles
1. **Silent failures are unacceptable** — Any error that occurs...
2. **Users deserve actionable feedback** — Every error message...
3. **Fallbacks must be explicit and justified** — ...
4. **Catch blocks must be specific** — ...
5. **Mock/fake implementations belong only in tests** — ...

### Your Tone
You are thorough, skeptical, and uncompromising... You:
- Call out every instance...
- Explain the debugging nightmares...
- Provide specific, actionable...
- Acknowledge when error handling is done well...
- Use phrases like "This catch block could hide..."
- Are constructively critical...
```

**After** (~30 tokens):
```markdown
Audit error handling in changed files. Flag silent failures, broad catches, 
missing logging, unhelpful error messages, and unjustified fallbacks. 
Prioritize findings that would cause debugging nightmares.
```

The Step 3 checklist already captures the *what to analyze* — the persona prose is redundant.

---

## Proposal 7: Coordinator Pre-Reads Files Once

**Impact**: ~50% file I/O token savings  
**Effort**: Medium

### Problem

Every agent independently reads all changed files. If 10 files changed, that's 10 file reads × 6 agents = **60 file reads** of the same content, each consuming input tokens.

### Fix

The coordinator reads all changed files **once** in Step 2 and passes file contents (or diff hunks per Proposal 1) as structured context to each agent. Agents analyze the pre-loaded content rather than re-reading from disk.

This is most effective when combined with Proposal 1 (diff-only context) and Proposal 4 (file routing):

```
Coordinator:
  1. Run detect-changed-files.sh --with-diff
  2. Read changed files / collect diff hunks (ONCE)
  3. Classify files by type
  4. Dispatch to agents with pre-loaded, filtered context
```

### Affected File

- `commands/run.md` — Step 2: Load Context (expand to include file pre-reading)

---

## Proposal 8: Structured Agent Output (JSON Instead of Markdown)

**Impact**: ~10% output token savings  
**Effort**: Low

### Problem

Each agent generates a full markdown report with table formatting. The coordinator then has to *parse* these markdown tables to de-duplicate findings and re-format into the consolidated report. This is both token-wasteful and error-prone.

### Fix

Tell agents to return findings as structured data (JSON-like list), not formatted markdown. The coordinator handles all formatting in one pass:

#### Agent output (before — markdown table):
```markdown
## Review: Error Handling Report

**Files Analyzed**: 5
**Review Scope**: feature branch diff

| # | Severity | File | Line | Finding | Recommendation |
|---|----------|------|------|---------|----------------|
| 1 | Critical | src/api.ts | 88 | Empty catch block | Log and re-throw |
```

#### Agent output (after — structured data):
```yaml
agent: errors
files_analyzed: 5
findings:
  - severity: 95
    file: src/api.ts
    line: 88
    finding: Empty catch block silences database errors
    recommendation: Log the error with context and re-throw
```

Benefits:
- Cheaper to generate (no table alignment tokens)
- Trivial to de-duplicate (structured comparison vs. string parsing)
- Coordinator formats the final report once, consistently

### Affected Files

- All `commands/*.md` — change Step 4/5 output format
- `commands/run.md` — update de-duplication to work on structured data

---

## Summary Impact Matrix

| # | Proposal | Token Savings | Speed Impact | Effort |
|---|----------|--------------|--------------|--------|
| 1 | Diff-only context | **50–80% input** | **Major** | Medium |
| 2 | Parallelize agents | None (tokens) | **~3–5× speedup** | Low |
| 3 | Extract shared preamble | ~40% prompt | None | Low |
| 4 | Route files to agents | ~20–30% input | Moderate | Medium |
| 5 | Early-exit irrelevant agents | Up to 100%/agent | Significant | Low |
| 6 | Trim verbose personas | ~15% prompt | Slight | Low |
| 7 | Pre-read files once | ~50% file I/O | Moderate | Medium |
| 8 | Structured agent output | ~10% output | Slight | Low |

### Recommended Implementation Order (highest ROI first)

1. **Proposal 1** — Diff-only context (largest single token savings)
2. **Proposal 2** — Parallelize agents (largest speed improvement)
3. **Proposal 3** — Extract shared preamble (low effort, high savings)
4. **Proposal 5** — Early-exit checks (low effort, avoids wasted runs)
5. **Proposal 4** — File routing (medium effort, good savings)
6. **Proposal 6** — Trim personas (low effort, moderate savings)
7. **Proposal 7** — Pre-read files (combines well with #1 and #4)
8. **Proposal 8** — Structured output (small but clean improvement)

### Combined Estimated Savings

Implementing all 8 proposals together:
- **Token usage**: ~60–75% reduction (depending on changeset size)
- **Wall-clock time**: ~3–5× faster (primarily from parallelization)
- **Architecture**: 6 agents + coordinator preserved, no behavioral changes
