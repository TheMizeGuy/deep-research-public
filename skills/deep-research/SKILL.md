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

## Step 4: For each domain -- dispatch collectors, then synthesize

Process each domain ONE AT A TIME (sequential). For each domain, YOU (the skill orchestrator) dispatch the Sonnet collectors directly, then pass their collected findings to an Opus synthesizer.

**Why the skill dispatches collectors, not a manager subagent**: Subagents do NOT reliably receive the Agent tool at runtime (confirmed Claude Code platform limitation -- affects both plugin types AND general-purpose subagents). The main agent (you, running this skill) DOES have the Agent tool. So collector dispatch happens here, at the skill level.

### 4a: Plan collector tasks for this domain

Break the domain's SCOPE into exactly COLLECTOR BUDGET non-overlapping tasks. Each collector covers ~3-5 sub-questions.

| Collector task type | What it does | Best for |
|---|---|---|
| Web research | 3-5 WebSearch queries on specific sub-topics | Current state, blog posts, engineering posts |
| Library docs | context7 resolve + query for specific libraries | API syntax, config options |
| Vault + memory scan | Read existing vault files + goodmem retrieve | Prior learnings, existing reference docs |
| GitHub/community | gh CLI searches, issue scans | Open issues, release notes |
| Academic/specs | WebSearch for arxiv, RFCs, official specs | Foundational concepts |

### 4b: Dispatch collectors ONE AT A TIME

For each planned task, dispatch a Sonnet collector. Collect its output. Then dispatch the next. If an earlier collector reveals gaps, adjust the next collector's briefing.

```
Agent({
  description: "Collect <task type> for <domain name>",
  subagent_type: "deep-research:data-collector",
  model: "sonnet",
  prompt: "You are a DATA COLLECTOR for the deep-research plugin. Execute ONE narrow data-collection task and return structured raw findings.\n\nTASK: <specific collection job>\nSOURCES TO CHECK: <explicit queries, URLs, library IDs, or paths>\nMAX OUTPUT: 2000 words\n\nReturn one H2 section per source with Date, Relevance, and Claims. End with a Collection Summary. Do NOT synthesize across sources. Do NOT write files. Flag any claim from model recall with [recall]."
})
```

Save each collector's returned findings. You will need ALL of them for the synthesizer.

### 4c: Dispatch the Opus synthesizer with collected findings

After ALL collectors for this domain have returned, dispatch an Opus agent to synthesize their findings into the vault file. The synthesizer does NOT need the Agent tool -- it only reads, grades, deduplicates, and writes.

```
Agent({
  description: "Synthesize: <domain name>",
  model: "opus",
  prompt: "<synthesizer prompt below, with all fields filled in>"
})
```

Synthesizer prompt (fill in all fields):

````
You are a RESEARCH SYNTHESIZER. You receive raw findings from data collectors and produce a structured vault reference file.

DOMAIN: <domain name>
OUTPUT PATH: <absolute path>
LINE COUNT TARGET: <800-1500>
WIKILINK SUGGESTIONS: <list>

You have been given findings from <N> data collectors below. Your job:
1. Deduplicate claims across collector outputs
2. Apply confidence grading to every non-trivial claim:
   [P] = primary source (official docs, engineering posts, arxiv)
   [S] = secondary source (blog posts, articles, tutorials)
   [P x N] = N primary sources independently confirm
   [V] = verified (only if YOU can verify via a tool call)
   [recall] = from model training data, unverified
3. Organize by topic following the SCOPE
4. Flag gaps -- sub-questions with no findings
5. Write the synthesis file to OUTPUT PATH
6. If GoodMem is configured, write ONE memory with key findings

Output file format:

---
goodmem_ingest: true
goodmem_scope: cross-project
type: reference
topic: <topic-keyword>
date: <today YYYY-MM-DD>
tags: [<topic>, <sub-topics>]
---

# <DOMAIN title>

<overview>

## <Sub-topic>

<Dense content: tables, code blocks, examples>

[Repeat per sub-topic]

## Gaps and Open Questions

<Unanswered sub-questions>

## References

<All sources cited>

Quality rules:
- Tables over prose
- Every claim gets inline confidence grade
- No AI slop, no emojis, no trailing summary
- Cross-link vault files with wikilinks
- Concrete examples, code blocks
- Absolute ISO dates only

Report back (under 200 words):
FILE: <path>
LINES: <count>
SECTIONS: <H2 list>
SOURCES: <count>
GAPS: <list or "none">

---

COLLECTOR FINDINGS:

<Paste ALL collector outputs here, separated by "--- COLLECTOR N ---" headers>
````

### 4d: After synthesizer returns

1. Mark the domain's TaskCreate entry as completed
2. Verify the output file exists: `wc -l <output path>`
3. Note any reported gaps for the MOC
4. Read the synthesizer's summary

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
