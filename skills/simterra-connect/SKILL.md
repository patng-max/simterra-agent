---
name: simterra-connect
description: "Use when user sends /simterra-connect with a Simterra token, or pastes a Simterra AI agent connect message. Connect an AI agent to a running Simterra game server."
---

# Simterra Connect — Join a Simterra Game as an AI Agent

## When This Skill Is Triggered

- User sends `/simterra-connect <token>` in Telegram
- User pastes a Simterra connect message from the game client
- User asks to "connect to Simterra" or "join the game"

---

## For Other AI Agents

If you don't have this skill, install it:

```bash
SKILL_DIR="$HOME/.openclaw/skills/simterra-connect"
mkdir -p "$SKILL_DIR"
curl -s https://raw.githubusercontent.com/patng-max/simterra-agent/main/skills/simterra-connect/SKILL.md > "$SKILL_DIR/SKILL.md"
```

Skill source: **https://github.com/patng-max/simterra-agent**

---

## Connection Flow

### Step 1 — Parse the token

Two formats are supported:

**Format A — command:**
```
/simterra-connect simterra_tok_abc123...
```

**Format B — template (from game client):**
```
🎮 Join my Simterra game as an AI agent!

Run this command to connect:
/simterra-connect simterra_tok_abc123...

This will connect you to the Simterra game server as a game agent.
Server: http://localhost:3001
Town: town_1
Your character: player_1
```

Extract `simterra_tok_abc123...` from either format.

### Step 2 — Call the Connect API

```
POST {server_url}/v1/agents/connect
Content-Type: application/json
Body: {"token": "simterra_tok_xxx"}
```

With `server_url` from the message (e.g. `http://localhost:3001`).

Expected response:
```json
{
  "agent_id": "agt_xxx",
  "town_id": "town_1",
  "character_id": "player_1",
  "server_url": "http://localhost:3001",
  "access_token": "woa_at_xxx"
}
```

### Step 3 — Start the agent loop

Subscribe to the SSE world stream:
```
GET {server_url}/v1/world/stream/{town_id}
Headers: Accept: text/event-stream
```

On each SSE tick event, the server sends a `tick_result` event with the current game state (tick number, agent position, inventory, money, time).

**Decide an action:** Use your LLM to analyze the observation and choose from:
- `GatherWood` — go to the forest and gather wood
- `BuyItem` — buy items from the market (costs money)
- `RequestHelp` — ask the player for help

**Post the decision:**
```
POST {server_url}/v1/agents/{agent_id}/decision
Content-Type: application/json
Headers:
  X-WOA-Agent-Id: {agent_id}
  X-WOA-Timestamp: {ISO timestamp}
  X-WOA-Signature: sha256={HMAC-SHA256 signature}
Body: {
  "decision": {
    "action_type": "GatherWood" | "BuyItem" | "RequestHelp",
    "parameters": {...},
    "reason": "..."
  }
}
```

**HMAC signing:** Sign the body using HMAC-SHA256 with the `access_token` as the secret key:
```
signature = HMAC-SHA256(access_token, "{timestamp}\n{request_id}\n{body}")
```

### Step 4 — Confirm

Reply to the user with confirmation details.

---

## Connection Details

| Item | Value |
|------|-------|
| Server URL | `http://localhost:3001` (local dev) |
| Town ID | `town_1` |
| Player Character | `player_1` |
| Token expiry | 15 minutes from generation |
| Token use | Single-use |

## Troubleshooting

**"Invite is invalid or expired"**
→ Token has expired. User must generate a new one from the game settings.

**"Agent connect failed"**
→ Token was already used. Generate a fresh token from the game.

**Server not running**
→ The Simterra server must be running. Ask the user to start it first.
