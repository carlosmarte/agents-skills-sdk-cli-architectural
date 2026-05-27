---
name: orchestration-sdk-client-state-isolation
description: Eliminate global configuration singletons and hidden static state from an SDK. Mandate explicit client instances (Client(api_key=...) / new Client({apiKey})) for multi-tenancy and safe concurrency, propagate per-request context with native primitives (ContextVars in Python, AsyncLocalStorage in Node.js) instead of a legacy global Hub/Scope, and expose a raw-payload escape hatch for undocumented fields. Use when building an SDK client, refactoring a wrapper that leaks state between concurrent requests or tenants, removing a global API-key singleton, or wiring distributed-tracing/telemetry context. Part of the orchestration-sdk-cli-architect suite.
tier: org
---

# SDK Client Instantiation & State Isolation

A correctly designed SDK has **no hidden global state**. Its behavior is fully determined by the arguments passed to an explicitly instantiated client, and per-request context rides on native language primitives — not a process-global Hub. This skill covers three linked concerns: explicit clients, native context propagation, and the raw-payload escape hatch.

## 1. Explicit clients, not global config singletons

### The anti-pattern

Legacy SDKs mandated a global API key (`stripe.api_key = "..."`) that the library implicitly read on every call. This binds the entire process to one tenant and creates hidden static state, so behavior depends on import-time side effects rather than call arguments. It **fails completely** in a multi-tenant server where one process serves many credentials concurrently.

### The fix: an explicit client facade

A single, explicit client class is the sole entry point. Every API interaction goes through an instance.

```python
# Python — explicit instance, multi-tenant safe
client_a = Client(api_key=tenant_a_key)
client_b = Client(api_key=tenant_b_key)   # different tenant, same process, zero bleed
```

```js
// Node ESM — explicit instance
const clientA = new Client({ apiKey: tenantAKey });
const clientB = new Client({ apiKey: tenantBKey });
```

This buys three things:

1. **Multi-tenancy** — many instances with different credentials live in one process.
2. **No hidden state** — behavior is predictable from constructor args alone.
3. **Trivial dependency injection** — pass a mocked client as a variable in tests instead of monkey-patching static class methods (see [`orchestration-devtool-test-harness`](../orchestration-devtool-test-harness/SKILL.md)).

> Real-world anchor: Stripe migrated to `StripeClient` (Python v15, Node v21) precisely to support multi-tenancy and kill the global-config anti-pattern.

**Deployment rule:** clients holding sensitive credentials (payments, secrets) stay **server-side**. Never embed such an SDK directly in a browser extension or client-side bundle — proxy through a secured backend.

## 2. Native context propagation, not a global Hub

SDKs that carry contextual state across async boundaries — user identity, tracing breadcrumbs, request metadata, telemetry tags — historically used a stack-based **Hub & Scope** model (Sentry-style). In a single-threaded async runtime this often degraded to a global singleton, letting one request's state **bleed into another**.

Modern design deprecates the Hub in favor of native primitives that isolate context automatically, aligned with OpenTelemetry's Global / Isolation / Current scope model.

### Python — `ContextVars`

```python
# src/core/context.py
from contextvars import ContextVar

_request_ctx: ContextVar[dict] = ContextVar("request_ctx", default={})

def set_context(**tags) -> None:
    _request_ctx.set({**_request_ctx.get(), **tags})

def get_context() -> dict:
    return _request_ctx.get()
```

`ContextVar` values are isolated per-task/thread automatically — no manual `push_scope()` / `pop_scope()`. A child task inherits a copy, so concurrent requests never share mutable state.

### Node.js — `AsyncLocalStorage`

```js
// src/core/context.mjs
import { AsyncLocalStorage } from "node:async_hooks";

export const requestContext = new AsyncLocalStorage();

// At the request boundary (e.g. an HTTP handler):
export function withRequestContext(ctx, fn) {
  return requestContext.run(ctx, fn);   // everything inside sees isolated `ctx`
}

// Anywhere downstream, sync or async:
export const getContext = () => requestContext.getStore() ?? {};
```

`AsyncLocalStorage` threads the store through the async call graph without globals, so breadcrumbs and tags stay isolated per request even on a single-threaded event loop.

### Why this matters

- Eliminates manual scope push/pop and its performance overhead.
- Removes the chief source of state leakage in high-concurrency async servers.
- Aligns with OpenTelemetry so tracing context composes with the rest of the observability stack.

## 3. The raw-payload escape hatch (handling structural drift)

Strictly-typed models drop unknown fields on deserialization, blocking access to new/beta API fields until the SDK is updated. Expose the raw payload alongside the typed model so callers can read undocumented fields without breaking strict typing.

```python
class Response(BaseModel):
    id: str
    model_config = {"extra": "allow"}     # retain unknown fields
    @property
    def raw(self) -> dict:
        return self.model_dump()           # full payload incl. undocumented keys
```

```js
class Response {
  constructor(raw) {
    this.id = raw.id;
    this.#raw = raw;
  }
  #raw;
  getRawJsonObject() { return this.#raw; }   // read beta/undocumented fields
}
```

## Refactor checklist

When hardening an existing wrapper, work this list:

- [ ] Remove every global/module-level credential or config singleton; require it in the constructor.
- [ ] Confirm two instances with different credentials can coexist in one process with no shared mutable state.
- [ ] Replace any global Hub/Scope or thread-global with `ContextVars` (py) / `AsyncLocalStorage` (mjs).
- [ ] Set per-request context once at the boundary; never push/pop manually downstream.
- [ ] Add a raw-payload accessor so undocumented fields are reachable.
- [ ] Verify the client is injectable as a plain variable for test mocking.

## Hand-off

The client topology (Singleton vs OO) is chosen in [`orchestration-sdk-pattern-selector`](../orchestration-sdk-pattern-selector/SKILL.md). The actual operations these clients call belong in the shared core — see [`orchestration-devtool-hexagonal-core`](../orchestration-devtool-hexagonal-core/SKILL.md). Test the isolation guarantees with the harnesses in [`orchestration-devtool-test-harness`](../orchestration-devtool-test-harness/SKILL.md).
