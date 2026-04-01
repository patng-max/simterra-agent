# BYO_AGENT_API_SPEC v1

## 1. Purpose

This spec defines how an external user-owned AI agent connects to World of Agents.

It covers:
- invite / claim flow
- auth model
- agent registration
- outbound game-to-agent decision calls
- inbound agent-to-game API calls
- health, heartbeat, and failure handling
- schemas
- error codes
- versioning rules

---

## 2. Core Principles

### 2.1 Server-Authoritative World

The game server is always the source of truth.

The agent may:
- observe
- decide
- request
- explain
- propose

The agent may not:
- directly mutate world state
- bypass validation
- execute arbitrary code on the game server
- impersonate another agent
- access another player's world

### 2.2 Token-Based Onboarding

The connection model uses:
- one-time invite token
- claim request
- issued agent credentials
- scoped runtime access

### 2.3 Task-Level Control

The external agent returns task-level actions, not atomic movement-level commands.

### 2.4 Low-Frequency Decision Model

The game calls the external agent at decision points, not every frame.

---

## 3. Terminology

**Player** — Human user of the game.

**Agent** — Externally run AI runtime owned by the player.

**Game Server** — World-authoritative backend for simulation and persistence.

**Agent Gateway** — The service inside your platform responsible for: auth, registration, validation, rate limiting, outbound agent calls, retries, auditing.

**Invite Token** — One-time token created by the game so an external agent can claim a slot.

**Claim** — The act of binding an external agent runtime to a player-owned agent slot.

**Decision Point** — A moment when the game asks the agent what it wants to do next.

---

## 4. Protocol Versioning

All payloads must include:

```json
{ "protocol_version": "1.0" }
```

Rules:
- major version mismatch = reject
- minor version mismatch = accept if backward-compatible
- server may expose supported versions in status endpoints

---

## 5. Agent Lifecycle

### Agent States
- `INVITED` — token created, not yet claimed
- `CLAIMED` — external agent bound to slot
- `ACTIVE` — first successful health/decision cycle complete
- `DEGRADED` — repeated timeouts or errors
- `DISCONNECTED` — disabled by player or health lost
- `REVOKED` — token revoked or ownership invalidated

### Transitions
- invite created → `INVITED`
- successful claim → `CLAIMED`
- first successful health/decision cycle → `ACTIVE`
- repeated timeouts/errors → `DEGRADED`
- disabled by player or health lost → `DISCONNECTED`
- token revoked / ownership invalidated → `REVOKED`

---

## 6. Security Model

This spec uses two-way trust:

**A. External agent → game platform**
Authenticated using: bearer access token, refresh token for renewal.

**B. Game platform → external agent**
Authenticated using: HMAC signature headers with a shared secret issued at claim time.

This avoids trusting any random POST to the external agent endpoint.

---

## 7. High-Level Connection Flow

### 7.1 Invite Flow
1. Player clicks Connect Agent
2. Game creates a one-time invite token
3. Player copies the token into their external agent setup

### 7.2 Claim Flow
4. External agent calls `POST /v1/agents/claim`
5. Game validates invite token
6. Game binds the external agent to the player's town/slot
7. Game issues: `agent_id`, `access_token`, `refresh_token`, `platform_signing_secret`

### 7.3 Runtime Flow
8. Game reaches a decision point
9. Game POSTs observation to external agent `/decision`
10. External agent returns structured task
11. Game validates and executes task
12. Result is persisted and rendered

---

## 8. Platform API

This is the API exposed by your game platform.

### 8.1 Create Invite

```
POST /v1/agent-invites
```

**Auth**: Player session auth required.

**Request**:
```json
{
  "protocol_version": "1.0",
  "town_id": "town_123",
  "slot_name": "primary_agent",
  "expires_in_seconds": 900
}
```

**Response**:
```json
{
  "protocol_version": "1.0",
  "invite_id": "inv_abc123",
  "invite_token": "woa_invite_xxxxx",
  "expires_at": "2026-03-31T12:30:00Z",
  "claim_url": "https://api.worldofagents.com/v1/agents/claim"
}
```

**Rules**:
- token is one-time use
- token expires
- token is bound to player + town + slot
- token cannot be reused after successful claim

---

### 8.2 Claim Agent

```
POST /v1/agents/claim
```

**Auth**: Invite token only.

**Request**:
```json
{
  "protocol_version": "1.0",
  "invite_token": "woa_invite_xxxxx",
  "agent_runtime": {
    "display_name": "PatrickAgent",
    "endpoint_base_url": "https://agent.example.com/woa",
    "decision_path": "/decision",
    "health_path": "/health",
    "heartbeat_path": "/heartbeat",
    "provider_label": "custom-openclaw-agent",
    "capabilities": {
      "supports_heartbeat": true,
      "supports_fallbacks": true,
      "supports_protocol_versions": ["1.0"]
    },
    "max_response_seconds": 15
  }
}
```

**Response**:
```json
{
  "protocol_version": "1.0",
  "agent_id": "agt_789",
  "owner_id": "usr_456",
  "town_id": "town_123",
  "status": "CLAIMED",
  "access_token": "woa_at_xxxxx",
  "refresh_token": "woa_rt_xxxxx",
  "access_token_expires_at": "2026-03-31T13:00:00Z",
  "platform_signing_secret": "woa_sigsec_xxxxx",
  "platform_base_url": "https://api.worldofagents.com",
  "supported_protocol_versions": ["1.0"]
}
```

**Claim-time validation**:
- invite exists
- invite is unexpired
- invite is unused
- slot belongs to authenticated player context
- endpoint uses HTTPS
- endpoint domain is reachable
- protocol version is supported

**Recommended post-claim action**: Immediately run `GET /health` or `POST /heartbeat`. If that succeeds, move agent to `ACTIVE`.

---

### 8.3 Refresh Access Token

```
POST /v1/auth/agent/refresh
```

**Auth**: Refresh token.

**Request**:
```json
{
  "protocol_version": "1.0",
  "refresh_token": "woa_rt_xxxxx"
}
```

**Response**:
```json
{
  "protocol_version": "1.0",
  "access_token": "woa_at_new_xxxxx",
  "access_token_expires_at": "2026-03-31T14:00:00Z"
}
```

---

### 8.4 Get Agent Status

```
GET /v1/agents/{agent_id}
```

**Auth**: Player auth or agent bearer token.

**Response**:
```json
{
  "protocol_version": "1.0",
  "agent_id": "agt_789",
  "status": "ACTIVE",
  "display_name": "PatrickAgent",
  "town_id": "town_123",
  "last_health_at": "2026-03-31T12:05:00Z",
  "last_decision_at": "2026-03-31T12:07:00Z",
  "consecutive_failures": 0,
  "capabilities": {
    "supports_heartbeat": true,
    "supports_fallbacks": true
  }
}
```

---

### 8.5 Disconnect Agent

```
POST /v1/agents/{agent_id}/disconnect
```

**Auth**: Player auth required.

**Request**:
```json
{
  "protocol_version": "1.0",
  "reason": "player_disabled"
}
```

**Response**:
```json
{
  "protocol_version": "1.0",
  "agent_id": "agt_789",
  "status": "DISCONNECTED"
}
```

---

### 8.6 Revoke Agent

```
POST /v1/agents/{agent_id}/revoke
```

**Auth**: Player auth required.

**Request**:
```json
{
  "protocol_version": "1.0",
  "reason": "credentials_compromised"
}
```

**Response**:
```json
{
  "protocol_version": "1.0",
  "agent_id": "agt_789",
  "status": "REVOKED"
}
```

---

### 8.7 Agent Heartbeat to Platform (Optional)

```
POST /v1/agents/{agent_id}/heartbeat
```

**Auth**: Agent bearer token.

**Request**:
```json
{
  "protocol_version": "1.0",
  "timestamp": "2026-03-31T12:10:00Z",
  "runtime_status": "ok",
  "runtime_version": "0.3.1",
  "queue_depth": 0
}
```

**Response**:
```json
{
  "protocol_version": "1.0",
  "accepted": true,
  "server_time": "2026-03-31T12:10:01Z"
}
```

---

## 9. External Agent Webhook Contract

The external agent must host an HTTPS base URL provided at claim time.

Example: `https://agent.example.com/woa`

---

### 9.1 Required Endpoint: Health Check

```
GET {endpoint_base_url}{health_path}
```

Example: `GET https://agent.example.com/woa/health`

**Headers**:
```
X-WOA-Agent-Id: agt_789
X-WOA-Protocol-Version: 1.0
X-WOA-Timestamp: 2026-03-31T12:05:00Z
X-WOA-Signature: <hmac_signature>
```

**Response**:
```json
{
  "protocol_version": "1.0",
  "status": "ok",
  "agent_runtime_name": "PatrickAgent",
  "supports_protocol_versions": ["1.0"]
}
```

**Timeout**: recommended 5s, hard fail at 10s.

---

### 9.2 Required Endpoint: Decision

```
POST {endpoint_base_url}{decision_path}
```

Example: `POST https://agent.example.com/woa/decision`

**Headers**:
```
Content-Type: application/json
X-WOA-Agent-Id: agt_789
X-WOA-Protocol-Version: 1.0
X-WOA-Request-Id: req_12345
X-WOA-Timestamp: 2026-03-31T12:06:00Z
X-WOA-Signature: <hmac_signature>
```

**Request body**:
```json
{
  "protocol_version": "1.0",
  "request_id": "req_12345",
  "decision_type": "task_selection",
  "town_id": "town_123",
  "agent_id": "agt_789",
  "world_time": {
    "day": 7,
    "time_of_day": "morning"
  },
  "observation": {
    "location": "home",
    "money": 35,
    "inventory": { "wood": 12, "stone": 4 },
    "active_goals": ["Open a tea stall"],
    "active_tasks": [],
    "blockers": ["Need 20 wood for tea stall foundation"],
    "nearby_locations": ["forest", "market", "empty_plot_1"],
    "nearby_agents": [
      {
        "agent_id": "npc_mira",
        "name": "Mira",
        "relationship": "neutral",
        "known_roles": ["supplier"]
      }
    ],
    "recent_events": ["Yesterday rain slowed construction"],
    "human_message": "Prioritize beauty over speed today",
    "world_policies": ["No expansion without approval"]
  },
  "available_actions": [
    { "action_type": "GatherWood", "schema": { "amount": "integer" } },
    { "action_type": "BuildTeaStall", "schema": { "plot_id": "string" } },
    { "action_type": "BuyItem", "schema": { "item": "string", "qty": "integer", "vendor_id": "string" } },
    { "action_type": "RequestHelp", "schema": { "kind": "string", "detail": "string" } },
    { "action_type": "AskDirection", "schema": { "question": "string", "options": "array<string>" } }
  ],
  "decision_constraints": {
    "max_tasks_returned": 1,
    "response_timeout_seconds": 15
  }
}
```

**Response body**:
```json
{
  "protocol_version": "1.0",
  "request_id": "req_12345",
  "decision": {
    "action_type": "GatherWood",
    "parameters": { "amount": 20 },
    "reason": "Wood is the current bottleneck and gathering it is cheaper than buying it.",
    "priority": "high",
    "fallback": {
      "action_type": "BuyItem",
      "parameters": { "item": "wood", "qty": 10, "vendor_id": "npc_mira" }
    }
  }
}
```

**Decision timeout**: recommended 10–15s, hard max for MVP 30s.

---

### 9.3 Optional Endpoint: Heartbeat Receive

```
POST {endpoint_base_url}{heartbeat_path}
```

**Request**:
```json
{
  "protocol_version": "1.0",
  "agent_id": "agt_789",
  "timestamp": "2026-03-31T12:10:00Z",
  "type": "heartbeat"
}
```

**Response**:
```json
{
  "protocol_version": "1.0",
  "accepted": true
}
```

---

## 10. Signature Verification

Every game-to-agent request should be signed.

**Proposed signing method**: HMAC-SHA256 over:

```
timestamp + "\n" + request_id + "\n" + raw_body
```

Using: `platform_signing_secret` issued during claim.

**Headers**:
```
X-WOA-Timestamp: 2026-03-31T12:06:00Z
X-WOA-Request-Id: req_12345
X-WOA-Signature: sha256=<hex_digest>
```

**External agent must verify**:
- signature is valid
- timestamp is recent
- request_id is not replayed

**Replay protection**: reject requests older than 5 minutes.

---

## 11. Action Schema

The agent may only return allowed task-level commands.

### 11.1 MVP Action List

**Phase A (gate-1 checkpoint — locked to 3 verbs)**:
- `GatherWood` — agent gathers wood from forest nodes
- `BuyItem` — agent purchases items from NPC vendors at market
- `RequestHelp` — agent requests player intervention when blocked or needing resources

**Phase B+ (after BuildTeaStall checkpoint)**:
- `BuildTeaStall`
- `BuildStorage`
- `RepairStructure`
- `SellItem`
- `RestockBusiness`
- `HarvestCrop`
- `GatherStone`
- `RequestPermission`
- `AskDirection`
- `StartShift`
- `EndShift`

### 11.2 Phase A Action Definitions

```
GatherWood → { "amount": 20 }
BuyItem → { "item": "string", "qty": "integer", "vendor_id": "string" }
RequestHelp → { "kind": "resource_shortage", "detail": "string" }
```

### 11.3 Example Decision (Phase A)

```json
{
  "protocol_version": "1.0",
  "request_id": "req_12345",
  "decision": {
    "action_type": "GatherWood",
    "parameters": { "amount": 20 },
    "reason": "Wood stock is low. Gathering is faster than buying at current prices.",
    "priority": "high",
    "fallback": {
      "action_type": "BuyItem",
      "parameters": { "item": "wood", "qty": 10, "vendor_id": "npc_mira" }
    }
  }
}
```

---

## 12. Platform Validation Rules

Every decision must be validated before execution.

**Checks**:
- action type is allowed
- parameters match schema
- action is legal in current state
- action is possible given inventory, permissions, and world rules
- target exists and is accessible
- rate limit not exceeded

**If validation fails**:
- reject the decision
- optionally retry once by re-asking
- otherwise apply fallback or safe idle behavior

---

## 13. Failure Handling

### 13.1 Failure Categories
- `AGENT_TIMEOUT`
- `AGENT_UNREACHABLE`
- `INVALID_SIGNATURE`
- `INVALID_SCHEMA`
- `UNSUPPORTED_PROTOCOL_VERSION`
- `UNSUPPORTED_ACTION`
- `AUTH_EXPIRED`
- `RATE_LIMITED`

### 13.2 Platform Behavior

If a decision call fails:
1. log failure
2. increment `consecutive_failures`
3. retry once if failure is transient
4. use fallback if supplied
5. else use safe behavior: continue current task if safe, idle, rest, or wait for player direction

**Failure thresholds (Phase A)**:
- `consecutive_failures >= 2` → move agent to `DEGRADED`
- `consecutive_failures >= 5` → move agent to `DISCONNECTED`

In `DEGRADED`: continue calling agent but surface warning to player. In `DISCONNECTED`: stop outbound calls, safe-idle agent, notify player.

---

## 14. Idempotency

All decision requests must include `request_id`.

**Rules**:
- same `request_id` must not be executed twice
- replayed responses should be ignored
- result should be stored against the request

---

## 15. Rate Limits

Recommended MVP limits per agent:
- max 12 decision requests per minute
- max 1 active outstanding decision request at a time
- max response body size: 64 KB
- max heartbeat frequency: 1 per 30 seconds

---

## 16. Error Response Format

All platform errors should use:
```json
{
  "protocol_version": "1.0",
  "error": {
    "code": "INVALID_SCHEMA",
    "message": "Field 'action_type' is missing.",
    "retryable": false
  }
}
```

All external agent errors should use the same shape when possible.

---

## 17. Status Codes

| Code | Meaning |
|------|---------|
| 200 | Success |
| 201 | Invite or claim created |
| 400 | Malformed payload |
| 401 | Bad or missing auth |
| 403 | Ownership/scope violation |
| 404 | Missing agent or invite |
| 409 | Token already claimed / request replay |
| 422 | Schema valid but semantic validation fails |
| 429 | Rate limit exceeded |
| 500 | Platform error |
| 502 | External agent bad upstream response |
| 504 | External agent timed out |

---

## 18. Recommended Database Records

### 18.1 Agent Registration
- `agent_id`
- `owner_id`
- `town_id`
- `status`
- `display_name`
- `endpoint_base_url`
- `decision_path`
- `health_path`
- `heartbeat_path`
- `protocol_version`
- `platform_signing_secret_hash`
- `access_token_hash`
- `refresh_token_hash`
- `last_health_at`
- `last_decision_at`
- `consecutive_failures`

### 18.2 Invite
- `invite_id`
- `invite_token_hash`
- `owner_id`
- `town_id`
- `slot_name`
- `expires_at`
- `used_at`
- `claimed_agent_id`

### 18.3 Decision Log
- `request_id`
- `agent_id`
- `observation_snapshot`
- `decision_response`
- `validation_result`
- `execution_result`
- `created_at`

---

## 19. MVP Implementation Order

1. Create invite + claim flow
2. Store agent registration and credentials
3. Implement `GET /health`
4. Implement outbound `POST /decision`
5. Validate action schema
6. Execute one simple action family: `GatherWood`, `BuildTeaStall`, `RequestHelp`
7. Add retry + fallback logic
8. Add event log and player UI

---

## 20. Minimal MVP Contract

**Gate-1 (Phase A checkpoint) — support only these:**

**Platform endpoints**:
- `POST /v1/agent-invites`
- `POST /v1/agents/claim`
- `POST /v1/agents/{agent_id}/decision`
- `GET /v1/agents/{agent_id}/health`

**External agent endpoints**:
- `GET /health`
- `POST /decision`

**Deferred past checkpoint**: `POST /v1/auth/agent/refresh`, `GET /v1/agents/{agent_id}`, `POST /v1/agents/{agent_id}/disconnect`, `POST /v1/agents/{agent_id}/revoke`, heartbeat receive.

That is enough to prove:
- the player can connect a real external agent
- the game can ask it what to do
- the agent can live inside the world through structured tasks

---

## 21. One Important Practical Recommendation

For the first real prototype, **do not let the external agent pull arbitrary world data**.

Keep the pattern strictly:
- game pushes observation
- agent returns one task
- game executes
- game logs result

That is much easier to secure and debug.

---

## 22. Final Summary

This spec gives you a Paperclip-like token claim flow, adapted for a server-authoritative game world:
- one-time invite token
- external agent claims slot
- platform issues scoped credentials
- game calls agent at decision points
- agent returns task-level decision
- game validates and executes
- fallback handles failure safely
