---
name: orchestration-cli-routing-architect
description: Architect a command-line tool's routing, extensibility, and startup performance. Choose directory-based command trees (oclif pattern — filesystem path maps to command) for large/pluggable CLIs vs decorator-based routing (Click/Typer) for small Python utilities, design a first-class plugin model, split the CLI into a ServerClient transport layer and a thin CommandLine orchestration layer, and kill the Node.js startup tax with aggressive lazy-loading. Use when building a CLI, picking a CLI framework, adding plugin support, or fixing slow CLI startup. Part of the orchestration-sdk-cli-architect suite.
tier: org
---

# CLI Routing & Performance Architect

A CLI is a complete terminal application, not a library. Its routing choice locks in the interaction model, the plugin strategy, and the contributor onboarding experience for the tool's entire lifespan. This skill makes that choice and sets up the structural layers and startup-performance discipline.

## Two-layer structure (always)

Regardless of framework, split the CLI internally into two layers:

1. **ServerClient layer** — the internal API for execution. Knows how to reach remote HTTP servers, handles OAuth/bearer auth, and manages exponential backoff on errors. (In a unified tool, this *is* the SDK / shared core — see [`orchestration-devtool-hexagonal-core`](../orchestration-devtool-hexagonal-core/SKILL.md).)
2. **CommandLine layer** — parses arguments, handles streams, generates help text. It is a thin **orchestrator** that calls the ServerClient to do the heavy lifting. No business logic lives here.

## Routing paradigm: directory-based vs decorator-based

### Directory-based command tree (oclif) — default for large CLIs

Command discovery and routing are tied to the **filesystem layout**. Invoking `myapp resources create` routes to `./src/commands/resources/create.ts`. Commands are classes extending a base `Command`; static properties declare flags/args.

- **Pick when:** the CLI is large, spans multiple teams, or needs third-party plugins. Powers Heroku, Salesforce, Shopify, Twilio CLIs.
- **Benefits:** adding a command is a localized file addition — no central registry to edit. Auto-generates help/markdown docs, validates flags, gives type safety. Discovery is predictable from the tree.

```
src/commands/
├── resources/
│   ├── create.ts      # myapp resources create
│   └── list.ts        # myapp resources list
└── auth/
    └── login.ts       # myapp auth login
```

### Decorator-based routing (Click / Typer) — small-to-medium Python utilities

Decorators attach directly to functions to declare commands/options. Reads cleanly; fast to prototype. Underpins the AWS CLI (Click).

- **Pick when:** a Python-focused team building a small-to-medium tool.
- **Limit:** requires explicit command **registration in code**. At hundreds of subcommands across teams, managing central registration becomes a structural tax.
- Alternatives: `docopt` (infers args from POSIX help text), `argparse` (imperative flag config).

| | Directory-based (oclif) | Decorator-based (Click/Typer) |
|---|---|---|
| Routing | Filesystem path | Decorators in code |
| Adding a command | Drop a file | Edit central registration |
| Plugins | First-class | Manual |
| Scale ceiling | Very high | Moderate |
| Best for | Large/multi-team/pluggable | Small/medium Python utils |

## Extensibility: first-class plugins

For platform ecosystems the CLI must be extensible by third parties **without merging into the core repo**. oclif treats plugins as first-class:

```bash
myapp plugins install @company/plugin-name
```

The plugin manages its own npm lifecycle and its commands surface as native first-class commands in the main hierarchy. Frameworks lacking a built-in plugin model force you to hand-build dynamic loading and dependency resolution later — a large retrofit tax. If extensibility is plausible, pick a directory-based framework up front.

## Performance: kill the startup tax

A plain Node.js process pays **200–500 ms** of init before any user code runs. For interactive CLIs that latency is unacceptable. Mitigations:

1. **Lazy-loading (the main lever):** the router statically walks the filesystem and `import()`s *only* the one command file the user invoked — never the whole codebase + all node_modules at boot. Directory-based routing makes this natural.

   ```js
   // Resolve the command path from argv, then dynamic-import only that module:
   const mod = await import(`./commands/${segments.join("/")}.mjs`);
   await mod.run(parsedFlags);
   ```

2. **Fast runtimes:** run via `bun` or `tsx` to bypass standard Node compilation delays for near-instant execution.
3. **Defer heavy imports:** import SDK clients, AWS SDK modules, etc. *inside* the command handler, not at module top-level.

## Rich terminal UIs

When you need multi-select menus, progress bars, or dashboards, keep the framework doing the plumbing (arg parsing, auth, routing, plugins) and delegate the presentation layer to a rendering library like **Ink** (React for the terminal). Don't fold TUI rendering into the routing layer.

## Checklist

- [ ] Two layers: thin CommandLine orchestrator over a ServerClient/shared core.
- [ ] Routing chosen against scale + plugin needs (directory-based for large/pluggable).
- [ ] Plugin model decided up front if third-party extension is plausible.
- [ ] Lazy-load command modules; defer heavy imports into handlers.
- [ ] Command bodies call the shared core, never duplicate SDK logic ([`orchestration-devtool-hexagonal-core`](../orchestration-devtool-hexagonal-core/SKILL.md)).

## Hand-off

The CLI is one **primary driving adapter** over the shared core — the SDK is the other. Wire both per [`orchestration-devtool-hexagonal-core`](../orchestration-devtool-hexagonal-core/SKILL.md). The ServerClient's client semantics follow [`orchestration-sdk-client-state-isolation`](../orchestration-sdk-client-state-isolation/SKILL.md).
