# 13 — Transport Compatibility

**LIRAQ v1.0**

---

## 1 Purpose

LIRAQ specifies document structure, state management, presentation, and
message semantics — but treats the transport layer as a deployment concern.
This document defines what LIRAQ requires from a transport, evaluates how
candidate transports meet those requirements, and describes specific bindings
for each.

LIRAQ MUST NOT depend on any single transport. An Explorer communicating over
Rabbit, HTTP/WebSocket, gRPC, or local IPC is equally conformant, so long as
the transport meets the requirements in §2.

---

## 2 Transport Requirements

| Requirement | Why | Priority |
|-------------|-----|----------|
| Reliable, ordered delivery | Mutation batches must arrive intact and in order | MUST |
| Bidirectional messaging | DCS→Runtime requests + Runtime→DCS notifications | MUST |
| Session establishment | DCS handshake, capability declaration, session lifecycle | MUST |
| Async notification push | Runtime pushes state-change and event notifications without polling | MUST |
| Serialization negotiation | Transport must carry the agreed serialization (JSON, TOML, etc.) | MUST |
| Identity model | Multi-agent DCS coordination needs authenticated identity | SHOULD |
| Human readability | Debugging, logging, auditability of messages on the wire | SHOULD |
| Federation | Multi-peer, multi-Explorer coordination | MAY |
| Pub/sub | State synchronization across peers | MAY |

---

## 3 Candidate Transports

### 3.1 Rabbit

[Rabbit](https://github.com/dgoldman0/Rabbit/) is a peer-to-peer, text-based,
federated communication protocol using Ed25519 identities, TLS 1.3 transport,
selector-based message routing, async lanes (multiplexed channels), and native
pub/sub with replay. Its wire format is human-readable UTF-8 frames.

| Requirement | Fit |
|-------------|-----|
| Reliable, ordered delivery | Yes — TCP/TLS, ordered within lanes |
| Bidirectional messaging | Yes — native bidirectional |
| Session establishment | Yes — Rabbit handshake with identity exchange |
| Async notification push | Yes — native push via lanes |
| Serialization negotiation | Partial — frames carry UTF-8 text; structured body type negotiation is an open Rabbit-side enhancement |
| Identity model | Yes — Ed25519 cryptographic identities |
| Human readability | Yes — text-based frames, human-readable by design |
| Federation | Yes — built-in peer federation |
| Pub/sub | Yes — native pub/sub with replay |

**Strengths:** Most naturally aligned transport. Selector routing maps to LIRAQ
resource paths. Lane multiplexing maps to DCS sessions. Pub/sub with replay
maps to state synchronization. Ed25519 identity maps to multi-agent
coordination. Human-readable wire format aids debugging.

**Gaps:** Not ubiquitous — requires Rabbit-aware peers. No browser-native
client; web-hosted Explorers need a WebSocket bridge. Structured body type
negotiation (to distinguish TOML vs. JSON payloads) is not yet formalized in
the Rabbit spec.

### 3.2 HTTP / WebSocket

| Requirement | Fit |
|-------------|-----|
| Reliable, ordered delivery | Yes — TCP, ordered per connection |
| Bidirectional messaging | Yes — WebSocket (HTTP alone is request/response) |
| Session establishment | Application-layer (cookies, tokens, custom handshake) |
| Async notification push | Yes — WebSocket push; also SSE for unidirectional |
| Serialization negotiation | Yes — `Content-Type` header |
| Identity model | Application-layer (bearer tokens, mTLS, custom) |
| Human readability | Depends on serialization (JSON is semi-readable) |
| Federation | No — must be layered on top |
| Pub/sub | No — must be layered (WebSocket rooms, SSE streams) |

**Strengths:** Universal. Every browser, every language, every proxy, every
CDN. WebSocket provides full-duplex push. Massive ecosystem of libraries,
tools, and infrastructure.

**Gaps:** No native identity model — authentication is application-layer.
No native pub/sub — session management, room abstraction, and replay must be
implemented. Stateless by default (HTTP); WebSocket adds statefulness but
requires lifecycle management.

### 3.3 gRPC

| Requirement | Fit |
|-------------|-----|
| Reliable, ordered delivery | Yes — HTTP/2, ordered per stream |
| Bidirectional messaging | Yes — bidirectional streaming |
| Session establishment | Yes — HTTP/2 connection + custom metadata |
| Async notification push | Yes — server streaming |
| Serialization negotiation | Protobuf-native; JSON transcoding available |
| Identity model | mTLS, token metadata |
| Human readability | No — binary wire format (Protobuf) |
| Federation | No — must be layered |
| Pub/sub | No — must be layered |

**Strengths:** Strong typing, code generation, efficient binary encoding,
bidirectional streaming.

**Gaps:** Binary format is not human-readable. Heavy toolchain. Browser support
limited to grpc-web (unary and server streaming only; no client or
bidirectional streaming). Overkill for the message complexity LIRAQ requires.

### 3.4 Local Pipes / IPC

| Requirement | Fit |
|-------------|-----|
| Reliable, ordered delivery | Yes — kernel-guaranteed on same machine |
| Bidirectional messaging | Yes — Unix sockets, named pipes |
| Session establishment | Trivial — process identity |
| Async notification push | Yes — file descriptor readability |
| Serialization negotiation | Application-layer |
| Identity model | OS-level (UID, process credentials) |
| Human readability | Depends on serialization |
| Federation | No — single machine only |
| Pub/sub | No |

**Strengths:** Zero network latency. Trivial to implement. Ideal for
same-machine DCS (e.g., a local language model controlling the runtime).

**Gaps:** Single-machine only. No federation, no multi-peer coordination.

---

## 4 The HTML/HTTP Parallel

HTML and HTTP were developed hand-in-hand but specified separately. LIRAQ and
Rabbit share a similar relationship:

- LIRAQ specifies the document model, runtime semantics, and message format.
- Rabbit specifies the transport protocol.
- This document evaluates fit and describes specific bindings.

Rabbit is the most naturally aligned transport — its human-readable frames,
lane multiplexing, pub/sub with replay, and Ed25519 identity model were
designed with the same philosophy. But natural alignment does not imply
dependency. An Explorer communicating over plain WebSocket is equally
conformant.

---

## 5 Rabbit Binding

When the transport is Rabbit, the following binding applies.

### 5.1 Selector Namespace

LIRAQ reserves the selector prefix `liraq/` for its messages:

| Selector | Purpose |
|----------|---------|
| `liraq/dcs/batch` | DCS mutation batch |
| `liraq/dcs/query` | DCS query |
| `liraq/dcs/result` | Runtime response |
| `liraq/dcs/notify` | Runtime notification |
| `liraq/state/sync` | State synchronization (multi-peer) |
| `liraq/surface/announce` | Announcement broadcast |

### 5.2 Frame Mapping

DCS messages are carried in Rabbit **type `u`** (utility) bundles:

```
Rabbit bundle:
  type: u
  selector: liraq/dcs/batch
  payload: <DCS message in TOML>
```

TOML is the natural serialization for Rabbit (both are human-readable,
text-based). JSON payloads are also valid if peers negotiate it.

### 5.3 Identity Mapping

| Rabbit concept | LIRAQ mapping |
|---------------|---------------|
| Peer identity (Ed25519 public key) | `dcs-id` in handshake |
| Peer display name | `_user.name` in state tree |
| Peer trust level | DCS capability scope |
| Lane membership | Surface group membership |

A DCS connecting via Rabbit is authenticated by its Rabbit identity. The
runtime can scope capabilities based on the peer's trust level.

### 5.4 State Synchronization via Pub/Sub

Rabbit's pub/sub model can synchronize state tree regions across peers:

```
Peer A modifies state → publishes to liraq/state/sync
                         → Peer B receives → merges into local state tree
```

State paths can be tagged for synchronization in the schema:

```yaml
paths:
  shared.mission.status:
    type: string
    sync: true
    sync-scope: "fleet/*"   # Rabbit selector for sync targets
```

Incoming sync messages are validated against the local schema before merge.
Sync is eventually consistent (last-writer wins). The journal records the
Rabbit peer identity for each sync mutation.

### 5.5 Federation Topologies

#### Hub-Spoke

One runtime acts as the primary, others sync state:

```
Secondary ──Rabbit──► Primary Runtime ◄──Rabbit── Secondary
```

#### Mesh

All runtimes are peers, each with their own DCS:

```
Runtime A ◄──Rabbit──► Runtime B ◄──Rabbit──► Runtime C
```

#### Broadcast

One DCS controls multiple runtimes:

```
        ┌──Rabbit──► Runtime A (bridge)
DCS ────┼──Rabbit──► Runtime B (engineering)
        └──Rabbit──► Runtime C (medical)
```

### 5.6 Announcement Federation

Announcements can be federated across Rabbit peers:

```json
{
  "op": "announce",
  "message": "Fleet alert: sector 7 anomaly detected",
  "priority": "assertive",
  "federate": true,
  "federate-scope": "fleet/*"
}
```

When `federate` is true, the announcement is published to peers matching
`federate-scope`. Receiving runtimes deliver the announcement to their local
surfaces.

---

## 6 HTTP/WebSocket Binding

When the transport is HTTP/WebSocket, the following binding applies.

### 6.1 Endpoints

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `POST /liraq/dcs` | HTTP | DCS request (query or mutation batch) |
| `GET /liraq/dcs/ws` | WebSocket upgrade | Bidirectional DCS session |
| `GET /liraq/events` | SSE | Unidirectional notification stream (fallback) |

### 6.2 WebSocket Session

The primary mode. After WebSocket upgrade:

1. DCS sends handshake message (JSON).
2. Runtime responds with capabilities.
3. Bidirectional message exchange begins.
4. DCS sends queries and mutations; runtime sends responses and notifications.

JSON is the natural serialization for HTTP/WebSocket. TOML is also valid if
peers negotiate it via a `serialization` field in the handshake.

### 6.3 Authentication

Identity is application-layer:

| Method | Mechanism |
|--------|-----------|
| Bearer token | `Authorization` header or handshake `auth-token` field |
| mTLS | Client certificate at TLS layer |
| Custom | Application-defined handshake fields |

The DCS API handshake (see [09-dcs-api](09-dcs-api.md) §8) carries identity
regardless of authentication mechanism.

### 6.4 Multi-Agent Coordination

Without Rabbit's native federation, multi-agent coordination over HTTP/WS
requires application-layer orchestration:

- Each DCS connects via its own WebSocket session.
- The runtime tracks sessions by `dcs-id` from the handshake.
- State synchronization across runtimes requires an application-layer
  coordinator (e.g., a separate sync service).

---

## 7 Rabbit-Side Accommodations (Non-Normative)

Rabbit may evolve to better serve LIRAQ. Potential changes on the Rabbit side
(tracked in the Rabbit repo, not here):

- **First-class LIRAQ selector prefix.** Reserve `liraq/` in the selector
  namespace so LIRAQ messages route without collision.
- **Structured body type.** A formal "structured message" body type with
  content-type negotiation would let Rabbit carry TOML or JSON payloads
  without treating them as opaque text.
- **Session resume.** Rabbit's lane model could support DCS session
  continuity across reconnects, aligning with the LIRAQ handshake protocol.

These are suggestions for the Rabbit spec, not LIRAQ requirements.

---

## 8 Security

### 8.1 Message Validation

All DCS messages received via any transport MUST be:

1. Parsed and validated against the DCS message format
   (see [01-formats](01-formats.md) §2).
2. Checked against the DCS session's capability scope.
3. Authenticated by the transport's identity mechanism.
4. Schema-validated for state sync messages.

Malformed or unauthorized messages are rejected. A journal entry records the
rejection.

### 8.2 Trust Scoping

Trust levels map to DCS capability scoping:

| Trust level | Capability scope |
|------------|-----------------|
| Local (same process) | Full access |
| Authenticated peer | Scoped to declared state paths |
| Known peer | Read-only queries |
| Unknown peer | No access (rejected at handshake) |

Trust level determination is transport-specific (Rabbit: Ed25519 trust graph;
HTTP/WS: bearer token or mTLS; IPC: OS process credentials).

---

## 9 Conformance

- A conforming LIRAQ runtime MUST support at least one transport binding.
- A conforming runtime MUST NOT require any specific transport; the choice is
  a deployment decision.
- Rabbit integration is OPTIONAL. HTTP/WebSocket integration is OPTIONAL.
  Both are RECOMMENDED.
- Transport adapters are platform adapters
  (see [09-dcs-api](09-dcs-api.md) §8).
