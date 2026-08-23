# Tool Bridge CLI reference

Load this reference before the first Tool Bridge operation in a session and whenever discovery, authentication, invocation, or feedback handling fails.

## Target configuration

The CLI resolves the gateway in this order:

1. Explicit `--base-url` and `--sk` flags
2. `TB_BASE_URL` and `TB_SK`
3. The selected local profile created by `tb login`

Avoid `--sk` because it can enter shell history and process listings. Prefer an existing profile for interactive use and secret-injected environment variables for automation.

Configure an interactive profile:

```sh
tb login --base-url https://gateway.example.com
tb whoami --json
```

Install the CLI only with user approval:

```sh
npm install -g @tool-bridge/cli
```

The npm CLI requires Node.js 22 or newer.

## Discovery commands

All commands below are scoped to what the current key can see:

```sh
tb whoami --json
tb tree --depth 2 --json
tb tree '<path>' --depth 2 --json
tb ls '<path>' --json
tb search '<query>' --json
tb help '<path>' --json
```

`tb search` may be absent on gateways without a search capability. Fall back to `tree`, `ls`, and `help` rather than treating that as a gateway-wide failure.

Node-level help is an index; it lists the commands under a node. Request `<node>/<command>` help to obtain a single command's complete input schema. Important command fields are:

- `path`: the full command path, used verbatim as the call target
- `name`: the command name
- `inputSchema`: JSON Schema for the arguments object
- `outputSchema` or `returns`: response contract when declared
- `scope`: required permission
- `effect`: typically `read`, `write`, or `destructive`
- `confirm`: whether the user must confirm before the call
- `feedback`: high-value operational notes from prior users

Unknown optional fields are forward-compatible and should be ignored.

## Invocation form

There is one call form. A command is a virtual leaf under its node, so `cmds[].path` is always the full command path. Pass it verbatim and send the arguments object as the request body:

```sh
tb call 'docs/search/query' --args '{"q":"tool bridge"}' --json
tb call 'system/status/get' --json
```

Take the path from `cmds[].path` exactly; do not assemble it from the node kind or invent a command name. Identifiers in a path (each segment and the command name) are case-insensitive and normalized to lowercase.

Arguments must be a JSON object. Inline JSON, `--args`, and `--args-file` are mutually exclusive. Prefer `--args-file` for long payloads:

```sh
tb call '<path>' --args-file '<temporary-json-file>' --json
```

Do not reuse a failed write or destructive call automatically. A timeout can leave the remote outcome unknown.

## Error handling

Tool Bridge errors use `{code,message,retryable}`. Common meanings:

- `not_found`: the path is absent or intentionally hidden from this identity
- `permission_denied`: the visible operation lacks a required scope
- `invalid_argument`: re-read command-level help and compare the payload with `inputSchema`
- `conflict`: refresh state before deciding whether to try again
- `unavailable`: upstream or gateway capability is temporarily unavailable
- `rate_limited`: retry only when safe, using bounded backoff
- `internal`: report the failure without exposing request secrets

The CLI may attach known feedback to failed calls. Treat that hint as the first troubleshooting branch and read the referenced item before changing the request:

```sh
tb feedback ls '<path>' --json
tb feedback get '<path>' '<feedback-id>' --json
```

If a listed entry accurately explains the behavior or provides a validated workaround, vote it up promptly instead of submitting a duplicate:

```sh
tb feedback vote '<path>' '<feedback-id>' up --json
```

Use `down` only when current runtime evidence shows that an entry is incorrect or harmful. Do not downvote merely because an entry was irrelevant to the current task.

When an abnormal call reveals a new reproducible issue or a validated resolution, submit feedback at the point of learning rather than waiting until the end of the task:

```sh
tb feedback submit '<path>' \
  --title '<short summary>' \
  --detail '<how to avoid or resolve the issue>' \
  --json
```

Before submitting:

1. Run `tb feedback ls '<path>' --json` again to prevent duplicates.
2. Keep the title to one concrete symptom or lesson.
3. State the observed condition and verified workaround in the detail.
4. Label an unresolved report as unresolved; do not present a guess as a fix.
5. Remove credentials, personal data, customer payloads, and internal-only URLs.

Feedback submission and voting require `call` permission on the target path. If the current task does not authorize gateway writes, draft the exact entry or vote and ask once for confirmation immediately. If permission is missing, report that fact and preserve the draft for an authorized user.
