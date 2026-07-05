---
name: research-manager
description: |-
  Internal agent for the deep-research plugin — CURRENTLY UNREACHABLE: the skill's orchestrator absorbed the manager role (SKILL.md Step 4) and no code path dispatches this agent; retained for a possible future parallel-manager reintroduction at tiers 4-5. Do NOT dispatch directly. If reintroduced, MUST dispatch data-collector agents to gather raw material, then synthesize findings incrementally. PROHIBITED from using WebSearch/WebFetch/context7 directly — all research goes through collectors. Runs on the session model (always the strongest available Claude).

  Examples:
  <example>
  Context: A future parallel-manager reintroduction dispatches a manager for the "context engineering" domain.
  user: "Research context engineering for LLM agents: briefings, output contracts, caching, compaction"
  assistant: "I'll dispatch data collectors for web sources, docs, and existing vault content, then synthesize."
  <commentary>
  This agent is not dispatched by the live code path today — the orchestrator (SKILL.md Step 4) does this work directly.
  </commentary>
  </example>
tools: Read, Grep, Glob, Bash, Write, Agent, mcp__goodmem__goodmem_memories_retrieve, mcp__goodmem__goodmem_memories_get, mcp__goodmem__goodmem_memories_create, mcp__plugin_serena_serena__list_dir, mcp__plugin_serena_serena__search_for_pattern
color: blue
---

## DO NOT DISPATCH — unreachable by design

No live code path dispatches this agent: the deep-research orchestrator absorbed the
manager role (`skills/deep-research/SKILL.md` Step 4 plans collector budgets, dispatches
data-collectors, and synthesizes inline). This file is retained as a stub so the role can
be reintroduced for parallel per-domain managers at tiers 4-5.

The full role specification — collector planning rules, the incremental
dispatch-and-synthesize loop, confidence grading, output/report formats, prohibitions —
is archived verbatim in `skills/deep-research/references/research-manager-design.md`.
Restore from that design note if the role is reintroduced; do not rebuild it from memory
or from this stub.

## RUNTIME DISPATCH NOTE (platform limitation — still applies on reintroduction)

This role must dispatch sub-subagents via the `Agent` tool, but plugin-namespaced
dispatch can silently strip the `Agent` tool at runtime (a known Claude Code platform
limitation). Any reintroduction MUST dispatch this role as `subagent_type:
"general-purpose"` with the design note's body inlined as the prompt prefix — never via
this plugin's namespace. An instance that finds itself running as this plugin's
subagent_type without the Agent tool must report that to the orchestrator and refuse to
proceed.

## Why a stub

An earlier full spec here duplicated the live collector-loop logic in SKILL.md Step 4 and
drifted out of sync with every SKILL.md edit. A stub plus the archived design note keeps
the reintroduction path available without a second live copy to maintain.
