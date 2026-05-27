---
name: orchestration-sdk-cli-architect
description: Orchestrate the design, generation, or refactor of an SDK and/or CLI wrapper around an existing API or implementation. Enforces modern architecture — stateless Singleton clients, explicit client instantiation, native context propagation, directory-based CLI routing, and Hexagonal shared-core logic so the SDK and CLI run identical business logic. Use when the user asks to build a new SDK, build a CLI tool, wrap a REST API, or refactor an existing API wrapper that suffers from scaling, state leakage, or dependency bloat. Routes to the focused sibling skills for each decision. Do NOT use for a single-file throwaway utility script with no versioning or multi-tenant needs.
tier: org
---

# SDK / CLI Architect (Orchestrator)

This is the entry point for building or refactoring a developer tool — an SDK, a CLI, or (most often) both sharing one engine. It sequences the work and delegates each decision to a focused sibling skill. Treat it as a router: read this once, then pull in the specific skill for the decision in front of you.

## When this applies

- **Use when:** creating a new SDK, building a CLI tool, wrapping a REST API or local implementation, or refactoring a wrapper with scaling limits, hidden global state, divergent CLI-vs-SDK behavior, or dependency bloat.
- **Do not use when:** the user wants a one-off, single-file script with no versioning, no multi-tenancy, and no second interface. Hexagonal separation is over-engineering for a throwaway.

## The sibling skill map

Each decision below has a dedicated skill. Invoke it when you reach that step; don't inline its logic here.

| Decision | Skill | What it owns |
|----------|-------|--------------|
| OO vs Singleton; data models | [`orchestration-sdk-pattern-selector`](../orchestration-sdk-pattern-selector/SKILL.md) | Picks the client topology and scaffolds stateless methods + `pydantic`/`zod` models. |
| Explicit clients & context | [`orchestration-sdk-client-state-isolation`](../orchestration-sdk-client-state-isolation/SKILL.md) | Kills global config singletons; wires `ContextVars` / `AsyncLocalStorage`; multi-tenancy; raw-payload escape hatch. |
| CLI routing & startup | [`orchestration-cli-routing-architect`](../orchestration-cli-routing-architect/SKILL.md) | Directory-based vs decorator routing, plugins, lazy-loading the startup tax. |
| Shared core | [`orchestration-devtool-hexagonal-core`](../orchestration-devtool-hexagonal-core/SKILL.md) | Ports & adapters; SDK + CLI as primary driving adapters over one core; monorepo topology. |
| Tests | [`orchestration-devtool-test-harness`](../orchestration-devtool-test-harness/SKILL.md) | Boundary mocking (moto / aws-sdk-client-mock), time-skipping (Temporal), model-native eval harnesses. |

Background canon (the 16 paradigms × 5 layers) lives in the `sdk-paradigms` skill — load it when you need the full vocabulary; this orchestrator is the action-oriented overlay.

## Execution workflow

Follow these phases in order. Each phase names the skill that does the heavy lifting.

### 1. Analyze

Establish the inputs before writing anything:

- **Target surface** — an OpenAPI spec, a REST API, or an existing local implementation to wrap.
- **Both interfaces?** Decide whether the deliverable is SDK-only, CLI-only, or both. If both, the core must be shared (phase 4) from the start — retrofitting Hexagonal separation later is expensive.
- **Scale & tenancy** — resource-hierarchy depth, expected number of endpoints, and whether one process serves multiple tenants/credentials concurrently. These drive the pattern choice.
- **Language(s)** — Node.js (ESM `.mjs`), Python 3.11+, or both as polyglot twins. The sibling skills give idioms for each.

Output of this phase: a one-paragraph architecture brief stating pattern, interfaces, languages, and tenancy model.

### 2. Scaffold

Lay down the Hexagonal directory structure so core logic is physically separated from adapters from commit one:

```
<tool>/
├── src/
│   ├── core/        # business logic + ports (interfaces). Knows nothing about HTTP, CLI, or files.
│   ├── sdk/         # PRIMARY driving adapter: programmatic client facade
│   ├── cli/         # PRIMARY driving adapter: argument parser + command tree
│   └── adapters/    # SECONDARY driven adapters: REST clients, file system, DB
└── tests/
```

Use [`orchestration-devtool-hexagonal-core`](../orchestration-devtool-hexagonal-core/SKILL.md) to get the ports/adapters wiring and the monorepo/workspace layout right.

### 3. Implement

- **SDK client topology** → [`orchestration-sdk-pattern-selector`](../orchestration-sdk-pattern-selector/SKILL.md). Default to stateless Singleton + data-only models for medium-to-large APIs.
- **State & instantiation** → [`orchestration-sdk-client-state-isolation`](../orchestration-sdk-client-state-isolation/SKILL.md). Explicit `Client(api_key=...)` instances, native context propagation, no global Hub.
- **CLI routing** → [`orchestration-cli-routing-architect`](../orchestration-cli-routing-architect/SKILL.md). Directory-based command tree for large CLIs; lazy-load to kill startup latency.
- Keep the CLI command bodies and SDK methods thin — both call into `src/core`.

### 4. Test

Scaffold harnesses that isolate the network/time/non-determinism boundary → [`orchestration-devtool-test-harness`](../orchestration-devtool-test-harness/SKILL.md). Match the harness to the boundary: `moto`/`aws-sdk-client-mock` for cloud calls, Temporal's `TestWorkflowEnvironment` for long-running workflows, model-native trace evaluation for agentic SDKs.

### 5. Commit & PR

Commit and open the PR strictly with `git` + `gh pr create`. Never use the MCP `github-create_pull_request` tool. Only commit/push when the user has asked you to.

## Non-negotiable defaults

These are the architectural commitments this suite enforces. When the user's instinct conflicts, surface the trade-off rather than silently complying.

1. **Stateless Singleton over stateful OO** for medium-to-large APIs — avoids verbose deep nesting and eases AI/OpenAPI code generation.
2. **Explicit client instances, never a global config singleton** — enables multi-tenancy and safe concurrency.
3. **Native context primitives** (`ContextVars` / `AsyncLocalStorage`) over legacy global Hub/Scope mechanisms.
4. **Directory-based command trees** for large CLIs (oclif pattern) over decorator routing — predictable discovery, first-class plugins.
5. **One shared core** behind both the SDK and CLI adapters — zero logic drift.
6. **`gh` CLI for all contributions** — never the MCP GitHub PR tool.

## Worked example

> "Build a Node.js CLI and SDK for our new payments API."

1. **Analyze:** medium API, multi-tenant (platform serves many merchants), both interfaces, Node ESM.
2. **Scaffold:** Hexagonal tree with `src/core` holding the payment operations.
3. **Implement:** Singleton `PaymentsClient` with stateless methods + `zod` models (`orchestration-sdk-pattern-selector`); explicit `new PaymentsClient({ apiKey })` per tenant with `AsyncLocalStorage` request context (`orchestration-sdk-client-state-isolation`); directory-based `commands/charges/create.ts` tree, lazy-loaded (`orchestration-cli-routing-architect`).
4. **Test:** `aws-sdk-client-mock`-style interception of the HTTP client to validate retry logic (`orchestration-devtool-test-harness`).
5. **Ship:** `gh pr create` (never the MCP PR tool).
