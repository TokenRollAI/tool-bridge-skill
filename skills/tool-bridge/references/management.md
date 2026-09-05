# Requested management and device setup

Read when the user requests administration or local device exposure. Tool use alone does not authorize these changes. Existing authorization for the requested change is sufficient; do not request the same permission again merely because this reference was loaded.

## Find the owning command family

Use `tb <family> --help` and the chosen leaf's `--help` for installed syntax. For gateway operations, read the relevant live `tb help 'system/<module>' --json`, then the selected command contract as needed. Some families adapt registry or other system operations; do not invent a system path from a CLI family name.

| Task | CLI family |
|---|---|
| Gateway health and summary | `tb status` |
| Install or recover an instance | `tb setup` |
| Runtime settings and applied state | `tb config` |
| S3 backend identities and credentials | `tb storage` |
| Compose deployment settings and executor | `tb deployment` |
| PostgreSQL / Redis maintenance | `tb maintenance` |
| Encryption/signing roots and secret backups | `tb keys` |
| Upstream tool or Context integrations | `tb integration`, `tb tool`, `tb ctx` |
| Remote gateway mount and federation policy | `tb server`, `tb federation` |
| Plugin runtime registration | `tb plugin` |
| Access keys and provider credentials | `tb sk`, `tb secret` |
| Skillhub publication and mounts | `tb skill` |
| Node notes or operational feedback | `tb note`, `tb feedback` |
| Local device exposure and lifecycle | `tb connect`, `tb mount fs`, `tb daemon` |

Prefer the dedicated subcommand where it exists: it supplies input handling and output safeguards. For other advertised commands use `tb call` with the exact runtime path. Do not bypass missing CLI/runtime capability by editing the database or inventing HTTP endpoints.

Read only the state and schema needed to prepare the change. Explain material effects outside the user's existing authorization before proceeding. In noninteractive execution, many destructive CLI prompts are bypassed; `--yes` and `--json` are not evidence of user authorization.

## Runtime configuration: save, apply, verify

PG is the configuration authority. Settings are not managed by editing legacy environment variables. Obtain the current schema and settings:

```sh
tb config schema --json
tb config get --json
```

Prepare a complete settings object by preserving unrelated current fields and changing the requested values. The file contains the settings object itself, not an invented wrapper or partial patch:

```sh
tb config validate --file '<settings-json>' --json
tb config update --revision '<observed-revision>' --file '<settings-json>' --json
tb config apply --revision '<saved-revision>' --json
tb config status --json
```

Use actual revisions returned by the gateway; do not calculate or guess the next one. Update saves **desired** settings; apply must succeed before the change is effective. Check the applied revision/effective state before reporting success. If the user requested only staging a change, stop after saving. On conflict, re-read and reconcile the intended change; do not replace a revision and replay a stale full settings snapshot.

## Storage: identity is separate from the active default

```sh
tb storage list --json
tb storage get '<backend-id>' --json
tb storage add --file '<protected-backend-json>' --json
tb storage test '<backend-id>' --revision '<backend-revision>' --json
tb storage activate '<backend-id>' --revision '<tested-revision>' \
  --active-revision '<observed-active-revision>' --json
```

Read the current write schema before preparing the file; credentials belong in protected file/stdin, not argv. A backend test performs real temporary S3 reads/writes/deletes, so include it in the authorized storage setup, not a routine read-only health check. Keep external probes bounded.

Endpoint/bucket/region are immutable backend identity. `storage update` rotates credentials at the same location; a different location needs a new backend. Activating it changes the default for **new** objects, not existing objects, uploads, or Context bindings. Verify the active state without claiming that old data migrated. Referenced or active backends cannot simply be removed.

## Integrations and credentials

For a built-in integration, inspect `tb integration catalog --search '<keywords>' --json`, selecting its `exportDetails` kind/auth/config contract. Then use `integration add --help`; multi-export providers require an explicit export selection. For an externally registered plugin, `tb plugin get '<id>' --json` exposes registered export descriptions. There is no `tb describe`; do not send a POST call to a guessed `~describe` path or bypass the gateway to inspect an upstream.

Choose `tb ctx mount --provider ... --export ...` for an external Context export, or `tb tool mount --kind tool --provider ... --export ...` for an external tool export. Inspect leaf help for the complete required arguments.

Store credentials under a SecretStore reference, then attach the reference:

```sh
tb secret set --name '<credential-ref>' --json < '<protected-credential-file>'
tb integration add '<path>' --provider '<provider-id>' --credential '<credential-ref>' --json
```

The file contains the provider's single secret or a JSON object of string-valued credential fields, according to its descriptor. Secret values do not belong in `providerConfig`, `--config`, static `--header`, or secret-valued argv fields. `authRef` / `--auth-ref` and integration `--credential` identify stored secrets, not their values.

MCP OAuth mounting and authorization are separate steps: `tb tool auth` starts authorization after mounting. If DCR is unavailable, the mount supports `--oauth-client-id` and an optional `--oauth-client-secret-ref`; confidential client secrets still use SecretStore. Removing an integration and deleting its stored credential are separate actions.

For a remote gateway, `server add --remote-url` is the upstream address and `--sk-ref` references its secret. Global `--base-url` still selects the gateway being administered. Inspect the applicable federation policy without broadening its allowlist as a workaround.

Creating SKs or completing setup may return a secret once. Arrange protected delivery before running the command so captured tool output does not place the key in the conversation. An empty/unrestricted scope is not a convenient default.

## Setup, deployment, and maintenance

- Setup/recovery uses local pairing, not a working Admin SK. Inspect `tb setup status`; `tb setup pair --directory '<bootstrap-directory>'` requires the local deployment host and admin utility. Use `--recovery` for an initialized instance. Protected token/config files feed the corresponding setup command; only one input can consume stdin.
- An initialized instance with failed PG connectivity is a recovery task, not a fresh install. Installation success requires a ready application; a listener or successful settings submission is insufficient.
- `tb deployment update` saves desired Compose settings. A separately running, restricted `tb deployment agent` on the deployment host performs application; inspect `tb deployment status` for actual outcome. There is no generic `tb deployment apply` command. Do not start a persistent executor merely to answer a status request.
- Use `tb maintenance` for database/Redis changes and `tb keys` for encryption/signing roots. Read the exact schema, revision, instance identity, and recovery prerequisites. These are not ordinary config updates. Never clear maintenance protection or delete runtime records to bypass a refusal.
- Key rotation can require resuming a re-encryption job; verify completion before retiring old roots. Signing rotation with `--revoke-existing` invalidates existing grants and requires that effect to be within scope. `tb keys backup --out '<new-protected-file>'` writes an owner-only secret file; it is not a full PG/S3 backup.
- After a maintenance timeout, inspect state before further mutation. Unknown commit state or missing recovery prerequisites requires stopping with the specific unresolved condition, not an automatic retry.

## Connect a local device

Use a Device SK scoped to the intended registration path. Inspect `tb connect --help` and expose only the requested commands/directories. Prefer a reviewed structured command profile with `--no-shell`; do not broaden to `--allow '*'` to resolve a failed command.

Persistent `tb daemon` currently requires Linux, a normal user, and systemd user services; it is not a macOS or root service. Only create it when persistent exposure is requested:

```sh
tb daemon install --device-id '<device-id>' --path '<mount-path>' \
  --no-shell --command-profile '<reviewed-profile-json>'
tb daemon status --json
```

Install freezes validated profile values; editing the source JSON does not update the installed daemon. Reinstall to apply a requested profile change. `inheritEnv` passes local environment values to child programs; it is not a SecretStore reference. Install/restart success waits for the installed configuration to reach `ready`.

Daemon uninstall removes the local service, not the login profile or server-side key. Device retirement may require separately authorized key revocation; queued operations still require their own cancellation workflow. Do not erase installation journals to retry work with an unknown outcome.
