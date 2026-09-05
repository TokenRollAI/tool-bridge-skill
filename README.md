# Tool Bridge Agent Skill

Give a coding agent access to the tools, context, files, and devices exposed by a [Tool Bridge](https://github.com/TokenRollAI/tool-bridge) gateway. The skill uses live contracts and the `tb` CLI to complete everyday calls, offline device work, and requested administration. Its short entrypoint loads detailed guidance only for the task at hand.

## Install

Install with the open Agent Skills CLI:

```sh
npx skills add TokenRollAI/tool-bridge-skill
```

Or use it once without installing:

```sh
npx skills use TokenRollAI/tool-bridge-skill@tool-bridge
```

The installer supports Codex, Claude Code, Cursor, OpenCode, and other compatible agents. Select the `tool-bridge` skill if prompted.

## Connect a gateway

Install the Tool Bridge CLI with Node.js 22 or newer, then configure a gateway profile:

```sh
npm install -g @tool-bridge/cli
tb login --base-url https://gateway.example.com
tb whoami --json
```

For automation, inject `TB_BASE_URL` and a least-privileged `TB_SK` through the agent environment. Do not commit either value or paste the key into prompts.

Once installed and connected, ask the agent naturally, for example:

```text
Find the documentation search tool in Tool Bridge and use it to answer this question.
```

```text
Use Tool Bridge to inspect the deployment status. Do not make changes.
```

The gateway's runtime description is always authoritative; this repository does not hard-code an instance URL, credential, or tool catalog.

The normal path is one call for a known tool, or `search --schemas --json` followed by a call for an unknown read-only capability. Broad exploration uses compact search results. Device delivery, Context and Store files, management, and failure recovery each have an on-demand reference. Feedback is consulted when it helps resolve a problem; votes and submissions require authorization.

Other requests the skill can handle:

```text
Run the device's maintenance command when it is reachable, queueing it if offline. Report the operation ID without waiting.
```

```text
Download the file identified by this store:// URI to a new local file.
```

```text
Update this gateway setting and verify that the saved revision is actually applied.
```

## License

MIT
