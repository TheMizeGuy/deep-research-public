---
name: deep-research
description: |-
  Conduct deep, multi-agent research on a topic and document findings in an Obsidian vault with optional goodmem ingestion. Auto-scales from single-agent (narrow topics) to full 3-tier hierarchy with Opus managers and Sonnet data collectors (broad topics). Use when the user asks to "deep research", "do comprehensive research on", "research everything about", "build a knowledge base on", or "create a reference on" a topic. Do NOT use for quick factual questions, single lookups, or casual "what is X" queries.
argument-hint: '<topic> [--path <vault-path>] [--tier <1|2|3>]'
allowed-tools: Bash, Read, Write, Grep, Glob, Agent, TodoWrite, TaskCreate, TaskUpdate, WebSearch, WebFetch, mcp__plugin_goodmem_goodmem__goodmem_memories_retrieve, mcp__plugin_goodmem_goodmem__goodmem_memories_get, mcp__plugin_goodmem_goodmem__goodmem_memories_create
---

# Deep Research

You are conducting a deep research run on the user's behalf. Your job is to decompose the topic, dispatch research agents, and produce a permanent Obsidian vault reference. You run fully autonomously — no checkpoints, no pauses. Report results at the end.

## Step 1: Parse arguments

The user passed an argument string. Parse it into:

1. **Topic** — everything before the first `--` flag. Required. If empty, ask the user what to research and stop.
2. **--path** — optional vault path override (absolute). If omitted, auto-detect in Step 3.
3. **--tier** — optional forced tier (1, 2, or 3). If omitted, auto-detect in Step 3.

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

| Domains | Tier | Behavior |
|---|---|---|
| 1-2 | 1 | Single research-manager-solo per domain, no collectors |
| 3-5 | 2 | Single research-manager per domain, each with 2-4 collectors |
| 6+ | 3 | Multiple research-managers, each with 2-4 collectors, intermediate synthesis files |

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

### 3d: Create scratch directory (Tier 3 only)

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

## Step 4: Dispatch managers

For each domain, dispatch a manager agent. Dispatch them ONE AT A TIME (sequential).

Two agent types exist — pick based on tier:

| Tier | Agent definition | Has Agent tool? | Has WebSearch/context7? | Dispatch method |
|---|---|---|---|---|
| 1 | `agents/research-manager-solo.md` | No | Yes — researches directly | General-purpose (needs WebSearch) |
| 2 or 3 | `agents/research-manager.md` | Yes — dispatches collectors | **No** — structurally cannot bypass | `subagent_type: "deep-research:research-manager"` (enforces tool restriction) |

**Tier 1**: dispatch as general-purpose (read agent file, inline in prompt). The solo manager needs WebSearch/context7 which general-purpose provides.

**Tier 2/3**: dispatch via `subagent_type: "deep-research:research-manager"`. This enforces the structural tool restriction — the manager has Agent tool for dispatching collectors but NOT WebSearch/WebFetch/context7, making it impossible to bypass collector dispatch.

### How to dispatch

**Tier 1:**
1. Read `agents/research-manager-solo.md` (everything after the second `---` frontmatter delimiter).
2. Dispatch as general-purpose with the system prompt + briefing:

```
Agent({
  description: "Research: <domain name>",
  model: "opus",
  prompt: "<solo agent system prompt>\n\n---\n\nBRIEFING:\n<briefing fields>"
})
```

**Tier 2/3:**
1. Dispatch via plugin agent type with briefing only (the agent gets its system prompt from the definition file):

```
Agent({
  description: "Research: <domain name>",
  subagent_type: "deep-research:research-manager",
  model: "opus",
  prompt: "<briefing fields only — no need to inline system prompt>"
})
```

### Briefing fields (appended after the agent system prompt)

```
DOMAIN: <domain name>
SCOPE:
- <sub-question 1>
- <sub-question 2>
- ... (10-30 sub-questions)

TIER: <1|2|3>
COLLECTOR BUDGET: <0 for Tier 1, 2-4 for Tier 2/3>
OUTPUT PATH: <absolute path — vault path for Tier 1/2, scratch dir for Tier 3>
LINE COUNT TARGET: <800-1500 depending on domain breadth>
WIKILINK SUGGESTIONS: <list of existing vault files to cross-link to>

FRONTMATTER TEMPLATE:
---
goodmem_ingest: true
goodmem_scope: cross-project
type: reference
topic: <topic-keyword derived from domain>
date: <today YYYY-MM-DD>
tags: [<topic>, <sub-topics>]
---

QUALITY BAR:
- Tables over prose
- Every non-trivial claim gets an inline confidence grade: [P] primary, [S] secondary, [P x N] cross-verified, [V] verified by tool, [recall] unverified
- No AI slop, no emojis, no trailing summary
- Cross-link to existing vault files using wikilinks
- Include concrete examples, code blocks, tool-call syntax where relevant
- Use absolute ISO dates, never relative
```

### After each manager returns

1. Mark its TaskCreate entry as completed via TaskUpdate
2. Verify the output file exists and has content: `wc -l <output path>`
3. **Tier 2/3 collector verification**: Check the manager's report for "COLLECTORS DISPATCHED: N". If N = 0 or the field is missing, the manager failed to dispatch collectors. Re-dispatch the same domain with the same agent file and an explicit prefix in the briefing: "CRITICAL: Your predecessor failed to dispatch collectors. You MUST dispatch data-collector agents via the Agent tool BEFORE anything else."
4. If the manager reported gaps, note them for the MOC's "gaps" section
5. Read the manager's summary (not the full file — it's in the Agent return)

## Step 5: Write vault docs (Tier 3) or MOC (all tiers)

### Tier 3: Rewrite from scratch dir to vault

For Tier 3, managers wrote to the scratch dir. Now move/rewrite to the vault:

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
- Tier: <1|2|3>
- Managers dispatched: <N>
- Total agents (managers + collectors): <N>
- Total output lines: <N>
```

## Step 6: Memory integration + cleanup

### Write distilled learning (if GoodMem configured)

Write ONE learning to goodmem summarizing what was produced:

```
goodmem_memories_create({
  space_id: "<learnings-space-id>",
  content_type: "text/markdown",
  original_content: "# <Topic> vault section: <N>-file reference at vault/<path>/\n\n## Context\n<1-2 sentences on what was built>\n\n## Vault map\n<table of files + line counts>\n\n## How to use\nRead 00 - Index.md first.\n\n## Complements\n<list of related vault sections>",
  metadata: {"type": "reference", "topic": "<topic-keyword>", "date": "<today>"}
})
```

### Clean up scratch dir (Tier 3 only)

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
Tier: <1|2|3> (<N> managers, <N> collectors)

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
