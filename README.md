# deep-research

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Claude Code](https://img.shields.io/badge/Claude%20Code-plugin-8A2BE2.svg)](https://claude.com/claude-code)
[![Model](https://img.shields.io/badge/model-Opus%204.7-orange.svg)](https://www.anthropic.com/claude)

A [Claude Code](https://claude.com/claude-code) plugin that conducts deep, multi-agent research on any topic and documents findings in an Obsidian vault. Auto-scales from a single Opus 4.7 researcher (narrow topics) to a full 3-tier hierarchy with Opus managers and Opus data collectors (broad topics).

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

| Domains | Tier | Collector floor | Agents |
|---|---|---|---|
| 1 | 1 | 2 | 1 Opus + 2+ Opus collectors |
| 2 | 2 | 3 | 1 Opus + 3+ Opus collectors per domain |
| 3-4 | 3 | 4 | 1 Opus + 4+ Opus collectors per domain |
| 5-7 | 4 | 6 | Multiple Opus managers, each with 6+ Opus collectors |
| 8+ | 5 | 8 | Large-scale multi-manager, 8+ Opus collectors each |

Collector budget scales with scope: `max(floor, ceil(questions/4))`, capped at 10 per manager. Every tier uses the same Opus manager that MUST dispatch Opus collectors. There is no solo/direct-research path.

### Structural enforcement

The manager agent (`research-manager.md`) does **not have** WebSearch, WebFetch, or context7 tools. It literally cannot do the research itself -- it must dispatch data-collector agents. This is a structural fix for the behavioral bypass where LLMs skip delegation and research directly.

## Installation

```bash
# 1. Add this repo as a marketplace
claude plugin marketplace add https://github.com/TheMizeGuy/deep-research-public.git

# 2. Install the plugin
claude plugin install deep-research@deep-research

# 3. Restart Claude Code for the plugin to load
```

After restart, verify with `claude plugin list`.

## Usage

| Invocation | What it does |
|---|---|
| `/deep-research React Server Components` | Auto-detect tier, research, write vault docs |
| `/deep-research "context engineering for LLM agents" --tier 3` | Force Tier 3 (multi-manager) |
| `/deep-research PostgreSQL LISTEN/NOTIFY --path ~/vault/PostgreSQL/` | Write to specific vault path |
| `/deep-research Tailwind CSS v4 migration` | Auto-classifies as library, writes to Libraries/ |

Natural-language triggers: "deep research on X", "research everything about X", "build a knowledge base on X", "create a reference on X".

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
| Skill | `deep-research` | Entry point; parses args, runs recon, decomposes topic, dispatches managers, writes MOC |
| Agent | `research-manager` | Tier 2/3 Opus manager; dispatches Opus collectors, synthesizes findings. No direct research tools |
| Agent | `data-collector` | Opus 4.7 collector; executes one narrow research task, returns structured raw findings with high-quality relevance filtering |

## Optional enhancements

The plugin works with just the built-in tools, but gets richer with:

- **[GoodMem](https://goodmem.ai/) MCP** -- semantic memory search for cross-session learnings; auto-ingest vault files
- **[Context7](https://context7.com/) MCP** -- live library/framework docs (used by Tier 1 solo manager and data collectors)
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

## License

MIT. See [LICENSE](LICENSE).

## Credits

Built by [mize](https://github.com/TheMizeGuy). Backed by the [Claude Code](https://claude.com/claude-code) plugin system and Anthropic's Opus 4.7 model.
