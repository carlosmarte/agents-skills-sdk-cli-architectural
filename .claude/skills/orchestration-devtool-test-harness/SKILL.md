---
name: orchestration-devtool-test-harness
description: Scaffold the right test harness for an SDK or CLI by matching it to the boundary being isolated. Boundary mocking for cloud/network calls — moto (@mock_aws) in Python, aws-sdk-client-mock in Node.js — to exercise real retry/error paths without live infrastructure; time-skipping for long-running workflows via Temporal's TestWorkflowEnvironment (auto + manual clock advance, activity mocking, replayer determinism checks); and model-native harnesses (Promptfoo / trace evaluation) for non-deterministic agentic SDKs. Use when testing an SDK/CLI, mocking AWS or HTTP at the network boundary, validating retries/timeouts/workflows, or evaluating an AI agent's behavior. Part of the orchestration-sdk-cli-architect suite.
tier: org
---

# Dev-Tool Test Harness

SDKs and CLIs talk to volatile networks, long-running processes, and (increasingly) non-deterministic models. A unit test alone can't validate them. The skill is to **identify the boundary** and pick the harness built for it. Test the shared core once and both adapters are covered (see [`orchestration-devtool-hexagonal-core`](../orchestration-devtool-hexagonal-core/SKILL.md)).

## Pick by boundary

| Boundary | Harness | Mechanism |
|----------|---------|-----------|
| Network / cloud calls | moto (py) · aws-sdk-client-mock (node) | Intercept SDK calls at the API boundary; in-memory simulation |
| Chronological / long-running workflow | Temporal `TestWorkflowEnvironment` | In-memory server, time-skipping |
| Non-deterministic model output | Promptfoo / trace evaluation | Evaluate execution traces, not return values |

## 1. Boundary mocking — cloud & network

Never test against live cloud infra: it adds latency, cost, and non-determinism, and risks mutating real resources. Mock precisely at the **network boundary** so production code paths (retries, error handling) still execute.

### Python — `moto`

`moto` intercepts `boto3` calls at the API level and routes them to an in-memory AWS simulation — not just patched return values, so real Boto3 code paths run.

```python
import boto3
from moto import mock_aws

@mock_aws
def test_upload_creates_object():
    s3 = boto3.client("s3", region_name="us-east-1")
    s3.create_bucket(Bucket="my-bucket")          # emulated in memory
    s3.put_object(Bucket="my-bucket", Key="k", Body=b"data")
    assert s3.get_object(Bucket="my-bucket", Key="k")["Body"].read() == b"data"
```

No Docker images or live AWS config — the bucket lives in local memory for the test lifecycle.

### Node.js — `aws-sdk-client-mock`

AWS SDK v3 is command-centric, so standard mocking fails. `aws-sdk-client-mock` intercepts the SDK's internal command dispatcher, letting you script granular responses or error states to validate retry/failover logic.

```js
import { mockClient } from "aws-sdk-client-mock";
import { S3Client, GetObjectCommand } from "@aws-sdk/client-s3";

const s3Mock = mockClient(S3Client);
s3Mock.on(GetObjectCommand).resolvesOnce({ Body: "ok" })
      .rejects(new Error("ServiceUnavailable"));   // drive the retry path
```

For a generic HTTP-based SDK (no AWS), mock at the transport/`fetch` boundary the same way — and prefer injecting a mocked client instance (the explicit-client pattern from [`orchestration-sdk-client-state-isolation`](../orchestration-sdk-client-state-isolation/SKILL.md)) over monkey-patching.

## 2. Time-skipping — long-running workflows

When the SDK orchestrates durable, long-running logic (background jobs, scheduled work, resilient workflows), the harness must collapse chronological time. Temporal's `TestWorkflowEnvironment` runs a lightweight in-memory server beside the test suite.

- **Automatic time skipping:** programmatic timers, sleeps, and timeouts fast-forward instantly; the skip pauses only while a deterministic activity is actually executing.
- **Manual control:** `await env.sleep(seconds)` advances the global clock to validate delayed callbacks, timeout thresholds, and exponential-backoff intervals precisely.
- **Activity mocking:** inside an isolated `ActivityEnvironment`, inject mocked network-heavy activities and capture heartbeats (`env.on_heartbeat`) — validates topology without paying for the real activity.
- **Replayer:** ingest production JSON Event Histories and replay them locally. If a code change violates determinism (e.g. branches logic unexpectedly), the replay fails — catching a catastrophic deploy before it ships.

```python
async def test_workflow_completes_quickly():
    async with await WorkflowEnvironment.start_time_skipping() as env:
        # a workflow that "sleeps" 30 days resolves instantly under time-skip
        result = await env.client.execute_workflow(MyWorkflow.run, id="t", task_queue="q")
        assert result.status == "done"
```

## 3. Model-native harnesses — agentic SDKs

For Agent SDKs, `assert response == expected` fails — output is non-deterministic. Model-native harnesses (Promptfoo, OpenAI Agents SDK) evaluate the **execution trace** against multidimensional criteria instead of strict return values:

- Outcome accuracy, process/reasoning logic, style adherence, token-consumption efficiency.
- The harness is an orchestration layer: spin up isolated per-issue sandboxed workspaces, capture the agentic stack trace, and route outputs through deterministic assertions or secondary LLM-based graders.
- AgentBench-style multi-environment benchmarks evaluate behavior in isolated OS/DB environments.
- For multi-agent systems, the Codex CLI is exposed as an MCP server so agents run scoped shell tasks while the harness observes the full trace for auditability.

## Workflow

1. **Classify the boundary** the code crosses (network, time, model). Most SDKs/CLIs hit the network boundary — start there.
2. **Scaffold the matching harness** from the snippets above in `tests/`.
3. **Assert against the boundary behavior** — retries fire, backoff timing is right, undocumented fields survive, the workflow stays deterministic.
4. **Inject mocked clients** rather than patching statics — the explicit-client pattern makes this trivial.

## Hand-off

These harnesses test the shared core defined in [`orchestration-devtool-hexagonal-core`](../orchestration-devtool-hexagonal-core/SKILL.md). Mockability depends on the explicit-client design in [`orchestration-sdk-client-state-isolation`](../orchestration-sdk-client-state-isolation/SKILL.md). Once green, ship with `git` + `gh pr create` (never the MCP PR tool).
