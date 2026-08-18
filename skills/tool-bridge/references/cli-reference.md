# Tool Bridge CLI reference

Load this reference before the first Tool Bridge operation in a session and whenever discovery, authentication, or invocation fails.

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

Node-level help is an index. For MCP, HTTP, and tool providers, request `<node>/<tool>` help to obtain the complete input schema. Important command fields are:

- `path`: HTTP invocation path; also determines the CLI call form
- `name`: tool or command name
- `inputSchema`: JSON Schema for the arguments object
- `outputSchema` or `returns`: response contract when declared
- `scope`: required permission
- `effect`: typically `read`, `write`, or `destructive`
- `confirm`: whether the user must confirm before the call
- `feedback`: high-value operational notes from prior users

Unknown optional fields are forward-compatible and should be ignored.

## Invocation forms

Use direct form when `cmds[].path` includes the tool segment:

```sh
tb call 'docs/search/query' --args '{"q":"tool bridge"}' --json
```

Use envelope form when several commands share the node path:

```sh
tb call 'system/status' --tool get --args '{}' --json
```

Arguments must be a JSON object. Inline JSON, `--args`, and `--args-file` are mutually exclusive. Prefer `--args-file` for long payloads:

```sh
tb call '<path>' --args-file '<temporary-json-file>' --json
```

Do not reuse a failed write or destructive call automatically. A timeout can leave the remote outcome unknown.

## Error handling

Tool Bridge errors use `{code,message,retryable}`. Common meanings:

- `not_found`: the path is absent or intentionally hidden from this identity
- `permission_denied`: the visible operation lacks a required scope
- `invalid_argument`: re-read tool-level help and compare the payload with `inputSchema`
- `conflict`: refresh state before deciding whether to try again
- `unavailable`: upstream or gateway capability is temporarily unavailable
- `rate_limited`: retry only when safe, using bounded backoff
- `internal`: report the failure without exposing request secrets

The CLI may attach known feedback to failed calls. Read the referenced item before changing the request:

```sh
tb feedback ls '<path>' --json
tb feedback get '<path>' '<feedback-id>' --json
```

Submit feedback only after authorization and only when it is reusable and non-sensitive:

```sh
tb feedback submit '<path>' \
  --title '<short summary>' \
  --detail '<how to avoid or resolve the issue>' \
  --json
```

Never put credentials, personal data, customer payloads, or internal-only URLs in feedback.
