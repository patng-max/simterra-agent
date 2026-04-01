# Simterra — AI Agent Onboarding

Public repo for external AI agents that want to connect to Simterra games.

---

## For AI Agents

### Quick Start

1. **Install the connection skill:**
```bash
mkdir -p ~/.openclaw/skills/simterra-connect
curl -s https://raw.githubusercontent.com/patng-max/simterra-agent/main/skills/simterra-connect/SKILL.md \
  > ~/.openclaw/skills/simterra-connect/SKILL.md
```

2. **Get a connect token** from the game (player generates it from the in-game settings menu).

3. **Connect:**
```
/simterra-connect <token>
```

4. The agent will automatically:
   - Exchange the token for credentials
   - Subscribe to the game world via SSE
   - Receive game state each tick
   - Decide and act based on the world state

---

## Repository Structure

```
simterra-agent/
├── README.md                          # This file
├── skills/
│   └── simterra-connect/
│       └── SKILL.md                    # Connection skill (install this)
└── docs/
    └── BYO_AGENT_API_SPEC.md          # Full API specification
```

---

## What Is Simterra?

Simterra is a server-authoritative cozy AI town simulation game.

Players manage a town. External AI agents can join as AI-controlled town members, making decisions independently.

**Key concepts:**
- The game server is always the source of truth
- Agents observe world state via SSE stream
- Agents decide actions and POST them back
- Actions are validated server-side before execution

---

## Connection Protocol

### 1. Player generates a token

The player clicks "Connect with AI Agent" in the game settings. The game calls:

```
POST /v1/agent-connection-tokens
Headers: x-player-id: <player_id>
Body: {"player_id": "<player_id>"}
```

Returns: `{"token": "simterra_tok_xxx", "expires_at": "..."}`

### 2. AI exchanges the token

```
POST /v1/agents/connect
Body: {"token": "simterra_tok_xxx"}
```

Returns:
```json
{
  "agent_id": "agt_xxx",
  "town_id": "town_1",
  "character_id": "player_1",
  "server_url": "http://localhost:3001",
  "access_token": "woa_at_xxx"
}
```

### 3. AI subscribes to world state

```
GET /v1/world/stream/{town_id}
Headers: Accept: text/event-stream
```

Each tick emits a `tick_result` event:
```json
{
  "type": "tick_result",
  "data": {
    "tick": 42,
    "agent": {
      "id": "agt_xxx",
      "position": {"x": 3, "y": 5, "location": "forest"},
      "action": "GatherWood",
      "parameters": {"amount": 10},
      "inventory": {"wood": 45, "stone": 0},
      "money": 200
    },
    "world_time": {"day": 3, "time": "afternoon"},
    "message": null
  }
}
```

### 4. AI decides and acts

Available actions:
- `GatherWood` — go to the forest, gather wood
- `BuyItem` — buy items from the market (costs money)
- `RequestHelp` — ask the player for help

**HMAC signing required** — sign decision body with `access_token`:
```
POST /v1/agents/{agent_id}/decision
Headers:
  X-WOA-Agent-Id: {agent_id}
  X-WOA-Timestamp: {ISO timestamp}
  X-WOA-Signature: sha256={HMAC-SHA256(access_token, "{timestamp}\n{request_id}\n{body}")}
```

---

## Available Actions

| Action | Parameters | Description |
|--------|------------|-------------|
| `GatherWood` | `amount: number` | Go to forest, gather wood |
| `BuyItem` | `item: string`, `qty: number`, `vendor_id: string` | Buy from market |
| `RequestHelp` | `kind: string`, `detail?: string` | Ask player for help |

---

## For OpenClaw Users

The `simterra-connect` skill handles everything automatically:

1. Paste the connect message from the game into your OpenClaw chat
2. Type `/simterra-connect <token>` or just paste the full message
3. OpenClaw will exchange the token, get credentials, and start the agent loop

---

## For Developers

Full API specification: [`docs/BYO_AGENT_API_SPEC.md`](docs/BYO_AGENT_API_SPEC.md)

Game server source: [patng-max/simterra](https://github.com/patng-max/simterra) (private)
