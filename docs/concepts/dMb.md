---
title: "Crash course: local agent + network monitor"
summary: "Orientation for running OpenClaw as a local code-generating agent and home-network monitor"
read_when:
  - You are new to OpenClaw and want a mental model before diving into config or code
  - You are setting up a local-only agent for coding or infrastructure monitoring
---

This is a practitioner's orientation, not a reference. It covers the mental models you
need before trusting this system with sensitive work -- code on your machines, credentials
in scope, network changes. Read the linked concept docs when you need depth on a specific area.

**Sources**: inline citations below reference the actual source files where each
concept is implemented. Read those files when the doc's summary is not enough.

## What OpenClaw is

_Sources: `src/entry.ts` (CLI entry), `openclaw.mjs` (launcher), `docs/concepts/architecture.md`_

OpenClaw is a **personal AI agent gateway** that runs on your own hardware. It has three
jobs:

1. **Receive messages** from one or more messaging channels (Telegram, WhatsApp, Slack,
   iMessage, Discord, a local web chat, or a direct CLI call).
2. **Run an AI agent** (powered by the Pi SDK, described below) against your workspace
   and tools.
3. **Send the reply back** through the same channel.

Everything runs locally -- your API keys, workspace files, session history, and tool
output never leave the gateway host unless you point a channel plugin at an external
service. The gateway is a persistent daemon you start once; clients (the macOS app, web
UI, CLI) connect to it over a local WebSocket.

For your use cases, the relevant surface is small: **one gateway, one agent session,
your local file system and network, and whichever channel you use to talk to it** (even
if that's just `openclaw agent --prompt "..."` from a terminal).

---

## Architecture in three layers

```
Channel (inbound)
   ↓
┌──────────────────────────────────────────┐
│  GATEWAY  (daemon, ws://127.0.0.1:18789) │
│  Validates frames, routes sessions,      │
│  manages the plugin registry             │
└──────────────────────────────────────────┘
   ↓
┌──────────────────────────────────────────┐
│  AGENT RUNTIME  (embedded Pi)            │
│  System prompt, tool execution,          │
│  model calls, streaming, compaction      │
└──────────────────────────────────────────┘
   ↓  ↕ (tool calls)
┌──────────────────────────────────────────┐
│  HOST  (your machine)                    │
│  File system, shell, network, processes  │
└──────────────────────────────────────────┘
   ↓
Channel (outbound reply)
```

The gateway is a single process. It owns exactly one agent runtime. Multiple client apps
or channels can be connected at the same time, but they all share one underlying agent.

---

## Pi: the AI engine

_Sources: `docs/pi.md` (architecture deep-dive), `src/agents/pi-embedded-runner/run.ts` (main entry),
`src/agents/pi-embedded-subscribe.ts` (event bridge), `src/agents/pi-tools.ts` (tool wiring),
`src/agents/system-prompt.ts` (prompt assembly)_

**Pi is not a persona and not a separate process.** It is a TypeScript SDK
(`@earendil-works/pi-coding-agent`) that OpenClaw imports and instantiates in-process.
It provides the core agentic loop:

- call an LLM
- execute tool calls the LLM requests
- stream the response
- persist the session transcript to disk
- compact long sessions when the context window fills

OpenClaw wraps Pi's `AgentSession` (created via `createAgentSession()`) and replaces Pi's
default tool set with its own. From Pi's perspective it is just getting custom tools and a
custom system prompt; the loop itself is Pi's.

### What Pi contributes vs. what OpenClaw adds

| Concern | Owned by Pi SDK | Owned by OpenClaw |
|---|---|---|
| LLM call + streaming | yes | |
| Tool execution loop | yes | |
| Session persistence (JSONL) | yes | |
| Session compaction | yes (core algorithm) | wraps with safeguards |
| Auth profiles + key rotation | | yes |
| System prompt assembly | | yes (per channel/context) |
| Channel delivery | | yes |
| Tool set | Pi defaults (replaced) | custom OpenClaw tools |
| Model failover | | yes |
| Skills | | yes |

### Pi packages in use

| Package | Role |
|---|---|
| `pi-ai` | LLM abstractions: `Model`, `streamSimple`, provider APIs |
| `pi-agent-core` | Agent loop, tool execution, `AgentMessage` types |
| `pi-coding-agent` | High-level SDK: `createAgentSession`, `SessionManager`, `AuthStorage`, `ModelRegistry` |
| `pi-tui` | Terminal UI components (local TUI mode only) |

The version pinned is `0.74.x`. OpenClaw does not spawn a `pi` subprocess; it does not
use `pi`'s RPC mode. If you have `pi` installed separately, it is a completely independent
installation.

### Key Pi types you will encounter

```typescript
// The live session object -- one per agent run
AgentSession {
  prompt(text: string, opts?: { images?: ... }): Promise<void>;
  // Pi drives the loop; OpenClaw subscribes to its events
}

// Persists conversation history to disk as JSONL
SessionManager {
  static open(path: string): SessionManager;
  // branching, compaction, node traversal
}

// Resolves provider + model for a run
ModelRegistry {
  listModels(): Model[];
  resolve(id: string): Model | null;
}

// Holds API keys -- OpenClaw injects keys at runtime
AuthStorage {
  setRuntimeApiKey(provider: string, key: string): void;
}
```

You rarely interact with these directly. They surface in error messages and debug logs.

---

## Data primitives

_Sources: `src/gateway/protocol/` (wire types), `src/channels/plugins/types.core.ts` (message types),
`src/plugin-sdk/session-binding-runtime.ts` (session types), `src/plugin-sdk/tool-payload.ts` (tool types),
`src/plugin-sdk/provider-model-types.ts` (model types)_

These are the types that matter for understanding what the system knows, stores, and
transmits. Trust comes from understanding exactly what flows through each boundary.

### 1. Messages

_`src/channels/plugins/types.core.ts`_

Every inbound message from a channel is normalized to this shape before the agent sees it:

```typescript
ChannelMessage {
  id: string;            // channel-native message ID
  timestamp: number;     // unix ms
  sender: {
    name?: string;
    identity: string;    // stable sender ID within this channel
    isBot?: boolean;
  };
  text?: string;         // plain text body
  media?: ChannelMedia[]; // images, files, audio
  thread?: { id: string; parentId?: string };
  metadata?: Record<string, unknown>; // channel-specific extras
}
```

The agent's reply is:

```typescript
ChannelReply {
  text?: string;
  html?: string;
  media?: OutboundMedia[];
  mentions?: MentionTarget[];
  thread?: { id: string };
  actions?: MessageAction[];
  status?: ReplyStatus;
}
```

For your local-only setup (CLI or local web chat), you do not need to think much about
`media` or `mentions`. The text path is what matters.

### 2. Sessions

_`src/gateway/protocol/schema/sessions.ts` (wire types), `src/plugin-sdk/session-binding-runtime.ts` (runtime contract),
`~/.openclaw/agents/<agentId>/sessions/` (on-disk layout)_

A **session** is the conversation container -- a named transcript the agent appends to on
each turn.

```typescript
SessionBinding {
  sessionId: string;           // stable, assigned by OpenClaw
  conversationId: string;
  location: ChannelLocation;   // { channel, account, target }
}
```

Sessions are stored as JSONL files at:

```
~/.openclaw/agents/<agentId>/sessions/<sessionId>.jsonl
```

The JSONL format is Pi's native transcript format: each line is a JSON node with `id`,
`parentId`, and a message payload. Nodes form a tree (compaction creates branch nodes).
Each run appends to the current leaf.

You can read these files directly -- they are plain text. A session that has never been
compacted is a linear append log. After compaction, older nodes are summarized into a
single node and the tree diverges.

Session metadata (start time, last interaction, model used, etc.) lives separately in:

```
~/.openclaw/agents/<agentId>/sessions/sessions.json
```

### 3. Memory

_`src/plugin-sdk/memory-*.ts` (memory backend contracts), `~/.openclaw/workspace/` (on-disk files),
`docs/concepts/memory.md`_

Memory is **plain Markdown files on disk**. There is no hidden database.

| File | Scope | Injected into context |
|---|---|---|
| `MEMORY.md` | Durable long-term facts | Every DM session start |
| `memory/YYYY-MM-DD.md` | Daily running notes | Today + yesterday |
| `AGENTS.md` | Operating instructions, rules | Every session start |
| `SOUL.md` | Persona, tone, boundaries | Every session start |
| `USER.md` | Who you are, how to address you | Every session start |
| `TOOLS.md` | Tool conventions (guidance only) | Every session start |

All of these live in the workspace directory (default `~/.openclaw/workspace`). The agent
can read and write them at any time using standard file tools. If you ask it "remember
that my router is at 192.168.1.1", it writes that to `MEMORY.md` or a daily note.

The memory system is durable across session resets and model switches. It is the
agent's long-term state.

### 4. Tools

_`src/agents/pi-tools.ts` (tool assembly), `src/agents/tools/` (individual tools),
`src/agents/bash-tools.ts` (exec/process), `src/agents/pi-tools.policy.ts` (policy filtering)_

Tools are the agent's hands. Each tool has a name, a JSON Schema for its parameters, and
an `execute` function. The LLM requests tool calls; OpenClaw executes them and returns
results.

```typescript
AgentTool {
  name: string;
  description: string;
  parameters: JsonSchema;
  execute: (toolCallId, params, signal, onUpdate) => Promise<ToolResult>;
}

ToolResult {
  type: "text" | "error" | "image" | "media";
  data: string | Buffer | ...;
}
```

**Built-in tools available in every session:**

| Tool | What it does |
|---|---|
| `read` | Read a file |
| `edit` | Edit a file (exact string replacement) |
| `write` | Write/overwrite a file |
| `exec` | Run a shell command (subject to approval policy) |
| `process` | List and inspect running processes |
| `browser` | Fetch a URL, interact with a headless browser |
| `message` | Send a message to a channel (for multi-turn use) |
| `sessions_send` | Send a prompt to another session |
| `cron` | Schedule a recurring task |
| `gateway` | Query gateway state, presence, health |
| `image_generate` | Generate an image (if a provider is configured) |

`exec` is the most powerful and the most important to understand from a security
perspective. See the trust model section below.

### 5. Config

_`~/.openclaw/openclaw.json` (your config), `src/gateway/protocol/schema/config.ts` (schema),
`docs/gateway/configuration-reference.md` (full reference)_

The config file is `~/.openclaw/openclaw.json` (JSON5 format). All settings are
documented at [Gateway configuration](/gateway/configuration).

Key sections relevant to your use case:

```json5
{
  // Where the agent works and reads files
  agents: {
    defaults: {
      workspace: "~/.openclaw/workspace",
      model: "anthropic/claude-sonnet-4-6",
      // sandbox: { ... }   -- optional, isolates exec
    },
  },

  // Which API providers to use
  models: {
    providers: {
      anthropic: { apiKey: "sk-ant-..." },
      // openai, ollama, openrouter, etc.
    },
  },

  // Session lifecycle
  session: {
    dmScope: "per-channel-peer",  // isolate sessions per sender
    reset: { idleMinutes: 120 },
  },

  // Tool execution policy
  tools: {
    exec: {
      // see exec approvals doc
    },
  },
}
```

### 6. Gateway wire frames

_`src/gateway/protocol/index.ts` (frame union types), `src/gateway/protocol/schema/` (per-domain schemas),
`docs/gateway/protocol.md`_

If you ever read gateway logs or build automation against the WS API, frames look like:

```typescript
// Client → Gateway
RequestFrame  { type: "req", id: string, method: string, params: unknown }

// Gateway → Client (response to a request)
ResponseFrame { type: "res", id: string, ok: boolean, payload?: unknown, error?: string }

// Gateway → Client (server-pushed event)
EventFrame    { type: "event", event: string, payload: unknown, seq?: number }
```

The `agent` method is the main one: it takes a prompt and starts an agent run. The
response ack comes immediately (`runId`, `status: "accepted"`), then events stream in
until the `lifecycle:end` event.

---

## For code generation

_`docs/concepts/agent.md`, `docs/concepts/agent-workspace.md`, `docs/tools/skills.md`,
`docs/tools/exec.md`_

This is the most straightforward use case. The agent can already read, edit, write, and
exec in your workspace. You configure the workspace to be your project root (or a
symlink to it) and then talk to the agent via Telegram, the CLI, or the web UI.

### Workspace setup

```json5
{
  agents: {
    defaults: {
      workspace: "/path/to/your/project",
    },
  },
}
```

Or keep the default (`~/.openclaw/workspace`) and use `exec` to navigate around. The
workspace is just a `cwd` default, not a hard boundary (unless sandboxing is on).

### Giving the agent project context

The workspace files are injected into every session. Write your project's conventions
in `AGENTS.md` inside the workspace:

```markdown
# Project

TypeScript monorepo. Tests with Vitest. Formatter is oxfmt. No Prettier.
Main packages: packages/core, packages/ui, extensions/<plugin-id>.
Run tests: pnpm test <path>. Build: pnpm build.
```

The agent reads this at the start of every session. Keep it under ~2000 tokens or it
will be truncated (use `/context list` to check).

### Skills

Skills are reusable agent procedures -- think of them as saved slash commands. They live
in `<workspace>/skills/` as Markdown files. The agent auto-discovers them and lists
them in context. Useful for codebases with repeating patterns (e.g. "add a new extension",
"write a test for X", "run the full check suite").

### Session resets

Code sessions can grow long. Compaction (`/compact`) summarizes older history into a
compact node and frees context. The agent remembers what it did via the compact summary.

For long autonomous coding runs, the 48-hour default agent timeout is generous. You can
interrupt with `/stop`, steer mid-run by sending a new message (it gets queued as a
steer), or switch models mid-session with `/new <model>`.

---

## For home network monitoring

_`docs/automation/cron-jobs.md`, `docs/tools/exec-approvals.md`, `docs/gateway/sandboxing.md`,
`docs/automation/hooks.md`, `docs/gateway/security/`_

This use case requires understanding a few additional things: scheduling, exec policy,
and how to build durable state across runs.

### The basic pattern

```
cron trigger (periodic)
   → agent wakes up
   → exec: read logs, run diagnostic commands
   → write findings to memory/daily note
   → exec: make config changes if needed (subject to approval)
   → send alert to your channel if something is wrong
```

### Log inspection

The `read` tool reads any file the process has access to. Point it at your log directory:

```
read /var/log/syslog
read /path/to/central-log-repo/*.log
read /var/log/nginx/access.log
```

For structured logs, `exec` lets you run `jq`, `grep`, `awk`, and any other tool you have
installed. There is no difference between "read a file" and "exec cat file" from the
agent's perspective, but `read` is more efficient and does not go through exec approval.

For central log aggregation: once you have a service (Loki, a syslog daemon, a simple
file aggregator) writing to a known path, you point the agent at that path. The agent
does not need native log client libraries -- it uses `exec` to run whatever you normally
use to query the log system (e.g. `logcli`, `journalctl`, `sqlite3`).

### Scheduling periodic runs

The `cron` tool registers recurring agent runs:

```
set a cron to check /var/log/auth.log every hour for failed SSH attempts
```

Or configure it in `~/.openclaw/openclaw.json`:

```json5
{
  cron: {
    jobs: [
      {
        id: "network-check",
        schedule: "0 * * * *",   // every hour
        prompt: "Check auth.log for anomalies. Write findings to today's memory note. Alert me if there are more than 5 failed auth attempts from a new IP.",
      },
    ],
  },
}
```

Cron sessions are isolated -- each run starts fresh and terminates. They do not share
context with your DM session. They can write to memory files, which your DM session will
pick up.

### Making network changes

The agent can run any shell command via `exec`. For security-sensitive changes (firewall
rules, config reloads, iptables edits), you want the exec approval system engaged:

```json5
{
  tools: {
    exec: {
      approvals: {
        mode: "require",             // require approval for dangerous commands
        allowPatterns: [             // patterns that auto-approve (no confirmation)
          "journalctl *",
          "ss -*",
          "ip *",
          "systemctl status *",
        ],
        blockPatterns: [             // patterns that always block
          "rm -rf *",
          "dd *",
        ],
      },
    },
  },
}
```

See [Exec approvals](/tools/exec-approvals) for the full policy grammar. The default is
`mode: "warn"` -- the agent can exec freely but logs the commands. For a network
monitor that might run autonomous fixes, start with `mode: "require"` and relax specific
patterns you are confident in.

### Sandboxing

Sandboxing constrains the agent's file and exec access to a defined root. For a monitoring
agent that should only read logs and write notes -- not touch system config -- enable it:

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "non-main",   // sandbox non-DM (cron) sessions
        workspaceRoot: "~/.openclaw/sandboxes",
      },
    },
  },
}
```

With sandboxing on, `read`/`edit`/`write` are constrained to the sandbox workspace. `exec`
still runs on the host unless you add container config. For most home setups,
`mode: "require"` on exec approvals plus careful `allowPatterns` is enough without full
container sandboxing.

### Building durable state

The agent's memory system is the right place to accumulate monitoring state across runs:

- Write a summary of each check to `memory/YYYY-MM-DD.md`.
- Write durable inventory facts (known devices, baseline port map, trusted IPs) to `MEMORY.md`.
- The agent loads today's and yesterday's daily notes automatically, so recent context
  is always available without manual injection.

Example AGENTS.md additions for a monitoring setup:

```markdown
# Network monitoring context

My home network: 192.168.1.0/24. Router at 192.168.1.1.
Known hosts: see MEMORY.md under "Known devices".
Log directory: /var/log/network-monitor/
Central syslog: /var/log/syslog

When running a network check:
1. Read today's memory note first to see what was already checked.
2. Check auth.log for new failed attempts since last check.
3. Run `ss -tlnp` to see open ports; compare to baseline in MEMORY.md.
4. Write findings (even if nothing notable) to today's memory note with a timestamp.
5. Only alert if there is a concrete anomaly.
```

---

## Trust model: what to verify before sensitive work

_`docs/gateway/security/`, `docs/gateway/sandboxing.md`,
`docs/gateway/sandbox-vs-tool-policy-vs-elevated.md`, `docs/tools/exec-approvals.md`,
`src/agents/pi-tools.policy.ts` (policy resolution)_

### The agent has real file system access

`read`, `edit`, and `write` are not sandboxed by default. The agent can read any file
your user can read, including `~/.ssh/`, credential files, and config files outside the
workspace. Before running autonomous work on a sensitive machine, either:

- Point the workspace at a specific project directory.
- Enable `agents.defaults.sandbox` to constrain file access.
- Review what the agent is reading with `exec ls` calls in session history.

### Exec runs as your user

`exec` runs shell commands as the OS user running the gateway. If the gateway runs as
root (don't do this) the agent has root exec. If it runs as your normal user, exec has
your normal user's permissions. This is the right model for most setups: normal user
access, with `sudo` available only if you have passwordless sudo configured and accept
that risk.

### Tool results go into the transcript

Every tool result (file reads, exec output, browser fetches) is stored in the session
JSONL file. If the agent reads a file containing secrets, that content is in the
transcript. The transcript is stored at `~/.openclaw/agents/<agentId>/sessions/` and is
only as protected as that directory.

Keep sessions local. Do not sync `~/.openclaw/` to a cloud service without understanding
what is in those JSONL files.

### API keys are in `~/.openclaw/credentials/`

Model provider keys are stored in `~/.openclaw/credentials/` (or system keychain on
macOS). The gateway reads them at startup. The agent does not have a tool to read
credential files (they are outside the workspace and the read tool path), but `exec cat`
could reach them. This is why exec approval policy matters.

### The gateway exposes a local WebSocket

By default the gateway binds to `127.0.0.1:18789` -- loopback only. It is not
reachable from other devices on your LAN without explicit remote access configuration
(Tailscale, SSH tunnel, or changing `gateway.host`). Keep it on loopback unless you
need remote access. If you add remote access, enable auth tokens (`gateway.auth`).

### Allowlists for messaging channels

If you connect a public-facing channel (Telegram, WhatsApp), configure `allowFrom` so
only your account can trigger the agent:

```json5
{
  channels: {
    telegram: {
      allowFrom: ["your-telegram-username"],
    },
  },
}
```

Without this, anyone who knows your bot token can prompt your agent (which has exec
access to your machine).

---

## Quick reference: data flow for a monitoring run

```
1. Cron fires
      ↓
2. Gateway creates an isolated session (cron scope)
      ↓
3. System prompt assembled:
     - OpenClaw base rules
     - AGENTS.md (your monitoring instructions)
     - MEMORY.md (known devices, baseline)
     - memory/today.md + yesterday.md (recent findings)
     - Tool schemas
      ↓
4. Prompt sent to model (e.g. claude-sonnet-4-6)
      ↓
5. Model requests tool calls:
     read /var/log/auth.log
     exec ss -tlnp
     exec journalctl -u nginx --since "1 hour ago"
      ↓
6. OpenClaw executes each tool (checking approval policy)
      ↓
7. Results returned to model as tool result nodes in session
      ↓
8. Model produces reply:
     - writes daily note via write tool
     - sends alert via message tool if anomaly found
     - responds NO_REPLY if nothing to report
      ↓
9. Session JSONL appended; cron run complete
```

---

---

## Suggested reading order

Read in this order. Each layer builds on the previous one. Stop when you have
enough to make progress; come back for the rest when a specific question arises.

### Layer 1: orientation (read first, any order within)

These give you the mental model without implementation detail.

| Doc | What it covers | Why it matters for your use case |
|---|---|---|
| [Gateway architecture](/concepts/architecture) | What the gateway is, how clients connect, wire protocol shape | Understanding the daemon you are running |
| [Agent runtime](/concepts/agent) | Workspace, bootstrap files, session concept, model refs | The config knobs you will actually turn |
| [Agent workspace](/concepts/agent-workspace) | File map, sandbox warning, where memory lives | Trust boundary: what files the agent can touch |
| [Session management](/concepts/session) | DM isolation, lifecycle, daily/idle resets, storage | Why context doesn't bleed between conversations |
| [Context](/concepts/context) | What the model sees, how to inspect it, budget | Debugging "why does it not know X" |

### Layer 2: core mechanics (read before doing complex work)

| Doc | What it covers | Why it matters |
|---|---|---|
| [Memory overview](/concepts/memory) | MEMORY.md, daily notes, memory_search | The durable state layer for monitoring |
| [Agent loop](/concepts/agent-loop) | Turn lifecycle, queueing, hooks, timeouts | Understanding run behavior, failures, timeouts |
| [Pi integration architecture](/pi) | Pi SDK internals, session files, tool pipeline, auth rotation | Deep trust source: what the actual AI loop does |
| [Streaming](/concepts/streaming) | Block streaming, chunking, soft chunks | Relevant if you add a messaging channel |
| [Compaction](/concepts/compaction) | Session size management, branching | Long-running monitoring sessions will need this |

### Layer 3: tools and execution (before trusting it with exec)

| Doc | What it covers | Why it matters |
|---|---|---|
| [Exec tool](/tools/exec) | What exec does, working directory, environment | The most powerful tool; understand it fully |
| [Exec approvals](/tools/exec-approvals) | Approval modes, allow/block patterns, require mode | How to gate dangerous commands |
| [Exec approvals: advanced](/tools/exec-approvals-advanced) | Pattern grammar, profile-specific policy | Fine-tuning for your specific command set |
| [Sandbox vs tool policy vs elevated](/gateway/sandbox-vs-tool-policy-vs-elevated) | Three isolation layers explained and compared | Choosing the right security posture |
| [Sandboxing](/gateway/sandboxing) | File and exec confinement, container config | Full isolation if you need it |
| [Gateway security](/gateway/security/) | Trust model, auth modes, pairing | Before exposing the gateway beyond loopback |

### Layer 4: automation and scheduling (for the monitoring use case)

| Doc | What it covers | Why it matters |
|---|---|---|
| [Cron jobs](/automation/cron-jobs) | Scheduling periodic agent runs | How to set up hourly/daily network checks |
| [Cron vs heartbeat](/automation/cron-vs-heartbeat) | When to use each | Choosing the right scheduling primitive |
| [Hooks](/automation/hooks) | Event-driven scripts, lifecycle events | Triggering on alerts or session events |
| [Standing orders](/automation/standing-orders) | Persistent agent instructions | Alternative to cron for ongoing monitoring |
| [Tasks](/automation/tasks) | Background task records | Tracking long-running checks |

### Layer 5: configuration reference (when you know what you want)

| Doc | What it covers |
|---|---|
| [Gateway configuration](/gateway/configuration) | Full config file reference |
| [Configuration reference](/gateway/configuration-reference) | All config keys with types and defaults |
| [Config: tools](/gateway/config-tools) | Tool policy, exec policy, approval config |
| [Config: agents](/gateway/config-agents) | Workspace, sandbox, model, session config |
| [Secrets](/gateway/secrets) | How to store credentials safely |
| [Gateway security](/gateway/security/) | Auth modes, trusted-proxy, Tailscale |

### Layer 6: deep dives (as needed)

| Doc | When to read it |
|---|---|
| [Model failover](/concepts/model-failover) | When you have multiple API keys and want rotation |
| [Multi-agent routing](/concepts/multi-agent) | When you want to run parallel specialist agents |
| [Session pruning](/concepts/session-pruning) | When sessions grow too large |
| [Skills](/tools/skills) | When you want reusable agent procedures |
| [Creating skills](/tools/creating-skills) | When you want to write your own skills |
| [ACP agents](/tools/acp-agents) | When you want subagents spawned by the main agent |
| [Slash commands](/tools/slash-commands) | Reference for in-chat control commands |
| [Diagnostics](/gateway/diagnostics) | When something is broken and you need logs |
| [Pi development workflow](/pi-dev) | When you are modifying the Pi integration itself |

---

## Getting started

```bash
# Install and initialize
pnpm install
openclaw setup          # creates ~/.openclaw/openclaw.json and workspace files

# Test with a direct prompt (no channel needed)
openclaw agent --prompt "What files are in my workspace?"

# Start the gateway (persistent daemon)
openclaw gateway

# Inspect state
openclaw status         # session store, recent activity
openclaw doctor         # config validation, common issues
```

Then configure your model provider in `~/.openclaw/openclaw.json` (or via `openclaw setup`),
set your workspace to the right directory, and start talking to it.

---

## Further reading

- [Agent runtime](/concepts/agent) -- workspace, bootstrap files, model refs
- [Agent workspace](/concepts/agent-workspace) -- file map, sandboxing, backup
- [Session management](/concepts/session) -- lifecycle, isolation, resets
- [Context](/concepts/context) -- what the model sees, how to inspect it
- [Memory](/concepts/memory) -- MEMORY.md, daily notes, memory search
- [Pi integration architecture](/pi) -- Pi SDK internals, session management, tool pipeline
- [Agent loop](/concepts/agent-loop) -- turn lifecycle, hooks, queueing
- [Gateway architecture](/concepts/architecture) -- gateway, clients, nodes, wire protocol
- [Exec approvals](/tools/exec-approvals) -- approval policy for shell commands
- [Hooks](/automation/hooks) -- event-driven scripts for lifecycle events
- [Gateway configuration](/gateway/configuration) -- full config reference
