# CLI, target selection, and discovery

Read for connection setup, search refinement, pagination, or argument syntax. Device delivery, files, administration, and failure recovery have separate references linked from SKILL.md.

## Connection and local help

Use `tb --help` or `tb <group> <command> --help` to inspect the installed CLI without contacting a gateway. `tb help '<path>' --json` instead requests live gateway help.

The CLI resolves each target field from explicit flags, then `TB_BASE_URL` / `TB_SK`, then the selected login profile. Avoid `--sk`: it exposes the key in argv. When selecting a different gateway, select matching credentials too; do not combine an old injected key with a new URL accidentally.

```sh
tb login --base-url https://gateway.example.com
tb whoami --json
```

For login automation, inject `TB_SK` through the environment; `login` has no secret-stdin option. The user can also log in from their own terminal, but its interactive SK prompt currently echoes input, so do not capture that interaction in an agent transcript. Do not ask for keys in chat. Verify `authenticated` once, then reuse the target until target/profile/identity changes: `whoami` can exit successfully with `authenticated: false`, and authentication does not prove permission for a particular tool. Do not dump profiles or the environment to diagnose it.

`tb use '<profile>'` selects a saved profile; environment variables still take precedence. There is no general `--profile` flag on calls (`login --profile` names a saved profile).

If `tb` is absent, the CLI package is `@tool-bridge/cli` and requires Node.js 22+. Install it when CLI setup is within the user's request; otherwise explain the missing prerequisite before making a global environment change:

```sh
npm install -g @tool-bridge/cli
```

Use installed `--help` to resolve a version mismatch; do not silently upgrade as a troubleshooting step.

## Search for capabilities

Choose the payload size for the next action:

```sh
# Browse candidates without downloading every argument schema.
tb search '<keywords>' --json
# Find a tool and prepare to call it in the same discovery round trip.
tb search '<keywords>' --schemas --json
```

Both return a page with `items`; full results request `items[].tool.inputSchema`. Construct a call path from `items[].path` and `items[].tool.name`. Reuse returned paths verbatim: federation prefixes are already included, so do not prepend `source.path` or call a remote source URL directly. Full results can still lack optional metadata; for offline delivery, resolve missing `delivery` from command help.

Search uses keyword matching, not semantic matching or automatic translation. Prefer a few distinctive words in the catalog's language. If recall is poor, revise the keywords, remove unnecessary terms, or try the other likely language. An empty page is not proof that the gateway lacks the capability.

Useful refinements, selected for the task rather than added by default:

```sh
tb search '<keywords>' --path-prefix '<visible-prefix>' --effect read --schemas --json
tb search '<keywords>' --matching all --json
tb search '<keywords>' --federation local --json
```

- `--effect` is repeatable: `read`, `write`, `destructive`, `unknown`. Do not filter to `read` when the requested task needs a mutation.
- `--matching best` is the default and starts with the highest matching coverage; `all` broadens results. `--min-coverage` accepts a fraction in `(0,1]`; do not combine `all` with a fraction other than `1`.
- `--federation local|recursive` selects search scope. Gateways advertising federated search default to recursive. Keep this setting and all query/filter choices when continuing a cursor.
- `partial: true` and `sources` describe incomplete source coverage. CLI warnings go to stderr while JSON remains on stdout. A useful hit may still be called; report the gap when the user's task requires a complete inventory.
- Use `--limit` and the returned `--cursor` only when more results are needed. Cursors are opaque: do not edit or decode them. If a cursor is invalidated by changed state, restart the same search without it; do not reuse it with different filters.
- Global `tb search --mode` accepts only `keyword`. A Context provider's optional semantic search is a different capability.

If search is unavailable or the user wants to explore the visible tree:

```sh
tb tree --depth 2 --json
tb ls '<visible-path>' --json
tb help '<node-or-command-path>' --json
```

Start shallow and follow relevant branches. `ls --json` returns a bare children array, not a page's `items`. Node help is an index; command help supplies the selected command's complete contract. `cmds[].path` is already the full invocation path; `outputSchema` is structured while `returns` is prose. Help's `feedback`, `hint`, and `note` are top-level fields. An absent schema is not an empty schema; inspect help before inventing arguments. Unknown effect is not permission to treat the tool as read-only. Ignore unknown optional response fields for compatibility.

## Arguments and output

Choose exactly one input form; omitting all sends `{}`:

```sh
tb call '<path>' '{"query":"example"}' --json
tb call '<path>' --args '{"query":"example"}' --json
tb call '<path>' --args-file '<json-file>' --json
tb call '<path>' --arg query=example --arg limit=10 --json
```

`--args-file -` reads the JSON object from stdin. Use file/stdin for nested structures, large payloads, or sensitive values; keep temporary files outside the repository with owner-only access, and remove them when done. Preserve values through proper shell quoting; JSON encoding alone is not shell escaping.

Repeatable `--arg key=value` builds a flat object: `true`/`false`, `null`, and plain integers/decimals become typed scalars; everything else stays a string. Duplicate keys use the last value. Use JSON for arrays, nested objects, or strings that must remain exactly `"true"` or `"42"`.

Prefer `--json` for machine consumption. Without an explicit delivery policy, `call` prints the command's returned value, not a universal `{ok:true,data:...}` envelope. With any explicit `--delivery`, success is `{delivery:"realtime",result:...}` or `{delivery:"mailbox",operation:...}`; see [Devices](devices.md). JSON failures go to stdout with a nonzero exit; `error` is a string:

```json
{"ok":false,"error":"failure message","code":"invalid_argument","retryable":false}
```

`code`, `retryable`, `hint`, and `feedback` may be absent. Diagnose with [Recovery](recovery.md). For downloads use [Data and files](data-and-files.md) so binary bytes never mix with JSON.
