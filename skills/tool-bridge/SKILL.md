---
name: tool-bridge
description: Discover and invoke self-described tools through a Tool Bridge gateway with the tb CLI. Use when an agent needs to find an available organizational tool, inspect an HTBP/MCP/HTTP capability, query connected context, call a gateway tool, explore the visible tool tree, or share reusable tool feedback. Requires an authenticated Tool Bridge target.
---

# Tool Bridge

Use the gateway's live descriptions as the source of truth. Never guess a path, tool name, argument schema, or capability from memory.

## Guardrails

- Prefer the `tb` CLI. Read [references/cli-reference.md](references/cli-reference.md) before the first gateway operation in a session or when a command fails.
- Keep the secret key out of prompts, logs, command arguments, source files, and generated artifacts. Use an existing `tb login` profile or secret-injected `TB_SK` environment variable.
- Use the least-privileged identity already provided for the task. Do not request an admin key merely because a path is hidden.
- Treat `effect: write`, `effect: destructive`, and `confirm: true` as external mutations. Obtain explicit user confirmation unless the user already requested that exact mutation.
- Do not register providers, mount nodes, create keys, change secrets, or administer the gateway unless the user explicitly asks for that management action.
- Interpret `404` as either nonexistent or invisible. Do not probe around it to infer hidden paths.
- Do not automatically retry calls that may have side effects. Check the error's `retryable` signal and the command effect first.

## Workflow

### 1. Verify the target

Check whether the CLI exists, then verify the configured target without exposing credentials:

```sh
command -v tb
tb whoami --json
```

If `tb` is missing, tell the user that Node.js 22+ and `@tool-bridge/cli` are required. Ask before installing a global package. If the target is missing or authentication is rejected, ask the user to configure a profile or inject `TB_BASE_URL` and `TB_SK`; do not ask them to paste a secret into chat when a secret-input mechanism is available.

### 2. Discover the smallest relevant surface

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

Do not dump a deep tree or every tool schema into context. Narrow to a promising node first.

### 3. Inspect the exact command

Read the tool-level help before invoking it:

```sh
tb help '<node>/<tool>' --json
```

Use `cmds[].path`, `cmds[].name`, `inputSchema`, `effect`, `confirm`, `scope`, and any feedback in the response. Satisfy the schema exactly and ignore unknown optional fields for forward compatibility.

If node-level help omits the schema, follow its `hint` and open the tool-level help. If the help requires a scope the current identity lacks, stop and explain the missing capability instead of seeking a broader credential.

### 4. Invoke exactly as described

For a direct tool path:

```sh
tb call '<node>/<tool>' --args '<json-object>' --json
```

For a command that shares its node path, use the envelope form:

```sh
tb call '<node>' --tool '<command>' --args '<json-object>' --json
```

Choose the form from `cmds[].path`; do not infer it from the node kind. Use `--args-file` for complex payloads and keep temporary files outside the project when they contain sensitive data.

### 5. Validate and report

Check the returned data against the task, not merely the process exit code. Summarize which gateway path and tool were used, the relevant result, and any limitation or partial failure. Never include the secret key.

When a real call reveals a reusable, non-sensitive pitfall, offer to submit concise feedback. Feedback submission is a write and requires authorization unless it was part of the user's request.
