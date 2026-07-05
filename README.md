# deep-research

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Claude Code](https://img.shields.io/badge/Claude%20Code-plugin-8A2BE2.svg)](https://claude.com/claude-code)

A [Claude Code](https://claude.com/claude-code) plugin that conducts deep, multi-agent research on any topic and documents findings in an Obsidian vault. Auto-scales from a single-agent pass (narrow topics) to a full 5-tier hierarchy of data collectors (broad topics), all running on the session model — whichever Claude tier is currently active, always the strongest available. The plugin never pins a model, so it stays current as Claude models change.

## How it works

```
/deep-research <topic> [--path <vault-path>] [--tier <1|2|3|4|5>]
```

The plugin runs fully autonomously:

1. **Reconnaissance** -- searches existing vault content and GoodMem (if configured) for prior art
2. **Topic decomposition** -- breaks the topic into non-overlapping research domains
3. **Tier selection** -- picks the right scale based on domain count
4. **Agent dispatch** -- sends out researchers sequentially, each producing a structured vault file
5. **MOC generation** -- writes a `00 - Index.md` with findings table, gaps, and cross-references
6. **Memory integration** -- optionally syncs to GoodMem for cross-session retrieval

## Tier system

Every tier runs a manager-role pass per domain (the orchestrator itself absorbs this role — see [Architecture](#architecture)) with mandatory data-collector dispatch. No tier does solo research; even a single domain gets collectors.

| Domains | Tier | Collector floor | Behavior |
|---|---|---|---|
| 1 | 1 | 2 | Single domain |
| 2 | 2 | 3 | One manager-role pass per domain |
| 3-4 | 3 | 4 | One manager-role pass per domain |
| 5-7 | 4 | 6 | More domains, intermediate synthesis files |
| 8+ | 5 | 8 | Large-scale multi-domain, extensive cross-referencing |

Collector budget scales with scope: `max(floor, ceil(questions/4))`, capped at 10 per domain. If the projected total across all domains exceeds 20 collectors, the skill stops and asks for explicit sign-off before dispatching.

## Architecture

```
skills/deep-research/SKILL.md (orchestrator + per-domain manager/synthesizer + documentor)
  -> agents/data-collector.md (session model, narrow data acquisition, one dispatch per turn)
```

The orchestrator (the SKILL.md body, running as the main agent) plans collector budgets, dispatches `data-collector` agents one at a time, and synthesizes their findings into the vault file itself — there is no separate synthesizer subagent. `agents/research-manager.md` is a retained stub: no live code path dispatches it today, but its full role spec is archived at `skills/deep-research/references/research-manager-design.md` in case a future release reintroduces parallel per-domain managers at tiers 4-5.

### Structural enforcement

The archived manager design deliberately withholds WebSearch, WebFetch, and context7 tools from the manager role — it cannot do research itself, only dispatch `data-collector` agents that do. This is a structural fix for the behavioral bypass where LLMs skip delegation and research directly; the constraint carries forward to whichever agent plans and dispatches collectors today (the orchestrator).

## Installation

```bash
# 1. Add this repo as a marketplace
claude plugin marketplace add https://github.com/TheMizeGuy/deep-research-public.git

# 2. Install the plugin
claude plugin install deep-research@deep-research-public

# 3. Restart Claude Code for the plugin to load
```

After restart, verify with `claude plugin list`.

## Quickstart

1. Install the plugin (above) and restart Claude Code.
2. Run `/deep-research <topic>` — or just ask in natural language ("deep research on X", "build a knowledge base on X").
3. The skill searches for prior art, decomposes the topic into domains, auto-picks a tier, and starts dispatching data collectors one at a time. Expect one assistant turn per collector dispatch and one per integration — a Tier 1 topic (2+ collectors) finishes in a handful of turns; Tier 4-5 topics with several domains take longer.
4. When it finishes, it prints a summary (topic, vault path, file count, tier, collector count, key findings) and the vault contains a `00 - Index.md` MOC plus one file per domain.

## Usage

| Invocation | What it does |
|---|---|
| `/deep-research React Server Components` | Auto-detect tier, research, write vault docs |
| `/deep-research "context engineering for LLM agents" --tier 3` | Force Tier 3 |
| `/deep-research PostgreSQL LISTEN/NOTIFY --path ~/vault/PostgreSQL/` | Write to specific vault path |
| `/deep-research Tailwind CSS v4 migration` | Auto-classifies as library, writes to Libraries/ |

Natural-language triggers: "deep research on X", "research everything about X", "build a knowledge base on X", "create a reference on X".

## Walkthrough

A worked example for `/deep-research PostgreSQL partitioning strategies`:

1. **Reconnaissance** — the skill checks GoodMem (if configured) and globs the vault for an existing `PostgreSQL` folder. Suppose nothing exists yet.
2. **Planning** — the topic decomposes into domains, e.g. "Declarative partitioning fundamentals," "Partition pruning and query performance," "Partition maintenance and migrations" (3 domains -> Tier 3, collector floor 4 per domain).
3. **Per-domain loop** — for each domain the skill presents a collector plan table, creates one Task entry per collector, then loops: read the domain file, dispatch collector `i` (one Agent call, one turn), read its returned findings, confidence-grade each claim (`[P]`/`[S]`/`[P x N]`/`[V]`/`[recall]`), and Edit the findings into the file — all before dispatching collector `i+1`.
4. **Per-domain finalization** — after all collectors for a domain return, the skill adds "Gaps and Open Questions" and "References" sections, writes the file, updates the MOC's map row for that domain, and (if GoodMem is configured) force-syncs the file.
5. **Cross-cutting finalization** — once every domain is done, the skill fills the MOC's Key Findings, Gaps, and Cross-references sections and flips its `status:` frontmatter to `published`.
6. **Report** — a final block prints the topic, vault path, file count, total lines, tier, collector count, and 3 key findings.

Output shape for this example: `~/vault/Libraries/PostgreSQL/00 - Index.md` plus `01 - Declarative Partitioning Fundamentals.md`, `02 - Partition Pruning and Query Performance.md`, `03 - Partition Maintenance and Migrations.md`.

## Output

Each research run produces:

- **`00 - Index.md`** -- Map of Contents with findings table, gaps, cross-references, session provenance
- **`01 - <Domain>.md`** through **`NN - <Domain>.md`** -- one file per research domain

Every file includes:
- Obsidian-compatible frontmatter (with `goodmem_ingest: true` for auto-sync if configured)
- Dense content: tables, code blocks, concrete examples
- Inline confidence grading on every non-trivial claim: `[P]` primary, `[S]` secondary, `[P x N]` cross-verified, `[V]` tool-verified, `[recall]` unverified
- Gaps and open questions section
- Full references list

## Components

| Type | Name | Purpose |
|---|---|---|
| Skill | `deep-research` | Entry point and orchestrator; parses args, runs recon, decomposes the topic, computes tier/collector budgets, dispatches collectors, synthesizes findings, writes the MOC |
| Agent | `data-collector` | Runs on the session model; executes one narrow research task per dispatch, returns structured raw findings with confidence-relevant relevance/date metadata, never synthesizes |
| Agent | `research-manager` | Currently unreachable stub — the orchestrator absorbed this role; full spec archived at `skills/deep-research/references/research-manager-design.md` for a possible future parallel-manager reintroduction at tiers 4-5 |

## Optional enhancements

The plugin works with just the built-in tools, but gets richer with:

- **[GoodMem](https://goodmem.ai/) MCP** -- semantic memory search for cross-session learnings; auto-ingest vault files
- **[Context7](https://context7.com/) MCP** -- live library/framework docs (used by data collectors for library-docs tasks)
- **[serena](https://github.com/oraios/serena) MCP** -- symbol-level code navigation for researching codebases

## Confidence grading

Every non-trivial claim in the output is tagged:

| Grade | Meaning |
|---|---|
| `[P]` | Primary source -- official docs, engineering posts, arxiv papers |
| `[S]` | Secondary source -- blog posts, articles, tutorials |
| `[P x N]` | N primary sources independently confirm the claim |
| `[V]` | Verified by tool call (not just reported by a source) |
| `[recall]` | From model training data, not verified via tool -- treat with caution |

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Skill stops after one collector on a multi-collector domain | The model treated early findings as "sufficient" and skipped the rest of the plan | Not a config issue — re-run the request; the skill's instructions require dispatching every planned collector before finalizing a domain |
| `research-manager` never seems to run | Expected — the agent is an unreachable stub | The orchestrator (SKILL.md) absorbed the manager role directly; this is by design, not a bug |
| No GoodMem/Context7/serena results | Those MCPs aren't configured in this Claude Code install | Optional — the skill degrades gracefully to vault-only output with plain WebSearch/WebFetch collectors |
| Research stalls asking for sign-off | Projected collector total across domains exceeds 20 | Expected safety gate — confirm to proceed, or re-run with `--tier` set lower to shrink scope |
| Vault path permission error | The target vault directory isn't writable | Pass an explicit writable `--path`, or fix directory permissions |

## License

MIT. See [LICENSE](LICENSE).

## Credits

Built by [TheMizeGuy](https://github.com/TheMizeGuy). Backed by the [Claude Code](https://claude.com/claude-code) plugin system and the session model — always the strongest available Claude.
