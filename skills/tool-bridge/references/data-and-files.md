# Context, Store, and skillhub

Read when the task involves connected content, an uploaded/downloaded file, or published skills. Use the resource identity to choose the interface.

| Resource or intent | Interface |
|---|---|
| Opaque file / device artifact with `store://default/...` | `tb store` |
| Named entry inside an existing Context namespace | `tb ctx` |
| Agent Skill bundle in a skillhub | `tb skill` |
| S3 endpoint, bucket, or active backend configuration | `tb storage`; see [Management](management.md) |

## Store objects

A `store://default/...` URI is a stable object identity, not content or a bearer credential. The current owner still needs permission to read it. Retrieve the file directly; no Context mount is needed:

```sh
tb store stat 'store://default/<object-id>' --json
tb store get 'store://default/<object-id>' --out '<new-local-file>' --json
```

Skip `stat` when metadata is unnecessary and the task already calls for downloading. `get` defaults to binary stdout; `--json` requires `--out`. Output files are create-only, so choose a new path instead of overwriting an existing file.

```sh
tb store list --json
tb store upload '<local-file>' --json
tb store rm 'store://default/<object-id>' --json
```

Upload creates a new opaque object; it does not replace a named Context entry. The CLI handles the advertised relay/direct transport. Do not construct grants, signed requests, or a second completion request yourself. An optional upload `--idempotency-key` identifies the same owner-scoped create attempt, not an overwrite path.

Create a share only for an explicitly requested external handoff:

```sh
tb store share 'store://default/<object-id>' --ttl 3600 --json
tb store revoke-share '<share-id>' --json
```

Choose TTL for the requested handoff; the example is not a universal default. Share success returns a short-lived bearer `$ref`. Deliver it directly to the intended recipient, keeping it out of logs, feedback, and persisted task notes. A stable Store URI may be retained; grants and signed URLs may not. Deleting an object invalidates its reads/shares; revoking a share targets that share.

## Context entries

Reuse a known namespace or discover it through the visible tree/help. Search entry content within that namespace, not with global tool search:

```sh
tb ctx ls '<namespace>' --json
tb ctx search '<namespace>' '<query>' --json
tb ctx cat '<namespace>' '<entry>' --json
```

An entry may contain text, JSON, or a short-lived `$ref` instead of inline bytes. A `$ref` is not the file's content; treat it as a bearer capability, never as a URL to which the gateway SK should be attached.

For authorized text/JSON authoring, inspect the namespace's write contract and use `put` (create or replace):

```sh
tb ctx put '<namespace>' '<entry>' --file '<utf8-file>' --content-type application/json --json
```

`--content` accepts inline text; with neither `--content` nor `--file`, `put` reads stdin. `--meta key=value` adds metadata; `--if-version '<observed-version>'` guards a replacement. A conflict requires reconciling fresh content with the intended edit, not blindly overwriting it.

Binary direct upload is **optional provider capability**. Use it only when runtime help advertises `create_upload`:

```sh
tb ctx upload '<namespace>' '<entry>' --file '<local-file>' --json
```

Existing entries fail with `conflict` unless replacement was explicitly requested and `--force` is supplied. The standard Node S3-backed Context currently does not advertise direct upload. If unavailable, use `put` only for actual text/JSON. A standalone binary artifact can use Store when that satisfies the user's goal; if a named binary Context entry is required, explain the missing capability rather than changing the destination or encoding silently.

## Skillhub

```sh
tb skill ls '<hub>' --json
tb skill search '<hub>' '<query>' --json
tb skill get '<hub>' '<skill-id>' --json
tb skill get '<hub>' '<skill-id>' --file '<bundled-path>' --json
```

Read only the bundle files needed for the task. `get --out '<directory>'` downloads the bundle; it does not by itself authorize executing its scripts or installing it into the agent environment. `tb skill publish '<hub>' '<local-directory>'` publishes/replaces a bundle; publishing, removal, and mounting require the corresponding user intent. Inspect leaf `--help` for these less frequent actions.
