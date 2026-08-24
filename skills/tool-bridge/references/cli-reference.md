# Tool Bridge CLI reference

Load this reference when target configuration or command syntax is unclear, a Context write/upload is needed, or discovery, authentication, invocation, or feedback handling fails. It is not a required preflight for a known read-only call.

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

Within one continuous task, reuse a successful `whoami` result while the selected profile/BaseURL and identity remain unchanged. Do not run it before every call.

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

If an exact command path, current schema, `effect: read`, and `confirm: false` are already known from the current runtime, skip discovery and call it directly. Otherwise prefer one `search --json`; when a single hit is unambiguous and contains enough schema/effect/confirm data, call it without an extra help request.

`tb search '<query>' --json` already includes each result's arguments schema at `items[].tool.inputSchema`. For human-readable output, add `--schemas` to print those same schemas inline without another request. Use command-level help when search does not expose a detail needed for the decision, the result is ambiguous, or the operation is mutating:

```sh
tb help '<node>/<command>' --json
```

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

Take the full path from command help's `cmds[].path`, or from a search result as the exact `<items[].path>/<items[].tool.name>` pair. Use only fields returned by the gateway; do not infer a path from the node kind or invent a command name. Identifiers in a path (each segment and the command name) are case-insensitive and normalized to lowercase.

Arguments must form a JSON object. Choose exactly one of four mutually exclusive input forms; omitting all four sends `{}`:

```sh
tb call '<path>' '{"query":"tool bridge"}' --json
tb call '<path>' --args '{"query":"tool bridge"}' --json
tb call '<path>' --args-file '<temporary-json-file>' --json
tb call '<path>' --arg query='tool bridge' --arg limit=10 --json
```

The first form is positional JSON after `<path>`. `--args` supplies the same object as a flag. `--args-file -` reads the entire JSON object from stdin:

```sh
printf '%s\n' '{"query":"tool bridge"}' | tb call '<path>' --args-file - --json
```

Repeated `--arg key=value` builds a flat object. It parses only `true`/`false` as booleans, `null` as null, and plain integers or decimals such as `42`, `-1`, and `1.5` as numbers; every other value remains a string. A repeated key uses its last value. Use positional JSON, `--args`, or `--args-file` for nested objects and arrays, exponent or hexadecimal notation, large integers, or strings that must remain exactly `"true"` or `"42"`.

Prefer `--args-file` for long payloads. Keep sensitive temporary files outside the project and remove them when no longer needed.

Do not reuse a failed write or destructive call automatically. A timeout can leave the remote outcome unknown.

## Context writes and uploads

Use `tb ctx put` for text or JSON that can be sent inline, from a UTF-8 file, or through stdin. It creates or replaces an entry and supports metadata and optimistic concurrency:

```sh
tb ctx put '<context>' '<entry>' --content '<text>' --json
tb ctx put '<context>' '<entry>' --file '<utf8-file>' --content-type application/json --json
```

Use direct upload for binary or large file content:

```sh
tb ctx upload '<context>' '<entry>' --file '<local>' --json
```

Upload is conditional by default: an existing entry fails with `conflict`. Add `--force` only when the user has explicitly authorized replacing that exact entry. The CLI obtains a short-lived upload grant and sends the bytes directly to object storage without the Tool Bridge key. Treat the grant URL and headers as temporary bearer secrets: do not print, log, store, cache, or include them in generated artifacts or feedback.

## Error handling

Gateway TBError responses use `{code,message,retryable}` internally. With `--json`, the CLI emits a flat failure object to stdout and exits with status 1:

```json
{"ok":false,"error":"failure message","code":"invalid_argument","retryable":false}
```

`error` is the message string, not a nested error object. `code`, `retryable`, `hint`, and `feedback` are omitted when unavailable. Common codes mean:

- `not_found`: the path is absent or intentionally hidden from this identity
- `permission_denied`: the visible operation lacks a required scope
- `invalid_argument`: re-read command-level help and compare the payload with `inputSchema`
- `conflict`: refresh state before deciding whether to try again
- `unavailable`: upstream or gateway capability is temporarily unavailable
- `rate_limited`: retry only when safe, using bounded backoff
- `internal`: report the failure without exposing request secrets

When `tb call` fails with `unavailable`, `internal`, `invalid_argument`, or `rate_limited`, the CLI makes a best-effort lookup on that exact path. It may add a human-readable `hint`; when matching entries exist, JSON output also includes at most three `feedback` summaries shaped as `{id,score,title}`. This lookup can fail silently and never replaces the primary error. Treat an attached entry as the first troubleshooting branch and fetch only the most relevant detail:

```sh
tb feedback get '<path>' '<feedback-id>' --json
```

Use `tb feedback ls '<path>' --json` only when the failed call did not attach a useful entry, the path is unfamiliar or failure-prone and warrants a preflight, or a new submission needs deduplication. Do not add feedback requests to every successful read call.

If a listed entry accurately explains the behavior or provides a validated workaround, it can be voted up after the requested result is secured, provided gateway writes are already authorized:

```sh
tb feedback vote '<path>' '<feedback-id>' up --json
```

Use `down` only when current runtime evidence shows that an entry is incorrect or harmful. Do not downvote merely because an entry was irrelevant to the current task.

When an abnormal call reveals a new reproducible issue or a validated resolution, submit feedback after securing the requested result when gateway writes are already authorized and the lesson is genuinely reusable:

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

Feedback submission and voting require `call` permission on the target path. If the current task does not authorize gateway writes, do not interrupt a successful result merely to request a vote or submission. Preserve or mention a draft only when it would materially help the user or an authorized operator.
