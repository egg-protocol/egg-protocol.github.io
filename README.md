# 🥚EGG — Elastic Gaming Grid Protocol
## Protocol Specification v0.1.0

**Author:** Edgar Paula  
**Affiliation:** Independent researcher, seminarian — Amapá, Brazil  
**Status:** Draft  
**Date:** April 2026  

---

## Abstract

EGG (Elastic Gaming Grid Protocol) is an open, decentralized protocol for the remote execution of interactive experiences — including games, educational modules, and lightweight applications — across a distributed network of compute nodes. EGG defines a relay-based event architecture, a client-node negotiation model, and a native economic layer using the Bitcoin Lightning Network for micro-payments. It is designed to operate under constrained network conditions, prioritizing minimal data consumption, offline-first resilience, and low-power device compatibility. EGG is an independent protocol, architecturally inspired by Nostr, but distinct in its event schema, execution semantics, and economic model.

---

## 1. Motivation

Access to interactive digital experiences — games, educational software, simulations — remains highly unequal across the global population. The primary barriers are:

1. **Hardware constraints**: Low-end devices lack the processing power to run modern applications locally.
2. **Connectivity constraints**: Rural and underserved regions have intermittent, low-bandwidth network access.
3. **Centralization**: Existing cloud gaming and remote execution platforms depend on proprietary infrastructure, creating single points of failure and economic exclusion.
4. **Lack of economic sovereignty**: Creators in developing regions have limited access to monetization infrastructure tied to traditional banking.

EGG addresses these constraints by providing:

- A protocol for remote execution that shifts compute burden to willing nodes in the network.
- An event architecture optimized for low-bandwidth transmission.
- A monetization layer using Bitcoin's Lightning Network, accessible to anyone with a compatible wallet, regardless of geography or banking access.
- An open specification that any developer, node operator, or organization may implement without permission.

---

## 2. Protocol Design

### 2.1 Overview

EGG operates through three primary roles:

| Role | Description |
|---|---|
| **Client** | End-user device (mobile, browser, embedded). Sends inputs, receives state. |
| **Node** | Compute provider. Executes game/application logic, publishes availability. |
| **Relay** | Message broker. Routes events between clients and nodes. Stateless. |

The protocol flow is:

```
Client → Relay → Node     (discovery, negotiation, input events)
Node   → Relay → Client   (state updates, output frames, payment requests)
```

No central authority coordinates this network. Relays are independently operated. Nodes self-publish their capabilities. Clients freely choose relays and nodes.

### 2.2 Transport Layer

EGG uses **WebSocket** as the default transport. This enables:

- Full-duplex communication.
- Wide compatibility across browsers and embedded systems.
- Low connection overhead compared to polling.

For environments where WebSocket is unavailable, EGG MAY fall back to HTTP long-polling. Relay implementors SHOULD support both.

### 2.3 Serialization

EGG events are serialized as **JSON**. Binary attachments (e.g., execution packets) are encoded in **base64** within the JSON envelope. This sacrifices some efficiency in exchange for universal compatibility and debuggability.

A future version of EGG MAY define a binary encoding (e.g., MessagePack or CBOR) for constrained environments.

---

## 3. Event Structure

All communication in EGG occurs through **Events**. An event is the atomic unit of the protocol.

### 3.1 Base Event Schema

```json
{
  "id":        "<32-byte hex string, SHA-256 of canonical fields>",
  "pubkey":    "<32-byte hex string, sender's public key>",
  "created_at": <Unix timestamp, integer>,
  "kind":      <integer, event type>,
  "tags":      [["<tag_name>", "<value>", "..."], ...],
  "content":   "<arbitrary string, kind-dependent>",
  "sig":       "<64-byte hex string, Schnorr signature over id>"
}
```

- **id**: Computed as `SHA-256(UTF-8(JSON([pubkey, created_at, kind, tags, content])))`. This makes the event self-authenticating and tamper-evident.
- **pubkey**: The sender's public key (secp256k1). Serves as identity.
- **sig**: Schnorr signature (BIP-340) over the event `id`.

### 3.2 Event Kinds

EGG reserves the following kind namespaces:

| Kind Range | Category | Description |
|---|---|---|
| `10000–10099` | **Node Announcement** | Node publishes compute availability |
| `10100–10199` | **Session Negotiation** | Client requests session; node responds |
| `10200–10299` | **Execution Events** | Input actions, state updates, frame deltas |
| `10300–10399` | **Economic Events** | Payment requests, receipts, invoices |
| `10400–10499` | **Control Events** | Pause, resume, terminate, error |
| `10500–10599` | **Packet Events** | Execution packet distribution, updates |
| `65535` | **Relay Metadata** | Relay self-description |

### 3.3 Node Announcement Event (kind: 10000)

Published periodically by nodes to advertise availability.

```json
{
  "kind": 10000,
  "content": "",
  "tags": [
    ["name", "node-amazonia-01"],
    ["capacity", "8"],
    ["games", "snake-v1", "pong-v2", "quiz-geo-br"],
    ["latency_region", "sa-east"],
    ["price_msat_per_tick", "10"],
    ["endpoint", "wss://node.example.com/egg"],
    ["expires", "1745000000"]
  ]
}
```

| Tag | Description |
|---|---|
| `name` | Human-readable node identifier |
| `capacity` | Maximum concurrent sessions |
| `games` | List of execution packet IDs the node can run |
| `latency_region` | Rough geographic hint for client-side routing |
| `price_msat_per_tick` | Cost per game tick in millisatoshis (0 = free) |
| `endpoint` | WebSocket URL for direct connection |
| `expires` | Unix timestamp after which this announcement is stale |

### 3.4 Session Request Event (kind: 10100)

Sent by a client to initiate a session.

```json
{
  "kind": 10100,
  "content": "",
  "tags": [
    ["game", "snake-v1"],
    ["node", "<node_pubkey>"],
    ["client_version", "egg-client/0.1"],
    ["payment_capability", "lightning"],
    ["max_budget_msat", "5000"]
  ]
}
```

### 3.5 Session Accept Event (kind: 10101)

Sent by a node in response to a valid session request.

```json
{
  "kind": 10101,
  "content": "",
  "tags": [
    ["session_id", "<uuid-v4>"],
    ["e", "<session_request_event_id>"],
    ["p", "<client_pubkey>"],
    ["endpoint", "wss://node.example.com/egg/session/<session_id>"],
    ["tick_rate_ms", "100"],
    ["price_msat_per_tick", "10"]
  ]
}
```

### 3.6 Input Event (kind: 10200)

Sent by a client during an active session.

```json
{
  "kind": 10200,
  "content": "{\"action\": \"move\", \"direction\": \"up\"}",
  "tags": [
    ["session_id", "<uuid-v4>"],
    ["tick", "4821"]
  ]
}
```

The `content` field is a JSON-encoded action object. Its schema is defined by the execution packet (the game).

### 3.7 State Update Event (kind: 10201)

Sent by a node after each game tick.

```json
{
  "kind": 10201,
  "content": "<base64-encoded delta or full state>",
  "tags": [
    ["session_id", "<uuid-v4>"],
    ["tick", "4821"],
    ["encoding", "delta_json"],
    ["checksum", "<sha256_of_full_state>"]
  ]
}
```

EGG RECOMMENDS delta encoding to minimize bandwidth. Full-state snapshots MAY be sent at configurable intervals for resynchronization.

### 3.8 Payment Request Event (kind: 10300)

Sent by a node to request payment for a session interval.

```json
{
  "kind": 10300,
  "content": "<BOLT-11 Lightning invoice>",
  "tags": [
    ["session_id", "<uuid-v4>"],
    ["ticks", "100"],
    ["amount_msat", "1000"],
    ["expires", "1745001000"]
  ]
}
```

### 3.9 Payment Receipt Event (kind: 10301)

Sent by a client to acknowledge payment.

```json
{
  "kind": 10301,
  "content": "<payment_preimage_hex>",
  "tags": [
    ["session_id", "<uuid-v4>"],
    ["e", "<payment_request_event_id>"]
  ]
}
```

---

## 4. Relay Architecture

### 4.1 Role of Relays

EGG relays are **stateless message brokers**. They:

- Accept WebSocket connections from clients and nodes.
- Receive events and route them to subscribers.
- Persist events for a configurable TTL (Time To Live).
- Do NOT execute game logic.
- Do NOT hold funds.

### 4.2 Relay Message Protocol

```
# Client → Relay
["SUB",  <subscription_id>, <filter>]   // Subscribe to events
["PUB",  <event>]                        // Publish an event
["UNSUB", <subscription_id>]             // Cancel subscription

# Relay → Client
["EVENT", <subscription_id>, <event>]    // Deliver an event
["OK",    <event_id>, true|false, "msg"] // Acknowledge publication
["NOTICE", "message"]                    // Relay-level notice
["EOSE",  <subscription_id>]             // End of stored events
```

### 4.3 Filter Schema

```json
{
  "kinds":   [10000, 10101],
  "authors": ["<pubkey>"],
  "since":   <unix_timestamp>,
  "until":   <unix_timestamp>,
  "#session_id": ["<uuid>"],
  "limit":   100
}
```

### 4.4 Relay Discovery

Relays MAY publish their own metadata as kind `65535`:

```json
{
  "kind": 65535,
  "content": "",
  "tags": [
    ["name", "Relay Amazônia"],
    ["description", "EGG relay operated in northern Brazil"],
    ["supported_kinds", "10000-10599"],
    ["max_event_size_bytes", "65536"],
    ["retention_hours", "24"]
  ]
}
```

Clients SHOULD maintain a list of known relays and connect to multiple simultaneously for redundancy.

### 4.5 Relay Interoperability with Nostr

EGG relays MAY optionally relay Nostr-compatible events (kind numbers outside the EGG namespace) if the operator chooses. This is NOT required by the protocol. EGG kind numbers are chosen to avoid collision with existing Nostr kinds.

---

## 5. Economic Layer

### 5.1 Payment Model

EGG uses the **Bitcoin Lightning Network** for all economic activity. Lightning is chosen because:

- Payments settle in milliseconds.
- Transaction fees are negligible (sub-satoshi routing fees).
- It is permissionless and globally accessible.
- No custodian holds user funds.

The unit of account is the **millisatoshi (msat)**: one thousandth of a satoshi.

### 5.2 Pay-Per-Tick Model

Nodes charge clients per **tick** — a discrete unit of game execution (typically 100ms). This enables:

- Granular billing proportional to actual usage.
- Sessions that can be paused without accumulating debt.
- Transparent cost visibility before and during play.

### 5.3 Payment Flow

```
1. Node publishes price_msat_per_tick in announcement (kind 10000).
2. Client accepts price during session negotiation (kind 10100).
3. Node executes N ticks, then issues a Lightning invoice (kind 10300).
4. Client pays the invoice via their Lightning wallet.
5. Client sends payment preimage as proof (kind 10301).
6. Node verifies preimage and continues execution.
```

If a client fails to pay within the invoice expiry window, the node MAY suspend or terminate the session (kind 10400).

### 5.4 Free Tier

Nodes MAY set `price_msat_per_tick` to `0`, offering free sessions. This is suitable for:

- Public educational content.
- Node operators subsidizing community access.
- Development and testing environments.

### 5.5 Creator Revenue

Execution packets (games, lessons) MAY embed a **creator royalty** in their metadata. Nodes that run these packets SHOULD route a configurable percentage of collected sats to the creator's Lightning address. Enforcement is social/reputational in v0.1; cryptographic enforcement is a planned future extension.

---

## 6. Execution Packets

### 6.1 Definition

An **Execution Packet** is a self-contained, versioned bundle that defines a game or application. It is the deployable unit of EGG.

### 6.2 Packet Structure

```json
{
  "id": "snake-v1",
  "version": "1.0.0",
  "name": "Snake",
  "author_pubkey": "<creator_pubkey>",
  "creator_lightning_address": "edgar@getalby.com",
  "royalty_pct": 10,
  "engine": "egg-wasm-v1",
  "entry": "snake.wasm",
  "assets": ["sprites.bin"],
  "tick_rate_ms": 100,
  "state_schema": "https://egg.protocol/schemas/snake-v1-state.json",
  "action_schema": "https://egg.protocol/schemas/snake-v1-action.json",
  "checksum": "<sha256_of_bundle>"
}
```

### 6.3 Execution Engines

EGG v0.1 defines one execution engine:

- **egg-wasm-v1**: Executes a WebAssembly module with a standardized ABI. The WASM module exports `tick(action_json) → state_delta_json`.

Future engines may support Lua, a custom bytecode VM, or sandboxed JavaScript.

### 6.4 Packet Distribution

Nodes MAY publish packet availability events (kind 10500). Clients and other nodes MAY subscribe to discover new packets. Packets are distributed as-is; integrity is verified via `checksum`.

---

## 7. Client Model

### 7.1 Client Types

| Type | Description | Typical Constraints |
|---|---|---|
| **Browser Client** | Web app running in a modern browser | WebSocket, WebAssembly support |
| **Mobile Client** | Native Android/iOS app | Intermittent connectivity, battery |
| **Lightweight Client** | Embedded or feature phone | HTTP only, no WASM, minimal RAM |
| **CLI Client** | Developer/testing tool | Full protocol access via terminal |

### 7.2 Offline-First Design

EGG clients SHOULD:

- Cache the last known game state locally.
- Queue input events during disconnection.
- Attempt automatic reconnection with exponential backoff.
- Resume sessions using the `session_id` and last verified `checksum`.

Nodes SHOULD accept resumption requests for sessions within a grace period (default: 300 seconds after disconnect).

### 7.3 Minimal Client Requirements

A conformant EGG client MUST implement:

1. WebSocket connection to at least one relay.
2. Event signing (secp256k1 / Schnorr).
3. Subscription and event parsing.
4. Rendering of state updates from the node.
5. Lightning invoice payment (via deep link, QR, or embedded wallet).

---

## 8. Security Considerations

### 8.1 Event Authentication

Every event is signed with a secp256k1 Schnorr signature. Relays and nodes MUST reject events with invalid signatures. This prevents impersonation and event forgery.

### 8.2 Denial of Service

Relays are exposed to spam and flooding. Mitigations include:

- **Rate limiting** by source pubkey and IP.
- **Proof-of-work** requirements (similar to Nostr NIP-13) for resource-intensive subscriptions.
- **Allowlists** for node pubkeys on private relays.

### 8.3 Node Trust

Clients SHOULD NOT unconditionally trust nodes. Mitigations:

- Verify execution state checksums against locally cached snapshots.
- Use multiple independent nodes for the same session (future: consensus mode).
- Maintain a pubkey-based reputation registry.

### 8.4 Payment Atomicity

Lightning payments are atomic at the protocol level. However, EGG v0.1 does not guarantee that a node will honor a paid session if it crashes immediately after payment. Clients SHOULD:

- Only pay for short tick intervals (100–500 ticks maximum per invoice).
- Maintain payment receipts (preimages) as dispute evidence.

### 8.5 Packet Integrity

Clients and nodes MUST verify the SHA-256 checksum of execution packets before loading. Nodes SHOULD run packets in a sandboxed environment (WASM sandbox) to prevent malicious code from affecting the host system.

### 8.6 Privacy

EGG uses public keys as identifiers. Users who wish for session privacy SHOULD generate ephemeral key pairs per session. Relays see traffic metadata (timing, volume) even if they cannot read encrypted content; users in sensitive environments SHOULD use relays over Tor or similar transports.

---

## 9. Comparison with Nostr

EGG is architecturally inspired by Nostr but is a distinct protocol. The following table summarizes the key differences:

| Dimension | Nostr | EGG |
|---|---|---|
| **Primary purpose** | Social messaging and content | Game/app remote execution |
| **Event kinds** | 0–65535 (open) | 10000–10599 (reserved, scoped) |
| **Event content** | Arbitrary text/JSON | Typed by kind; binary payloads via base64 |
| **Economic layer** | None (optional, external) | Native Lightning micro-payment per tick |
| **Session concept** | None | Stateful sessions with session_id |
| **State management** | Stateless (events are facts) | Stateful (node holds execution state) |
| **Execution** | None | WASM execution packets on nodes |
| **Delta encoding** | Not defined | Required for state updates |
| **Offline resilience** | Not specified | Explicit reconnection and queue protocol |
| **Relay compatibility** | Nostr relays | EGG relays (optional Nostr compatibility) |

EGG deliberately reuses Nostr's cryptographic primitives (secp256k1, Schnorr, SHA-256) and its relay-client WebSocket framing pattern, as these are well-specified and widely implemented. EGG does NOT define itself as a Nostr NIP and is not bound by the Nostr specification process.

---

## 10. Future Extensions

The following features are deferred to future protocol versions:

| Feature | Description |
|---|---|
| **Multiplayer sessions** | Multiple clients sharing one node-executed session |
| **Consensus nodes** | Two or more nodes executing the same session; majority state wins |
| **Creator royalty enforcement** | Cryptographic commitment to royalty routing at payment time |
| **Binary encoding** | MessagePack or CBOR serialization for constrained devices |
| **Encrypted content** | NaCl box encryption for private session state |
| **Packet marketplace** | Signed, discoverable registry of execution packets |
| **Reputation system** | On-chain or relay-based scoring for nodes and relays |
| **eSIM / SMS transport** | Ultra-lightweight transport for feature phones without data |

---

## Appendix A: Canonical Event Serialization

To compute the event `id`, serialize the following fields in order as a JSON array, UTF-8 encoded, with no whitespace:

```
[pubkey, created_at, kind, tags, content]
```

Example:

```
["abc123...", 1745000000, 10000, [["name","node-01"]], ""]
```

Apply SHA-256 to the UTF-8 bytes of this string. The result is the event `id`.

---

## Appendix B: Glossary

| Term | Definition |
|---|---|
| **Client** | End-user software that connects to relays and nodes |
| **Node** | Compute provider that executes game logic |
| **Relay** | Stateless message broker |
| **Event** | Signed, immutable unit of protocol communication |
| **Tick** | One cycle of game state computation (~100ms) |
| **Execution Packet** | Self-contained bundle defining a game or app |
| **msat** | Millisatoshi; 1/1000th of a satoshi |
| **Session** | A stateful relationship between one client and one node |
| **Preimage** | Proof of Lightning payment |
| **Delta** | Partial state update (only changes since last tick) |

---

*EGG Protocol Specification v0.1.0 — Edgar Paula — Amapá, Brazil — April 2026*  
*This document is released into the public domain. Implementations are free to adopt, fork, and extend without restriction.*
