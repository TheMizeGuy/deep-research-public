# Changelog

All notable changes to the `deep-research` plugin will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.1.3] - 2026-07-14

### Changed

- Roster hygiene: moved `agents/research-manager.md` to
  `skills/deep-research/references/research-manager-agent.md`. As a roster agent its full
  CURRENTLY-UNREACHABLE description shipped into every session's agent-type listing for a role
  no code path dispatches — pure context cost. Spec preserved verbatim minus agent frontmatter;
  README references repointed. No behavior change (the agent was never dispatched by design).

## [0.1.2] - 2026-07-06

### Fixed

- `SKILL.md` frontmatter `allowed-tools` now includes `Edit` and `TaskList` — steps 4c/4e/5
  edit the output file and the pre-finalization gates check the task list, but neither tool
  was in the allowlist. Dropped the unused `TodoWrite` (the body only uses
  TaskCreate/TaskUpdate/TaskList).
- Restored the aggregate fan-out sign-off gate as Step 3b.2. The README and Troubleshooting
  sections documented a >20-collector sign-off stop, but the step itself was missing from
  `SKILL.md`.
- Removed the dead scratch-directory feature: step 3d created `/tmp/deep-research-<ts>` for
  "managers" that no longer exist, step 6 deleted it, and nothing ever wrote to it. Steps
  renumbered (3e→3d, 3f→3e; cross-references updated in `moc-template.md`); tier-table rows
  no longer promise "intermediate synthesis files" or "multiple managers".
- `agents/data-collector.md` recency flags no longer hardcode a model cutoff date
  ("pre-May-2025") — reworded to the model-agnostic "model recall — not fetched this run".

### Changed

- `plugin.json` and `marketplace.json`: descriptions no longer call the two-layer
  architecture a "5-tier hierarchy" and no longer claim a solo "single-agent pass" — tier 1
  floors at two collectors; the accurate phrasing is five tiers of collector fan-out.
  `plugin.json` gained `homepage`, `repository`, and `license` fields.
- Tier-table Behavior wording harmonized across `SKILL.md` and `README.md`.

### Added

- `README.md`: "When to use it (and when not)" section.

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

[0.1.2]: https://github.com/TheMizeGuy/deep-research-public/compare/v0.1.1...v0.1.2
[0.1.1]: https://github.com/TheMizeGuy/deep-research-public/compare/v0.1.0...v0.1.1
[0.1.0]: https://github.com/TheMizeGuy/deep-research-public/releases/tag/v0.1.0
