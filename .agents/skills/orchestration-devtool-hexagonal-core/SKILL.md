---
name: orchestration-devtool-hexagonal-core
description: Unify an SDK and a CLI behind one shared core using Hexagonal Architecture (Ports & Adapters), so the programmatic call (myapp.resource.create()) and the terminal command (myapp resource create) run identical business logic, validation, and error handling. Treats both the SDK facade and the CLI parser as primary driving adapters over a core that knows nothing about HTTP, terminals, or files; isolates outbound REST/DB/filesystem behind secondary driven adapters. Covers monorepo/workspace topology (core package imported by a cli app). Use when building both an SDK and a CLI, eliminating logic drift between two interfaces, or structuring a dev-tool monorepo. Part of the orchestration-sdk-cli-architect suite.
tier: org
---

# Hexagonal Core: One Engine, Many Adapters

When a tool ships both an SDK and a CLI, the single biggest failure mode is **logic drift** — the CLI refreshes auth tokens differently than the SDK, or validation rules diverge. Hexagonal Architecture (Ports & Adapters) eliminates this by forcing all interfaces through one core. This skill is the structural backbone of the `orchestration-sdk-cli-architect` suite.

## The principle

The **core business logic is entirely ignorant of its environment.** It must not know whether it was invoked from a web request, a cron job, an SDK call, or a terminal command.

- **Ports** — abstract interfaces the core defines, describing what operations are possible and what it needs from the outside world.
- **Adapters** — concrete implementations that plug into ports.
  - **Primary (driving) adapters** *trigger* the core. In a dev tool, **both the SDK facade and the CLI parser are primary adapters.** Each captures user intent (function args / terminal flags) and translates it into one standardized payload handed to the core.
  - **Secondary (driven) adapters** are *called by* the core for outbound work: REST APIs, databases, the local file system.

Result: `myapp.resource.create(...)` (SDK) and `myapp resource create` (CLI) flow through the **exact same** validation, business rules, and error handling. Parity is structural, not maintained by discipline.

## Directory layout

```
src/
├── core/
│   ├── operations.mjs      # business logic — pure, no HTTP/CLI/fs imports
│   ├── models.mjs          # data-only validation models (zod/pydantic)
│   └── ports.mjs           # interfaces: ApiPort, StoragePort, ...
├── sdk/
│   └── client.mjs          # PRIMARY adapter: programmatic facade → core
├── cli/
│   └── commands/...        # PRIMARY adapter: arg parser → core
└── adapters/
    ├── rest.mjs            # SECONDARY adapter implements ApiPort
    └── filesystem.mjs      # SECONDARY adapter implements StoragePort
```

**Dependency rule:** `core/` imports nothing from `sdk/`, `cli/`, or `adapters/`. Adapters depend on the core, never the reverse.

## Minimal wiring (Node ESM)

```js
// src/core/operations.mjs — knows nothing about transport
export function makeOperations({ apiPort }) {   // ports injected
  return {
    async createResource(input) {
      const data = ResourceInput.parse(input);   // shared validation
      return apiPort.post("/resources", data);     // via a PORT, not fetch()
    },
  };
}
```

```js
// src/sdk/client.mjs — PRIMARY adapter
import { makeOperations } from "../core/operations.mjs";
import { RestAdapter } from "../adapters/rest.mjs";
export class Client {
  constructor({ apiKey }) {
    this.ops = makeOperations({ apiPort: new RestAdapter({ apiKey }) });
  }
  createResource(input) { return this.ops.createResource(input); }  // thin
}
```

```js
// src/cli/commands/resources/create.mjs — PRIMARY adapter
import { makeOperations } from "../../../core/operations.mjs";
import { RestAdapter } from "../../../adapters/rest.mjs";
export async function run(flags) {
  const ops = makeOperations({ apiPort: new RestAdapter({ apiKey: flags.apiKey }) });
  const out = await ops.createResource(flags);     // SAME core call as the SDK
  console.log(JSON.stringify(out, null, 2));
}
```

The CLI command body and the SDK method both reduce to one `ops.createResource(...)` call. Any fix to the core benefits both instantly.

## Anti-patterns to reject

- **Business logic in the adapter.** Don't validate or apply rules inside the CLI command or inside an HTTP handler / MediatR-style intermediary. Logic lives in `core/`.
- **Core importing a transport.** If `core/` imports `fetch`, `httpx`, `fs`, or a CLI lib, the boundary is broken — route it through a port.
- **Two copies of validation.** One model set in `core/`, consumed by both adapters.

## Monorepo / workspace topology

Implement the shared core as its own package consumed by the interface apps:

```
packages/core/        # @tool/core — session mgmt, auth, config, operations, ports
apps/cli/             # imports @tool/core; adds terminal UI, headless mode, shell flows
apps/sdk/  (or root)  # imports @tool/core; the programmatic facade
```

- Mirrors the `@cline/core` + `apps/cli` model: the CLI is an internal workspace app layering terminal concerns *on top of* the shared engine.
- For internal-only shared helpers, relative imports across workspace packages (Windmill-style) avoid the overhead of publishing to npm/PyPI.
- A bug fix or optimization in `core` reaches both the SDK and CLI with no cross-repo sync.

## Canonical proof: Claude Code vs Claude Agent SDK

These run on the **same underlying harness** — identical agent loop, tool-calling, context management. They differ only in the primary driving adapter:

- **CLI adapter (Claude Code):** interactive terminal UI, human-in-the-loop, slash commands, inline diffs, manual permission prompts.
- **SDK adapter (Claude Agent SDK):** no terminal UI; exposes the core as a programmatic library streaming typed messages; permissions resolved via programmatic hooks (`permissionMode` / `bypassPermissions`).

When a core capability is added (e.g. bubblewrap sandboxing), both the interactive CLI and the automated pipeline inherit it simultaneously. That is the payoff of getting this boundary right.

## Hand-off

Client topology for the SDK adapter → [`orchestration-sdk-pattern-selector`](../orchestration-sdk-pattern-selector/SKILL.md); instantiation/context → [`orchestration-sdk-client-state-isolation`](../orchestration-sdk-client-state-isolation/SKILL.md); CLI adapter routing → [`orchestration-cli-routing-architect`](../orchestration-cli-routing-architect/SKILL.md); test the core once and both adapters are covered → [`orchestration-devtool-test-harness`](../orchestration-devtool-test-harness/SKILL.md).
