# agents-skills-sdk-cli-architectural

Agent skills for **architecting, generating, and refactoring SDKs and CLI wrappers** around an existing API or implementation. Distilled from the architecture canon in `.plans/01/` (the SDK/CLI patterns paper and the two topology diagrams), this is a suite of focused, composable skills enforcing one opinionated stance: stateless Singleton clients, explicit instantiation, native context propagation, directory-based CLI routing, and a single Hexagonal core shared by both interfaces.

## The SDK / CLI architect suite

Start at the orchestrator (`orchestration-sdk-cli-architect`); it routes each decision to the focused skill below. All live under `.claude/skills/<name>/SKILL.md` and are scoped `tier: org`.

| Skill                                                                                                          | Owns                                                                                                                       | Source (paper)       |
| -------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------- | -------------------- |
| [`orchestration-sdk-cli-architect`](.claude/skills/orchestration-sdk-cli-architect/SKILL.md)                   | Orchestrator: analyze → scaffold → implement → test → ship. Routes to the rest.                                            | README + whole paper |
| [`orchestration-sdk-pattern-selector`](.claude/skills/orchestration-sdk-pattern-selector/SKILL.md)             | Object-Oriented vs Singleton topology; stateless methods + `zod`/`pydantic` data models.                                   | Part I               |
| [`orchestration-sdk-client-state-isolation`](.claude/skills/orchestration-sdk-client-state-isolation/SKILL.md) | Explicit clients (no global config singleton); `ContextVars`/`AsyncLocalStorage`; multi-tenancy; raw-payload escape hatch. | Parts II–III         |
| [`orchestration-cli-routing-architect`](.claude/skills/orchestration-cli-routing-architect/SKILL.md)           | Directory-based (oclif) vs decorator (Click/Typer) routing; plugins; lazy-loading the startup tax.                         | Part IV              |
| [`orchestration-devtool-hexagonal-core`](.claude/skills/orchestration-devtool-hexagonal-core/SKILL.md)         | Ports & Adapters; SDK + CLI as primary driving adapters over one core; monorepo topology.                                  | Part VI              |
| [`orchestration-devtool-test-harness`](.claude/skills/orchestration-devtool-test-harness/SKILL.md)             | Boundary mocking (moto / aws-sdk-client-mock); time-skipping (Temporal); model-native eval.                                | Part V               |

For the broader vocabulary (the 16 SDK paradigms × 5 layers), these skills defer to the user-global `sdk-paradigms` skill — this suite is the action-oriented overlay.

## When the suite fires

- Building a new SDK, a CLI tool, or both over a REST API / local implementation.
- Refactoring a wrapper with scaling limits, hidden global state, divergent CLI-vs-SDK behavior, or dependency bloat.

It deliberately does **not** apply to single-file throwaway scripts with no versioning or multi-tenancy.

## Install

### Per skill — `npx skills add`

Install any single skill into Claude Code:

```bash
npx skills add carlosmarte/agents-skills-sdk-cli-architectural \
  --skill orchestration-sdk-cli-architect -a claude-code \
  --skill orchestration-sdk-pattern-selector -a claude-code \
  --skill orchestration-sdk-client-state-isolation -a claude-code \
  --skill orchestration-cli-routing-architect -a claude-code \
  --skill orchestration-devtool-hexagonal-core -a claude-code \
  --skill orchestration-devtool-test-harness -a claude-code
```

Swap `--skill` for any name from the table above.

## Layout

Skills live under `.agents/skills/<name>/` (the source of truth). The Claude Code harness auto-discovers skills from `.claude/skills/`, so each one is mirrored there as a relative symlink (`.claude/skills/<name> -> ../../.agents/skills/<name>`).

```
.agents/skills/                                       # source of truth
├── orchestration-sdk-cli-architect/SKILL.md          # entry point / router
├── orchestration-sdk-pattern-selector/SKILL.md
├── orchestration-sdk-client-state-isolation/SKILL.md
├── orchestration-cli-routing-architect/SKILL.md
├── orchestration-devtool-hexagonal-core/SKILL.md
├── orchestration-devtool-test-harness/SKILL.md
└── agent-skill-kit-*/ , skill-repo-readme/ , ...     # vendored toolkit skills (see skills-lock.json)

.claude/skills/                                       # symlinks → ../../.agents/skills/<name> (harness discovery)
```

The vendored `agent-skill-kit-*` skills (tracked in `skills-lock.json`) provide the `skills-ref` validator/linter used to QA these definitions. They are git-ignored at both their `.agents/skills/` source and `.claude/skills/` symlink.
