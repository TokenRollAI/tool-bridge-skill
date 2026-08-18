# Tool Bridge Agent Skill

Give a coding agent access to the tools, context, and devices exposed by a [Tool Bridge](https://github.com/TokenRollAI/tool-bridge) gateway. The skill teaches the agent to discover capabilities from live `~help`, use feedback before and during troubleshooting, promptly contribute validated lessons, call the correct tool, and respect side-effect and credential boundaries.

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

The feedback loop is deliberate: agents read relevant feedback before a call, consult it immediately after abnormal behavior, vote on useful existing guidance, and submit new non-sensitive findings when authorized.

## License

MIT
