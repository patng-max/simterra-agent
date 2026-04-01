---
name: simterra-connect
description: "Use when user sends /simterra-connect with a Simterra token, or pastes a Simterra AI agent connect message."
---

# Simterra Connect

## When Triggered

- User sends `/simterra-connect <token>`
- User pastes a Simterra connect message from the game client

## What To Do

### 1. Extract token

Parse `simterra_tok_xxx` from the message.

### 2. Call connect API

```bash
curl -s -X POST http://localhost:3001/v1/agents/connect \
  -H "Content-Type: application/json" \
  -d "{\"token\": \"<token>\"}"
```

Response:
```json
{
  "agent_id": "agt_xxx",
  "town_id": "town_1",
  "character_id": "player_1",
  "server_url": "http://localhost:3001",
  "access_token": "woa_at_xxx",
  "platform_signing_secret": "woa_sigsec_xxx"
}
```

### 3. Save state

Write to `~/.openclaw/secure/simterra-agent.json`:
```json
{
  "agent_id": "agt_xxx",
  "town_id": "town_1",
  "character_id": "player_1",
  "server_url": "http://localhost:3001",
  "access_token": "woa_at_xxx",
  "platform_signing_secret": "woa_sigsec_xxx",
  "tick": 0
}
```

### 4. Spawn background agent

```bash
mkdir -p ~/.openclaw/secure
MINIMAX_API_KEY=$MINIMAX_API_KEY tsx ~/.openclaw/scripts/simterra-bg-agent.ts >> /tmp/simterra-bg-agent.log 2>&1 &
echo "pid=$!"
```

The script reads credentials from the state file and starts the game loop.

### 5. Confirm

```
✅ Connected to Simterra!
- Town: town_1
- Character: player_1
- Server: http://localhost:3001
The AI agent is now playing. Decisions will be posted here as they happen.
```

### Stopping

```bash
pkill -f 'simterra-bg-agent'
rm ~/.openclaw/secure/simterra-agent.json
```

## How It Works

The background script:
1. Reads credentials from `~/.openclaw/secure/simterra-agent.json`
2. Subscribes to SSE stream
3. On each tick: calls MiniMax LLM → POSTs signed decision
4. Continues until game ends or you say "stop"

## Available Actions

| Action | Parameters |
|--------|------------|
| `GatherWood` | `amount: number` |
| `BuyItem` | `item: string`, `qty: number`, `vendor_id: string` |
| `RequestHelp` | `kind: string`, `detail?: string` |

## Troubleshooting

**"Token expired or used"** → Generate fresh token from game settings.

**"Server not running"** → Start: `cd /Users/patrickclaw/dev/simterra/server && pnpm dev`

**Server restarted** → Token becomes invalid. Stop agent, generate new token, reconnect.
