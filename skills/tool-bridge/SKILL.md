---
name: tool-bridge
description: Discover and invoke self-described tools through a Tool Bridge gateway using the shortest safe tb CLI path, and use operational feedback when it materially affects a call. Use when an agent needs to find an available organizational tool, inspect an HTBP/MCP/HTTP capability, query connected context, call a gateway tool, explore the visible tool tree, or troubleshoot abnormal tool behavior. Requires an authenticated Tool Bridge target.
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

JSON search results already carry `items[].tool.inputSchema`, `effect`, and `confirm`. If one result is unambiguous and contains enough information for a read-only call, use its exact `<items[].path>/<items[].tool.name>` pair and call it without another help request.

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

### Recovery path: abnormal behavior

Treat errors, timeouts, schema-valid but surprising results, and upstream inconsistencies as abnormal behavior:

1. Preserve the non-sensitive error code, message, path, and relevant conditions.
2. Inspect any `hint` and summarized `feedback` already attached by the failed `tb call`; fetch the single most relevant entry with `tb feedback get`.
3. Run `tb feedback ls` only when the failure carried no useful entry, or before submitting a new entry to avoid duplicates.
4. Try a documented workaround only when it matches live schema and the call is safe to retry. Treat timed-out mutations as outcome unknown; do not retry them.

Keep recovery bounded: normally make at most one workaround retry. Do not claim a workaround is verified until that retry or other evidence confirms it.

Feedback writes are not part of the happy path. After securing the task result, vote for a useful existing entry or submit a verified new lesson only when gateway writes are already authorized and doing so adds value. Otherwise mention a draft only when it would materially help the user.

## Validate and report

Check the returned data against the task, not merely the process exit code. Summarize which gateway path and tool were used, the relevant result, and any limitation or partial failure. Never include the secret key.

Mention feedback only when it changed recovery behavior, was written, or remains a valuable draft requiring authorization.
