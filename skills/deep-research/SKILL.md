---
name: deep-research
description: |-
  Conduct deep, multi-agent research on a topic and document findings in an Obsidian vault with optional goodmem ingestion. Auto-scales from single-agent (narrow topics) to full 3-tier hierarchy with Opus managers and Sonnet data collectors (broad topics). Use when the user asks to "deep research", "do comprehensive research on", "research everything about", "build a knowledge base on", or "create a reference on" a topic. Do NOT use for quick factual questions, single lookups, or casual "what is X" queries.
argument-hint: '<topic> [--path <vault-path>] [--tier <1-5>]'
allowed-tools: Bash, Read, Write, Grep, Glob, Agent, TodoWrite, TaskCreate, TaskUpdate, WebSearch, WebFetch, mcp__plugin_goodmem_goodmem__goodmem_memories_retrieve, mcp__plugin_goodmem_goodmem__goodmem_memories_get, mcp__plugin_goodmem_goodmem__goodmem_memories_create
---

# Deep Research

You are conducting a deep research run on the user's behalf. Your job is to decompose the topic, dispatch research agents, and produce a permanent Obsidian vault reference. You run fully autonomously — no checkpoints, no pauses. Report results at the end.

## Step 1: Parse arguments

The user passed an argument string. Parse it into:

1. **Topic** — everything before the first `--` flag. Required. If empty, ask the user what to research and stop.
2. **--path** — optional vault path override (absolute). If omitted, auto-detect in Step 3.
3. **--tier** — optional forced tier (1-5). If omitted, auto-detect in Step 3.

## Step 2: Reconnaissance

Query existing knowledge BEFORE planning. Run these in parallel:

1. **GoodMem Learnings** (if configured) — search for the topic using `goodmem_memories_retrieve`. The orchestrator's environment determines the space ID and reranker config.

2. **Existing vault** — Glob for related folders and files at the user's vault path (typically `~/vault/` or wherever Obsidian stores notes):

```
Glob: <vault-root>/**/*<topic-keywords>*
Glob: <vault-root>/Libraries/*/00 - Index.md
Glob: <vault-root>/Projects/*/00 - Index.md
```

3. **Existing vault content check** — if files already exist at the target vault path, read the MOC to understand what's covered.

### Recon decision

If the topic is already well-covered (existing vault section with 5+ files and recent dates), tell the user:
> "Found existing research at <path> (<N> files, last updated <date>). Want me to update/extend it, or do a fresh deep research run?"

Wait for the user's answer. If they want an update, adjust your plan to fill gaps rather than re-research everything.

If the topic is NOT well-covered, proceed to Step 3.

## Step 3: Planning

### 3a: Decompose the topic

Break the research topic into N non-overlapping domains. Each domain should be:
- Self-contained (can be researched independently)
- Non-overlapping (no two domains cover the same sub-topic)
- Roughly equal in scope (each produces a file of similar size)

For each domain, produce:
- A domain name (becomes the vault file title)
- A numbered file name following the `NN - Title.md` convention
- 10-30 specific sub-questions to investigate
- A list of likely source types (web, context7, vault, GitHub)

### 3b: Auto-detect tier (unless --tier forced)

Every tier dispatches an Opus manager with mandatory Sonnet collectors. No tier does solo research.

| Domains | Tier | Collector floor | Behavior |
|---|---|---|---|
| 1 | 1 | 2 | Single manager |
| 2 | 2 | 3 | Single manager per domain |
| 3-4 | 3 | 4 | Single manager per domain |
| 5-7 | 4 | 6 | Multiple managers, intermediate synthesis files |
| 8+ | 5 | 8 | Large-scale multi-manager, intermediate synthesis, extensive cross-referencing |

### 3b.1: Compute COLLECTOR BUDGET per domain

The tier floor is a MINIMUM, not the actual budget. Compute per domain:

```
questions = number of sub-questions in this domain's SCOPE
floor = tier floor from table above (2, 3, 4, 6, or 8)
computed = ceil(questions / 4)     # ~4 questions per collector
budget = max(floor, computed)
budget = min(budget, 10)           # hard cap at 10 per manager
```

Examples:
- Tier 1, 6 questions: max(2, ceil(6/4)) = max(2, 2) = 2
- Tier 2, 10 questions: max(3, ceil(10/4)) = max(3, 3) = 3
- Tier 3, 16 questions: max(4, ceil(16/4)) = max(4, 4) = 4
- Tier 4, 20 questions: max(6, ceil(20/4)) = max(6, 5) = 6
- Tier 4, 30 questions: max(6, ceil(30/4)) = max(6, 8) = 8
- Tier 5, 40 questions: min(max(8, ceil(40/4)), 10) = min(10, 10) = 10

This ensures narrow domains don't waste collectors while broad domains get proportional coverage. The budget is computed per domain, not globally — different domains may get different budgets.

### 3c: Auto-detect vault path (unless --path forced)

```
1. Check existing vault folders:
   Glob <vault-root>/Libraries/*/ and <vault-root>/Projects/*/
   If topic matches an existing folder name (case-insensitive substring)
     -> use that existing folder
2. If no match, classify:
   Known library/framework/tool -> <vault-root>/Libraries/<Name>/
   Everything else -> <vault-root>/Projects/<Sanitized Topic>/
3. If path exists with content -> write alongside, update MOC
```

Sanitize the topic for use as a folder name: preserve spaces (Obsidian handles them fine), remove special characters except hyphens.

### 3d: Create scratch directory (Tier 4+ only)

```bash
mkdir -p /tmp/deep-research-$(date +%s)
```

Store this path — managers will write intermediate files here.

### 3e: Log the plan

Use TaskCreate to log each domain as a task. This gives the user visibility into progress:

```
TaskCreate: "Research domain: <domain name>" for each domain
TaskCreate: "Write vault docs + MOC" for the final phase
TaskCreate: "Memory integration + cleanup" for the last phase
```

## Step 4: For each domain -- collect and synthesize

Process each domain ONE AT A TIME (sequential). For each domain, YOU (the skill orchestrator, Opus main agent) dispatch Sonnet collectors and synthesize their findings incrementally into the vault file yourself. No separate synthesizer subagent -- you ARE the synthesizer.

**Why you do both dispatching AND synthesis**: Subagents do NOT reliably receive the Agent tool at runtime (confirmed Claude Code platform limitation). You (the main agent running this skill) are the only agent guaranteed to have it. And since you're already Opus, there's no quality loss from synthesizing inline vs dispatching a separate Opus synthesizer.

### 4a: Plan ALL collector tasks upfront (commit before dispatching)

Compute COLLECTOR BUDGET per 3b.1. Break the domain's SCOPE into EXACTLY that number of non-overlapping tasks -- no more, no fewer. Each collector covers ~3-5 sub-questions.

| Collector task type | What it does | Best for |
|---|---|---|
| Web research | 3-5 WebSearch queries on specific sub-topics | Current state, blog posts, engineering posts |
| Library docs | context7 resolve + query for specific libraries | API syntax, config options |
| Vault + memory scan | Read existing vault files + goodmem retrieve | Prior learnings, existing reference docs |
| GitHub/community | gh CLI searches, issue scans | Open issues, release notes |
| Academic/specs | WebSearch for arxiv, RFCs, official specs | Foundational concepts |

**MANDATORY PRE-COMMITMENT**: Before dispatching ANY collector, present the full collector plan as a table and commit to executing every row:

```
Domain: <name> -- COLLECTOR BUDGET: N
| # | Type | Task | Sub-questions covered |
|---|---|---|---|
| 1 | Web research | <specific task> | Q1, Q3, Q7 |
| 2 | Library docs | <specific task> | Q2, Q5 |
| ... (continue for all N collectors) |
```

Then create TaskCreate entries for each collector -- one per planned task:
```
TaskCreate: "Collector 1/N: <task>" (domain <domain name>)
TaskCreate: "Collector 2/N: <task>" (domain <domain name>)
... (one per collector, N total)
```

This pre-commitment is the enforcement mechanism. You will verify all N TaskCreate entries are completed before moving to Step 4d.

### 4b: Scaffold the output file

Before dispatching any collectors, write the skeleton to the OUTPUT PATH:

```markdown
---
goodmem_ingest: true
goodmem_scope: cross-project
type: reference
topic: <topic-keyword>
date: <today YYYY-MM-DD>
tags: [<topic>, <sub-topics>]
---

# <DOMAIN title>

<1-2 sentence overview of the domain>
```

### 4c: Serial collector loop -- incremental synthesis (STRICTLY one dispatch per turn)

**EXECUTION RULE -- overrides Claude Code's default parallel-tool-call bias for this step.**

Your system prompt tells you "if you intend to call multiple tools and there are no dependencies between the calls, make all of the independent calls in the same function_calls block." **That rule does NOT apply here.** In this loop, each collector's briefing is DERIVED from the state of OUTPUT_PATH after the previous collector's synthesis -- so there IS a hard data dependency between dispatches, even though step 4a pre-committed the task list.

Concrete consequences:
- Emit **exactly ONE Agent tool call per assistant turn** during this loop. Never two.
- Never put an Agent call in the same `function_calls` block as another Agent call.
- Never dispatch collector `i+1` until collector `i`'s findings have been read, integrated into OUTPUT_PATH via Edit, and the integration verified by re-reading the file.
- If you find yourself composing two Agent calls together, STOP. Delete one. Run them in separate turns.

If you dispatch all N collectors without synthesis between them, you have violated this rule and the incremental-synthesis purpose of this skill is defeated: collector findings become a 12-20K-word context bomb at the end, later briefings cannot adapt to earlier gaps, and the output file is written in one rushed pass.

You MUST dispatch every collector planned in Step 4a. Stopping early because findings seem "sufficient" is a failure. The whole point of N collectors is proportional coverage -- one collector cannot substitute for the full plan.

**The loop -- N iterations, one collector per iteration:**

FOR i = 1 to N:

(a) **Read OUTPUT_PATH first.** Use the Read tool on OUTPUT_PATH before doing anything else in this iteration. For i=1 the file is just the scaffold from 4b -- read it anyway so the pattern is identical across iterations. The current file state IS the input to step (b); without this Read, you cannot compute the next briefing correctly.

(b) **Compute collector i's briefing** from two inputs:
  - The task you planned for position `i` in step 4a's table
  - The current OUTPUT_PATH content you just read -- specifically: what sub-questions are already answered, what gaps remain, what sources are already cited
  If earlier collectors already covered a sub-question that was on collector i's slate, pivot collector i to an adjacent gap in the same domain. The 4a plan is the minimum coverage commitment, not a verbatim script -- refinement based on accumulated file state is expected.

(c) **Mark TaskCreate entry i as in_progress** via TaskUpdate. This and step (d) may be in the same turn.

(d) **Dispatch collector i** -- this assistant turn must contain ONE Agent call and no other tool calls:
   ```
   Agent({
     description: "Collect <task type> for <domain name>",
     subagent_type: "deep-research:data-collector",
     model: "sonnet",
     prompt: "You are a DATA COLLECTOR for the deep-research plugin. Execute ONE narrow data-collection task and return structured raw findings.\n\nTASK: <specific collection job from step (b)>\nSOURCES TO CHECK: <explicit queries, URLs, library IDs, or paths>\nALREADY COVERED (do not re-fetch): <bullets from step (a), or 'nothing yet' if i=1>\nMAX OUTPUT: 2000 words\n\nReturn one H2 section per source with Date, Relevance, and Claims. End with a Collection Summary. Do NOT synthesize across sources. Do NOT write files. Flag any claim from model recall with [recall]."
   })
   ```
   Also announce the dispatch in your assistant text so the user can see the count: `Dispatching collector <i>/N for domain "<domain>": <task type> -- <task description>`.

(e) **When the collector returns**, in the NEXT turn, integrate its findings:
   - Read the collector's findings from the tool return value
   - Apply confidence grading to each claim: [P] primary, [S] secondary, [P x N] cross-verified, [V] tool-verified, [recall] unverified
   - Deduplicate against the OUTPUT_PATH content you already hold from step (a)
   - Edit OUTPUT_PATH to append the graded findings to the appropriate sections (use Edit for targeted inserts; Write only if the file needs a full rewrite)
   - Do NOT dispatch another collector in this turn. Integration is the only tool work in this turn.

(f) **Verify the write.** Read OUTPUT_PATH again and confirm the new section landed correctly. Note any fresh gaps this collector surfaced -- they feed step (b) of iteration i+1.

(g) **Mark TaskCreate entry i as completed** via TaskUpdate. This and the verification Read in step (f) may be in the same turn.

(h) Only NOW may you begin iteration i+1 at step (a). Do not skip ahead.

END FOR

**Anti-batching checklist -- if ANY of these is true, you violated the execution rule and must correct course:**

| Symptom | Fix |
|---|---|
| Two or more Agent tool calls in one assistant turn | Split -- one per turn |
| Dispatched collector i+1 without an Edit on OUTPUT_PATH since collector i returned | Stop. Integrate collector i first, then resume |
| Skipped the Read at step (a) | Read OUTPUT_PATH now; without it, the next briefing is uninformed |
| Reused step 4a's briefing verbatim without reflecting what's already in the file | Re-read the file, recompute gaps, adjust the briefing |
| Marked multiple TaskCreate entries completed in one turn | Each completion must be tied to the just-finished collector, not batched |
| OUTPUT_PATH was only written once (at the end) rather than N+1 times (scaffold + once per collector) | You batched. Restart the loop with integration between dispatches |

**Absolute prohibitions**:
- Do NOT stop after collector 1 because "the findings look sufficient" -- run all N
- Do NOT merge two planned collectors into one dispatch to save time -- each is separate
- Do NOT skip a collector because earlier ones covered "most" of its sub-questions -- gaps from your own judgment are not a substitute for parallel source coverage
- Do NOT conclude the domain is complete until all N TaskCreate entries for this domain are marked completed

**Pre-finalization gate**: Before moving to Step 4d, verify via TaskList that all N collector tasks for this domain are completed. If any are still in_progress or pending, resume the loop at that iteration.

### 4d: Final pass

**FIRST**: Verify all N collector TaskCreate entries for this domain are marked completed. Use TaskList to check. If ANY are still pending or in_progress, go back to Step 4c and dispatch the missing ones. Do NOT proceed with final pass until every collector in your plan has returned.

After verification:

1. Read the full output file
2. Add `## Gaps and Open Questions` section (sub-questions no collector answered)
3. Add `## References` section (consolidate all sources cited)
4. Quality check: confidence grades on every claim, tables over prose, no AI slop
5. Write the final version to OUTPUT PATH
6. If GoodMem is configured, write ONE memory with key findings
7. Mark the domain's TaskCreate entry as completed
8. Verify the output file: `wc -l <output path>`

## Step 5: Write vault docs (Tier 3) or MOC (all tiers)

### Tier 4-5: Rewrite from scratch dir to vault

For Tier 4-5, managers wrote to the scratch dir. Now move/rewrite to the vault:

1. Read each manager's synthesis file from the scratch dir
2. Write final vault files to the vault path with the same content (the manager already applied the frontmatter and quality bar)
3. Verify each file landed: `wc -l <vault path>/<file>`

### All tiers: Write the MOC

Write `00 - Index.md` at the vault path. Follow this structure:

```markdown
---
goodmem_ingest: true
goodmem_scope: cross-project
type: moc
topic: <topic-keyword>
date: <today YYYY-MM-DD>
updated: <today YYYY-MM-DD>
tags: [<topic>, moc]
status: published
---

# <Topic>

<1-2 sentence overview of what this vault section covers and when it was compiled.>

## Map of this vault section

| # | Note | What's in it | Lines |
|---|---|---|---|
| 00 | [[00 - Index]] | You are here | this |
| 01 | [[01 - <Title>]] | <one-line summary> | <N> |
| ... | ... | ... | ... |

## Key findings

| Finding | Evidence | File |
|---|---|---|
| <most important finding 1> | <source citation> | [[01 - ...]] |
| <most important finding 2> | <source citation> | [[02 - ...]] |
| ... | ... | ... |

## Gaps and open questions

| Gap | Source file | When to fill |
|---|---|---|
| <unanswered question from manager reports> | [[NN - ...]] | <suggested trigger> |

## Cross-references

- [[../existing vault section 1]] -- <why relevant>
- [[../existing vault section 2]] -- <why relevant>

## Session provenance

- Date: <today>
- Tier: <1-5>
- Managers dispatched: <N>
- Total agents (managers + collectors): <N>
- Total output lines: <N>
```

## Step 6: Cleanup + report

Each manager already wrote its domain findings to goodmem directly (one memory per domain with key findings, gaps, and vault file path). You do NOT need to parse their output or write a distilled learning — they handled it.

### Clean up scratch dir (Tier 4+ only)

```bash
rm -rf /tmp/deep-research-<timestamp>
```

### Report to user

Print a final summary:

```
Research complete.

Topic: <topic>
Vault path: <path>
Files: <N> + MOC index
Total lines: <N>
Tier: <1-5> (<N> managers, <N> collectors)

Key findings:
- <finding 1>
- <finding 2>
- <finding 3>
```

Do NOT add a trailing summary or explanation beyond this block.

## Error handling

| Error | Action |
|---|---|
| No topic provided | Ask user and stop |
| goodmem retrieve fails | Continue without prior art (degrade gracefully) |
| Manager agent fails to return | Log the failure, skip the domain, note it as a gap in the MOC |
| Manager output file missing | Log warning, note as gap, continue |
| Vault path permission error | Tell user and stop |
| All managers fail | Tell user the research failed and suggest retrying with --tier 1 |
