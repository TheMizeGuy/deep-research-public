# Changelog

All notable changes to the `deep-research` plugin will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.1.1] - 2026-07-05

### Changed

- Removed all `model:` pins from agent frontmatter and dispatch examples
  (`deep-research/agents/data-collector.md`, `deep-research/agents/research-manager.md`,
  `deep-research/skills/deep-research/SKILL.md`); agents now inherit the session model
  unconditionally — always the strongest available Claude — instead of a fixed Opus pin. Added an
  "Execution mode" note to `SKILL.md` documenting the inline-vs-dispatch allowance.
- Collapsed the anti-batching rule in `SKILL.md` Step 4c into one canonical statement
  (`RULE ONE-PER-TURN`), cited by name everywhere else it previously repeated in full.
- Reduced `agents/research-manager.md` to a stub: the agent is currently unreachable (the
  `SKILL.md` orchestrator absorbed the manager role), so its full role spec — collector planning,
  the dispatch-and-synthesize loop, output format, prohibitions — moved to
  `skills/deep-research/references/research-manager-design.md` for a possible future
  parallel-manager reintroduction at tiers 4-5, instead of drifting out of sync as a second live copy.
- Externalized the `00 - Index.md` MOC skeleton out of `SKILL.md` Step 3f into
  `skills/deep-research/references/moc-template.md`.
- Added a worked confidence-grading example to `agents/data-collector.md` and `SKILL.md` Step 4c
  showing one claim graded `[P]` from a tool result and reasoning through `[P x N]`/`[V]`/`[recall]`
  upgrades, plus a "Self-check before returning" list on `data-collector.md`.
- `plugin.json` and `.claude-plugin/marketplace.json` author/owner updated to `TheMizeGuy`;
  `plugin.json` author carries a contact email.

### Added

- `README.md`: Quickstart, Walkthrough (a worked PostgreSQL-partitioning example), and
  Troubleshooting sections; a `## Architecture` section with the two-layer dispatch diagram.
- This changelog.

## [0.1.0] - 2026-04-14

### Added

- Initial public release: skill-driven orchestrator (`skills/deep-research/SKILL.md`) that
  decomposes a topic into domains, auto-selects a 1-5 tier based on domain count, and dispatches
  `agents/data-collector.md` agents one at a time per domain with incremental synthesis into
  Obsidian vault files plus a `00 - Index.md` MOC.
- Confidence-grading vocabulary (`[P]`, `[S]`, `[P x N]`, `[V]`, `[recall]`) applied to every
  non-trivial claim in the output.
- MIT license.

[0.1.1]: https://github.com/TheMizeGuy/deep-research-public/compare/v0.1.0...v0.1.1
[0.1.0]: https://github.com/TheMizeGuy/deep-research-public/releases/tag/v0.1.0
