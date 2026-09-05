# Device calls and offline delivery

Read before choosing delivery for a device or handling a returned operation. To install or expose a local device, use [Management](management.md).

## Select delivery from the live contract

Command metadata `delivery` is a **capability**: `realtime` (also the default when absent in command help), `mailbox`, or `both`. The CLI flag is a **per-call policy**: `realtime` (default), `mailbox`, or `fallback`.

| User's need | Policy |
|---|---|
| Get an immediate result from an online command | `realtime` |
| Explicitly queue durable work on a mailbox-capable command | `mailbox` |
| Execute now if possible, otherwise queue when supported | `fallback` |

For a known mailbox-capable command, offline-tolerant work is one invocation:

```sh
tb call '<full-command-path>' --args '<json-object>' --delivery fallback --json
```

For an unknown capability, first use `tb search '<keywords>' --schemas --json`, then call the suitable result. Inspect command help for incomplete metadata or a mutation; existing user authorization still applies. In particular, search can omit `delivery` even for a mailbox-capable command: resolve it from help before choosing offline delivery. Offline presence does not hide a device or prevent mailbox discovery. Do not require an online check before a valid offline-tolerant call.

The gateway makes the fallback decision. For `both`, it tries realtime and enqueues only when it can prove `not_dispatched`. Mailbox-only commands enqueue directly. A completed realtime call, including a device business error, never enqueues. A send with unknown outcome never enqueues and is not retryable. Do not infer dispatch certainty from an error code or HTTP status, and never follow a failed call with a manual enqueue.

Optional enqueue controls:

```sh
tb call '<path>' --args-file '<json-file>' --delivery mailbox \
  --ttl 3600 --idempotency-key '<key-for-this-logical-enqueue>' --json
```

`--ttl` is a positive number of seconds until operation expiry. TTL and idempotency key require `mailbox` or `fallback`. The key deduplicates a caller-owner-scoped mailbox enqueue; it does **not** make realtime business execution idempotent, so it does not justify repeating an ambiguous fallback call. Do not create a new key to work around an unresolved operation.

## Interpret the returned result

With an explicit `--delivery`, `tb call --json` returns a discriminated result:

```json
{"delivery":"realtime","result":{"example":"command output"}}
```

```json
{"delivery":"mailbox","operation":{"operationId":"...","deviceId":"...","targetPath":"...","state":"queued"}}
```

The operation example abbreviates the remaining metadata. Read `result` for realtime and `operation` for mailbox. This also applies to explicit `--delivery realtime`; omitting the flag returns the raw command value. HTTP status and headers are not part of CLI JSON. Use `operation.deviceId`, not an ID guessed from the normalized mount path. `tb device op get` subsequently returns the operation detail directly, without this delivery wrapper.

Report queued work as queued, with its operation identity. End the synchronous flow there by default. If the user's current goal needs the state or terminal result, read it once:

```sh
tb device op get '<device-id>' '<operation-id>' --json
```

If it remains pending, report that fact. Do not create a fixed polling loop unless the user explicitly requests ongoing monitoring. Claim/renew/complete belong to the device runtime, never to the caller.

| State or signal | Meaning for the caller |
|---|---|
| `queued` / `claimed` | Pending / leased; neither proves task completion |
| `succeeded` | Check the operation's result against the task |
| `rejected` / `failed` | Read the error; do not automatically create another execution |
| `result_unknown` | Execution may have started; result is not recoverable |
| `expired` with `executionMayHaveOccurred: true` | Previously claimed; execution may have occurred |
| `cancelled` | Terminal cancellation; interpret alongside prior execution information |

For either ambiguous execution state, do not re-call, re-enqueue, or retry. A new execution requires independently proven business idempotency and user authorization. Expiry or cancellation is not a general rollback guarantee.

## Inspect or cancel when requested

```sh
tb device ls --json
tb device op ls '<device-id>' --state queued --state claimed --json
tb device op cancel '<device-id>' '<operation-id>' --json
```

Listings are paginated. Cancelling a queued operation prevents its handler from starting; cancelling a claimed one is cooperative and can leave it claimed with `cancelRequestedAt`. Report cancellation requested until terminal evidence establishes more.

Current limitations matter to task selection: caller SK revocation does not itself cancel already authorized operations; use the operation cancellation workflow. Mailbox handlers do not receive realtime call-scoped Store upload capability, so do not promise that a realtime file-producing command will produce the same artifact offline without a supporting runtime contract.
