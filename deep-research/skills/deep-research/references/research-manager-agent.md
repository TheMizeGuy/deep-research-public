# research-manager — preserved agent spec (unreachable by design)

> Moved out of `agents/` 2026-07-14 (env-audit M4): as a roster agent its full
> CURRENTLY-UNREACHABLE description shipped into every session's agent-type listing for a
> role no code path dispatches. Restore by re-adding agent frontmatter and moving back to
> `agents/` if parallel per-domain managers return at tiers 4-5. Design doc: research-manager-design.md.

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
