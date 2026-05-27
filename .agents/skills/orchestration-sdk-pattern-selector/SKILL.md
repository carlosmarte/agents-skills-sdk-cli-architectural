---
name: orchestration-sdk-pattern-selector
description: Choose the SDK client topology — Object-Oriented (stateful factory-returns-resource-objects) vs Singleton (centralized stateless method calls over data-only models) — and scaffold the chosen shape in Node.js (.mjs) or Python. Default to Singleton + pydantic/zod models for medium-to-large or OpenAPI-generated APIs; reserve OO for small, shallow, highly intuitive APIs. Use when designing a new SDK, wrapping a REST API, picking between resource-object nesting and flat method calls, or refactoring a verbose deeply-nested client. Part of the orchestration-sdk-cli-architect suite.
tier: org
---

# SDK Pattern Selector: Object-Oriented vs Singleton

When wrapping a REST API you choose one of two foundational topologies. The choice is permanent in practice — it dictates dependency weight, serialization behavior, bundle size, and how well the SDK survives AI/OpenAPI code generation. This skill makes the decision and scaffolds the result.

## The two patterns

### Object-Oriented (stateful)

A top-level client acts as a **factory** that returns lower-level, stateful resource objects, each carrying its own CRUD methods — like an ORM mapped to the API's resource hierarchy.

```
client.create_user()          → User object
user.create_session()         → Session object
```

- **Strength:** immediate readability; maps to the human mental model of nested resources.
- **Weakness:** degrades badly at scale. Deep routes become verbose fluent chains (`client.locations(id).customers(id).orders(id).get()`). Stateful objects complicate serialization and memory in highly concurrent runtimes. Developers must traverse the whole object graph to touch a leaf.

### Singleton (stateless) — the default

All interaction methods live on a single client class (or a strictly namespaced tree of stateless classes). Resource payloads are **data-only objects** (`pydantic` models / `zod` schemas) used purely for type hints and validation. Transport logic is separated from data schemas.

```
client.call_method(data)      → Data Model (validated, data-only)
```

- **Strength:** no over-instantiation; clean tree-shakeable imports; scales via **namespacing** (`client.users.create(...)`), not deeper nesting.
- **Why it won:** AI codegen platforms (Stainless, Speakeasy) ingest OpenAPI specs and emit Singleton SDKs because operations map to discrete RPC endpoints rather than object lifecycles. Generated `pydantic`/`zod` models enforce runtime validation and cross-language naming parity — solving the "Platform Problem" of divergent SDK behavior across languages.

## Decision matrix

| Factor | Pick OO | Pick Singleton |
|--------|---------|----------------|
| API size | Small, few endpoints | Medium → large enterprise |
| Resource nesting | Shallow | Deep |
| Codegen | Hand-written | OpenAPI / AI-generated |
| Concurrency | Low | High (web servers, async) |
| Bundle-size sensitivity | Low | High (tree-shaking matters) |
| Cross-language parity needed | No | Yes (polyglot twins) |

**Default to Singleton.** Choose OO only when the API is small, shallow, and the readability of resource objects clearly outweighs the scaling cost. When in doubt, Singleton.

## Scaffold: Singleton (Node.js ESM)

```js
// src/sdk/client.mjs
import { z } from "zod";

// Data-only model — validation + types, no behavior.
export const User = z.object({
  id: z.string(),
  email: z.string().email(),
});

export class Client {
  #apiKey;
  #transport;
  constructor({ apiKey, transport }) {        // explicit instance — no global config
    this.#apiKey = apiKey;
    this.#transport = transport;
    this.users = new UsersNamespace(this);    // namespacing, not nesting
  }
  _request(op, payload) { /* delegates to core/transport */ }
}

class UsersNamespace {
  #client;
  constructor(client) { this.#client = client; }
  async create(data) {
    const parsed = User.partial().parse(data);          // validate at the boundary
    return User.parse(await this.#client._request("users.create", parsed));
  }
}
```

## Scaffold: Singleton (Python 3.11+)

```python
# src/sdk/client.py
from pydantic import BaseModel, EmailStr

class User(BaseModel):          # data-only model
    id: str
    email: EmailStr

class _Users:
    def __init__(self, client: "Client") -> None:
        self._client = client
    def create(self, **data) -> User:
        return User.model_validate(self._client._request("users.create", data))

class Client:
    def __init__(self, *, api_key: str, transport=None) -> None:  # explicit instance
        self._api_key = api_key
        self._transport = transport
        self.users = _Users(self)        # namespacing
    def _request(self, op: str, payload: dict): ...   # delegates to core/transport
```

## Rules

- **Models are data-only.** No network calls or business logic on `pydantic`/`zod` models — that lives in `src/core` (see [`orchestration-devtool-hexagonal-core`](../orchestration-devtool-hexagonal-core/SKILL.md)).
- **Scale by namespacing**, not deeper object chains. `client.billing.invoices.create(...)` is namespaced singletons, not a factory tree of stateful objects.
- **Validate at the boundary** — parse inbound/outbound payloads through the model so structural drift surfaces immediately. For undocumented/beta fields, expose a raw escape hatch (see [`orchestration-sdk-client-state-isolation`](../orchestration-sdk-client-state-isolation/SKILL.md)).
- The client must be **explicitly instantiated** with its credentials — never read a global config singleton. That concern is owned by [`orchestration-sdk-client-state-isolation`](../orchestration-sdk-client-state-isolation/SKILL.md).

## Hand-off

After picking and scaffolding the topology, continue to [`orchestration-sdk-client-state-isolation`](../orchestration-sdk-client-state-isolation/SKILL.md) for instantiation/context, and place the real logic in the shared core per [`orchestration-devtool-hexagonal-core`](../orchestration-devtool-hexagonal-core/SKILL.md).
