---
name: tool-bridge
description: Discover and invoke self-described tools through a Tool Bridge gateway using the shortest safe tb CLI path, and use operational feedback when it materially affects a call. Use when an agent needs to find an available organizational tool, inspect an HTBP/MCP/HTTP capability, query connected context, call a gateway tool, reach a device command that may be offline via durable mailbox delivery, explore the visible tool tree, or troubleshoot abnormal tool behavior. Requires an authenticated Tool Bridge target.
---

# Tool Bridge

Use the gateway's live descriptions as the source of truth. Never guess a path, tool name, argument schema, or capability from memory.

## Safety boundaries

- Prefer the `tb` CLI. Read [references/cli-reference.md](references/cli-reference.md) only when target configuration or command syntax is unclear, a Context write/upload is needed, or a command fails.
- Keep the secret key out of prompts, logs, command arguments, source files, and generated artifacts. Use an existing `tb login` profile or secret-injected `TB_SK` environment variable.
- Use the least-privileged identity already provided for the task. Do not request an admin key merely because a path is hidden.
- Treat `effect: write`, `effect: destructive`, and `confirm: true` as external mutations. Obtain explicit user confirmation unless the user already requested that exact mutation.
- Do not register providers, mount nodes, create keys, change secrets, or administer the gateway unless the user explicitly asks for that management action.
- Interpret `404` as either nonexistent or invisible. Do not probe around it to infer hidden paths.
- Do not automatically retry calls that may have side effects. Check the error's `retryable` signal and the command effect first.
- Treat a mailbox operation ending in `result_unknown`, or `expired` with `executionMayHaveOccurred: true`, as execution ambiguity: the device may have run the command. Do not re-call, re-enqueue, or retry; create a new execution only when business idempotency is independently proven and the user authorizes it.
- Delivery mode never relaxes authorization: `effect`, `confirm`, and the user's confirmation boundary apply identically to realtime, mailbox, and fallback calls.
- Never publish credentials, customer data, private payloads, or unverified speculation as feedback. Prefer voting on an existing matching entry over creating a duplicate.

## Choose the shortest safe path

Reuse a target, full command path, and schema already verified during the current task while the selected profile/BaseURL, identity, and runtime contract remain unchanged. Re-verify after any of those changes, or when the gateway reports that the path or arguments are invalid.

### Fast path: known read-only command

Verify a target once per target/session, not before every call:

```sh
tb whoami --json
```

If the exact full command path, arguments schema, `effect: read`, and `confirm: false` are already known from the current runtime, call it directly. Do not add search, help, or feedback requests merely as ceremony:

```sh
tb call '<node>/<command>' --args '<json-object>' --json
```

For a known device command that must also work while the device is offline, the fast path is still a single call: add `--delivery fallback` (see the device delivery section) instead of adding discovery or a second request.

If `tb` is missing, tell the user that Node.js 22+ and `@tool-bridge/cli` are required. Ask before installing a global package. If authentication fails, ask the user to configure a profile or inject `TB_BASE_URL` and `TB_SK`; do not ask them to paste a secret into chat when a secret-input mechanism is available.

### Discovery path: unknown capability or contract

Start with search when the desired capability is known:

```sh
tb search '<capability in a few keywords>' --json
```

If search is unavailable, or the task is exploratory, browse progressively:

```sh
tb tree --depth 2 --json
tb ls '<path>' --json
tb help '<path>' --json
```

JSON search results already carry `items[].tool.inputSchema`, `effect`, `confirm`, and — for device commands — `delivery`. If one result is unambiguous and contains enough information for a read-only call, use its exact `<items[].path>/<items[].tool.name>` pair and call it without another help request.

Open command-level help only when a required field is missing, results are ambiguous, the command is unfamiliar or failure-prone, or the operation may mutate state:

```sh
tb help '<node>/<command>' --json
```

Use `cmds[].path`, `inputSchema`, `effect`, `confirm`, and `scope` from live help. Satisfy the schema exactly and ignore unknown optional fields for forward compatibility. If help already embeds feedback that is clearly relevant, fetch only the entry needed to decide the call:

```sh
tb feedback get '<exact-tool-or-node-path>' '<feedback-id>' --json
```

For an unfamiliar or failure-prone path, feedback can be checked before calling:

```sh
tb feedback ls '<exact-tool-or-node-path>' --json
```

Do not list feedback on every normal read call, and do not fetch every visible entry. Feedback is operational experience, not a replacement for live schema.

For any write, destructive, or `confirm: true` operation, inspect command help and apply the user's authorization boundary before calling.

### Device delivery: one call, no second enqueue

Device commands carry a `delivery` capability in live metadata: `realtime` (default when absent), `mailbox` (durable enqueue only), or `both`. Separately, each call chooses a policy with `--delivery realtime|mailbox|fallback` (default `realtime`). Do not confuse the two: metadata says what the command supports, the flag says what this call wants.

When the full path, schema, and `delivery` are known and the task must tolerate an offline device, make exactly one call:

```sh
tb call '<node>/<command>' --args '<json-object>' --delivery fallback --json
```

The gateway owns the fallback decision. For a `both` command it tries realtime first and enqueues only when it can prove the call was never dispatched; for a mailbox-only command it enqueues directly. A completed realtime result — including a business error from the device — and an outcome-unknown send never enqueue, and outcome-unknown is never retryable. Never follow a failed synchronous call with a second "enqueue" request of your own; that is the gateway's decision, not a client retry strategy.

A `200` response with `x-tb-delivery: realtime` is the command result. A `202` with `x-tb-delivery: mailbox` is an operation identity (`operationId`, state `queued`). When a call returns an operation identity, the synchronous flow is finished: report the queued operation and stop. Do not poll on a fixed interval. Only when the user's current goal actually needs the state or terminal result, read it once:

```sh
tb device op get '<device-id>' '<operation-id>' --json
```

`tb device op ls '<device-id>' --json` lists a device's operations and `tb device op cancel` requests cancellation — cancel of an already-claimed operation is cooperative and does not prove the device stopped. Claiming, renewing, and completing operations belong to the device runtime, never to the calling agent.

Offline presence does not hide a device: an offline device and its mailbox-capable commands stay discoverable, and enqueueing to them is normal.

For any write, destructive, or `confirm: true` operation, delivery mode changes nothing: inspect command help and apply the user's authorization boundary before calling.

### Recovery path: abnormal behavior

Treat errors, timeouts, schema-valid but surprising results, and upstream inconsistencies as abnormal behavior:

1. Preserve the non-sensitive error code, message, path, and relevant conditions.
2. Inspect any `hint` and summarized `feedback` already attached by the failed `tb call`; fetch the single most relevant entry with `tb feedback get`.
3. Run `tb feedback ls` only when the failure carried no useful entry, or before submitting a new entry to avoid duplicates.
4. Try a documented workaround only when it matches live schema and the call is safe to retry. Treat timed-out mutations as outcome unknown; do not retry them.

Keep recovery bounded: normally make at most one workaround retry. Do not claim a workaround is verified until that retry or other evidence confirms it.

Feedback writes are not part of the happy path. After securing the task result, vote for a useful existing entry or submit a verified new lesson only when gateway writes are already authorized and doing so adds value. Otherwise mention a draft only when it would materially help the user.

## Store references in results

A result containing a `store://default/...` URI is an object identity, not content. Read the current owner's object with `tb store stat` / `tb store get`. Create a share only when the user explicitly asks to hand the file to an external audience:

```sh
tb store share 'store://default/<key>' --json
```

Treat the successful share output as a secret being delivered to the user: hand it over immediately and never copy it into logs, feedback, or other diagnostics. Revoke with `tb store revoke-share` when it is no longer needed. Stable `store://` URIs may be persisted; grants, share URLs, and presigned URLs may not.

## Validate and report

Check the returned data against the task, not merely the process exit code. Summarize which gateway path and tool were used, the relevant result, and any limitation or partial failure. Never include the secret key.

Mention feedback only when it changed recovery behavior, was written, or remains a valuable draft requiring authorization.
