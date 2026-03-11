# Configuration for Better Harness Engineering in OpenClaw: Run External Coding Agents (Codex, Claude Code, etc.) via ACP (Agent Client Protocol)

This guide walks through binding Codex and Claude Code to dedicated Telegram forum topics via OpenClaw's ACP (Agent Client Protocol) runtime.

Compared to the conventional [sub-agent](https://docs.openclaw.ai/tools/subagents) approach, ACP sessions offer three key advantages:

- **Resilient session lifecycle.** ACP sessions survive OpenClaw gateway restarts - your Codex or Claude Code task picks up where it left off rather than being lost on a crash or redeploy.
- **Dedicated workspaces.** You can assign each coding agent its own working directory, and their conversations are isolated from your main chat - no context bleed between agents or with your primary OpenClaw session.
- **Protocol-level benefits.** As ACP evolves, improvements to streaming, tooling, and multi-agent coordination flow in as the protocol matures.

---

## Prerequisites

- OpenClaw >= 2026.3.8 installed
- A working Telegram bot configured in OpenClaw (Telegram and Discord are the only two supported channels for this setup; this guide uses Telegram as the example)
- Codex CLI and Claude Code CLI installed and authenticated

---

## Step 1: Configure Telegram Bot Privacy

Before adding the bot to any group, disable its privacy mode.

BotFather → `/mybots` → select your bot → Bot Settings → Group Privacy → **Turn off**

> **Important:** Do this before adding the bot to the group. If done after, the bot may miss group messages until it is removed and re-added.

---

## Step 2: Create a Telegram Group and Forum Topics

1. Create a new Telegram group
2. Add your bot to the group and set it as Admin
3. Group Settings → Topics → Enable forum mode
4. Create two topics: `#Codex` and `#Claude`

---

## Step 3: Get Group ID and Topic IDs

Send a message in each topic, then query the Bot API:

```bash
curl -s "https://api.telegram.org/bot<TOKEN>/getUpdates"
```

Replace `<TOKEN>` with your bot token. Locate `chat.id` (Group ID) and `message_thread_id` (Topic ID) in the response, e.g.:

- `chat.id`: `-1009876543210`
- `message_thread_id`: `2`, `3`

---

## Step 4: Install and Configure the acpx Plugin

```bash
# Install the plugin
openclaw plugins install acpx

# Enable it
openclaw config set plugins.entries.acpx.enabled true

# Declare trust (suppresses startup warnings)
openclaw config set plugins.allow '["acpx"]'

# ACP sessions have no TTY - auto-approve all file and command operations
openclaw config set plugins.entries.acpx.config.permissionMode approve-all

# Fail on unhandleable permission requests rather than silently skipping
openclaw config set plugins.entries.acpx.config.nonInteractivePermissions fail

# Give the queue owner enough time to acquire the session before it's declared dead (Codex/Claude Code have slow startup)
openclaw config set plugins.entries.acpx.config.queueOwnerTtlSeconds 60
```

Resulting config:

```jsonc
"plugins": {
  "allow": ["acpx"],
  "entries": {
    "acpx": {
      "enabled": true,
      "config": {
        "permissionMode": "approve-all",
        "nonInteractivePermissions": "fail",
        "queueOwnerTtlSeconds": 60
      }
    }
  }
}
```

---

## Step 5: Global ACP Configuration

```bash
openclaw config set acp.enabled true
openclaw config set acp.dispatch.enabled true
openclaw config set acp.backend acpx
openclaw config set acp.allowedAgents '["codex", "claude"]'
openclaw config set acp.maxConcurrentSessions 8
openclaw config set acp.stream.coalesceIdleMs 300
openclaw config set acp.stream.maxChunkChars 1200
openclaw config set acp.runtime.ttlMinutes 120
```

Resulting config:

```jsonc
"acp": {
  "enabled": true,
  "dispatch": { "enabled": true },
  "backend": "acpx",
  "allowedAgents": ["codex", "claude"],
  "maxConcurrentSessions": 8,
  "stream": {
    "coalesceIdleMs": 300,    // Stream output coalescing interval (ms)
    "maxChunkChars": 1200     // Max characters per push
  },
  "runtime": {
    "ttlMinutes": 120         // Session idle timeout (minutes)
  }
}
```

---

## Step 6: Enable Thread Binding

```bash
openclaw config set session.threadBindings.enabled true
openclaw config set session.threadBindings.idleHours 24
openclaw config set channels.telegram.threadBindings.enabled true
openclaw config set channels.telegram.threadBindings.spawnAcpSessions true
```

Resulting config:

```jsonc
"session": {
  "threadBindings": {
    "enabled": true,
    "idleHours": 24
  }
},
"channels": {
  "telegram": {
    "threadBindings": {
      "enabled": true,
      "spawnAcpSessions": true  // Allow dynamic ACP session spawning in any topic or DM thread
    }
  }
}
```

> With `spawnAcpSessions: true`, you can use `/acp spawn --thread auto` to create an ACP session in any topic or DM thread, supplementing static bindings.

---

## Step 7: Configure Group and Topics

Run `openclaw config edit` and add the following under `channels.telegram`:

```jsonc
"groups": {
  "<GROUP_ID>": {
    "requireMention": false,
    "topics": {
      "<CODEX_TOPIC_ID>": { "requireMention": false },
      "<CLAUDE_TOPIC_ID>": { "requireMention": false }
    }
  }
},
"accounts": {
  "default": {
    "groupPolicy": "allowlist",
    "groupAllowFrom": ["<YOUR_TELEGRAM_USER_ID>"]
  }
}
```

`groupPolicy` and `groupAllowFrom` go under `accounts.default`; without both, group messages are silently dropped.

Replace `<GROUP_ID>`, `<CODEX_TOPIC_ID>`, and `<CLAUDE_TOPIC_ID>` with the values from Step 3.

---

## Step 8: Define Agents

Run `openclaw config edit` and append a `list` under `agents` (leave existing `defaults` intact):

```jsonc
"agents": {
  "defaults": {
    // ... keep existing config ...
  },
  "list": [
    {
      "id": "codex",
      "runtime": {
        "type": "acp",
        "acp": {
          "agent": "codex",
          "backend": "acpx",
          "mode": "persistent",
          "cwd": "/your/workspace/path"
        }
      }
    },
    {
      "id": "claude",
      "runtime": {
        "type": "acp",
        "acp": {
          "agent": "claude",
          "backend": "acpx",
          "mode": "persistent",
          "cwd": "/your/workspace/path"
        }
      }
    }
  ]
}
```

---

## Step 9: Configure Bindings

Run `openclaw config edit` and add a top-level `bindings` array. Use `type: "acp"` for automatic session lifecycle management. The `peer.id` must use the canonical format `<GROUP_ID>:topic:<TOPIC_ID>`:

```jsonc
"bindings": [
  {
    "type": "acp",
    "agentId": "codex",
    "match": {
      "channel": "telegram",
      "accountId": "default",
      "peer": {
        "kind": "group",
        "id": "<GROUP_ID>:topic:<CODEX_TOPIC_ID>"
      }
    },
    "acp": { "label": "codex" }
  },
  {
    "type": "acp",
    "agentId": "claude",
    "match": {
      "channel": "telegram",
      "accountId": "default",
      "peer": {
        "kind": "group",
        "id": "<GROUP_ID>:topic:<CLAUDE_TOPIC_ID>"
      }
    },
    "acp": { "label": "claude" }
  }
]
```

> `type: "route"` only handles routing - session lifecycle is not managed and you must manually run `/acp spawn`. `type: "acp"` ensures the session exists automatically and is recommended for persistent bindings.

> **Note:** Telegram group IDs start with `-100` (e.g. `-1009876543210`). Make sure to include the full ID with the minus sign.

---

## Step 10: Restart Gateway and Verify

```bash
openclaw gateway
```

In each topic, send:

```
/acp doctor
```

Expected response: `healthy: yes` with backend details. Run `/acp status` to confirm session state, then send `Who are you?` in each topic to verify the correct agent responds.

---

## Architecture Overview

```
Telegram DM → default account → main agent (existing behavior unchanged)

Telegram Group
  ├─ Codex topic → default account → ACP binding → codex agent → acpx → Codex CLI
  └─ Claude topic → default account → ACP binding → claude agent → acpx → Claude Code CLI
```

A single bot handles DMs and group topics through independent session paths with no interference.

---

Enjoy harness engineering!

## References

- [OpenClaw Docs - ACP Agents](https://docs.openclaw.ai/tools/acp-agents)
