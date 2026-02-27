# 13 — Rabbit Integration

**LIRAQ v1.0**

---

## 1 Purpose

This document describes how LIRAQ integrates with the **Rabbit protocol** — a
peer-to-peer, text-based, federated communication protocol using Ed25519
identities, TLS 1.3 transport, and a selector-based message routing system.

LIRAQ and Rabbit operate at different layers. Rabbit is a network communication
protocol. LIRAQ is a document presentation framework. They are complementary:
Rabbit can serve as a transport for DCS messages, a source of state data, and
an identity system for multi-agent coordination.

---

## 2 Layer Architecture

```
┌─────────────────────────────────────────────────────┐
│              Application Layer                      │
│   LIRAQ (document structure, state, behavior)       │
├─────────────────────────────────────────────────────┤
│              Session Layer                          │
│   DCS protocol (LCF queries/mutations)              │
├─────────────────────────────────────────────────────┤
│              Transport Layer                        │
│   Rabbit (P2P messaging, identity, federation)      │
├─────────────────────────────────────────────────────┤
│              Network Layer                          │
│   TLS 1.3 / TCP                                     │
└─────────────────────────────────────────────────────┘
```

LIRAQ never depends on Rabbit. The integration is optional. A LIRAQ runtime can
operate without Rabbit using any other transport (HTTP, pipes, function calls).

---

## 3 Integration Points

### 3.1 DCS Transport

Rabbit can carry DCS↔Runtime messages using Rabbit's **type `u`** (utility)
bundles:

```
Rabbit bundle:
  type: u
  selector: liraq/dcs/batch
  payload: <LCF message>
```

The Rabbit adapter (see [09-dcs-api §8.4](09-dcs-api.md)) wraps LCF messages
in Rabbit bundles and routes them through Rabbit's selector system.

#### Selector Namespace

LIRAQ reserves the selector prefix `liraq/` for its messages:

| Selector | Purpose |
|----------|---------|
| `liraq/dcs/batch` | DCS mutation batch |
| `liraq/dcs/query` | DCS query |
| `liraq/dcs/result` | Runtime response |
| `liraq/dcs/notify` | Runtime notification |
| `liraq/state/sync` | State synchronization (multi-peer) |
| `liraq/surface/announce` | Announcement broadcast |

### 3.2 State Synchronization

Rabbit's pub/sub model can synchronize state tree regions across peers:

```
Peer A modifies state → publishes to liraq/state/sync
                         → Peer B receives → merges into local state tree
```

#### Sync Configuration

State paths can be tagged for Rabbit synchronization in the schema:

```yaml
paths:
  shared.mission.status:
    type: string
    sync: true
    sync-scope: "fleet/*"   # Rabbit selector for sync targets
```

- `sync: true` — mutations to this path are published via Rabbit.
- `sync-scope` — Rabbit selector pattern for recipients.
- Incoming sync messages are validated against the local schema before merge.

### 3.3 Identity Mapping

Rabbit provides Ed25519 cryptographic identities. These map to DCS session
identity:

| Rabbit concept | LIRAQ mapping |
|---------------|---------------|
| Peer identity (Ed25519 public key) | `dcs-id` in handshake |
| Peer display name | `_user.name` in state tree |
| Peer trust level | DCS capability scope |
| Lane membership | Surface group membership |

A DCS connecting via Rabbit is authenticated by its Rabbit identity. The runtime
can scope capabilities based on the peer's trust level in the Rabbit network.

### 3.4 Multi-Agent Coordination

Rabbit's federated model enables multiple DCS instances across peers:

```
┌──────────┐     Rabbit      ┌──────────┐
│  DCS A   │◄───network────►│  DCS B   │
│(local AI)│                 │(remote)  │
└────┬─────┘                 └────┬─────┘
     │                            │
     ▼                            ▼
┌─────────────────────────────────────────┐
│            LIRAQ Runtime                │
│  (accepts mutations from both DCS)      │
└─────────────────────────────────────────┘
```

Each DCS has its own Rabbit identity and session. The journal tracks which
DCS (by Rabbit identity) authored each mutation.

### 3.5 Announcement Federation

Announcements can be federated across Rabbit peers:

```toml
[[batch]]
op = "announce"
message = "Fleet alert: sector 7 anomaly detected"
priority = "assertive"
federate = true
federate-scope = "fleet/*"
```

When `federate = true`, the announcement is published to peers matching
`federate-scope` via Rabbit. Receiving runtimes deliver the announcement to
their local surfaces.

---

## 4 UIDL Metadata for Rabbit

The `<meta>` element carries Rabbit-specific configuration:

```xml
<meta id="rabbit-config" key="rabbit-identity" value="Ed25519:abc123..." />
<meta id="rabbit-lane" key="rabbit-lane" value="bridge-console" />
<meta id="rabbit-sync" key="rabbit-sync-scope" value="fleet/*" />
```

These metadata elements are not projected onto any surface. They are read by the
Rabbit integration layer at initialization.

---

## 5 Rabbit-Bound State

State paths can bind to Rabbit subscriptions using a `rabbit:` prefix in bind
expressions:

```xml
<label id="fleet-status"
  bind="=fleet.status" />
```

The state path `fleet.status` is populated by Rabbit sync (§3.2). The
UIDL binding is unaware that the data arrives via Rabbit — it simply reads the
state tree.

This maintains a clean separation: Rabbit populates state, UIDL binds to state,
surfaces project UIDL. No component needs to know about the others' transport.

---

## 6 Security Considerations

### 6.1 Message Validation

All LCF messages received via Rabbit MUST be:

1. Parsed and validated as well-formed LCF.
2. Checked against the DCS session's capability scope.
3. Authenticated by the Rabbit peer's Ed25519 identity.
4. Schema-validated for state sync messages.

Malformed or unauthorized messages are rejected silently. A journal entry
records the rejection.

### 6.2 State Sync Conflicts

When state arrives from multiple Rabbit peers simultaneously:

- Last-writer wins (consistent with local multi-DCS behavior).
- The journal records the Rabbit peer identity for each sync mutation.
- A DCS can implement conflict resolution by reading the journal and applying
  corrective mutations.

### 6.3 Trust Scoping

Rabbit peer trust levels map to DCS capability scoping:

| Trust level | Capability scope |
|------------|-----------------|
| Local (same node) | Full access |
| Trusted peer | Scoped to declared state paths |
| Known peer | Read-only queries |
| Unknown peer | No access (rejected at handshake) |

Trust levels are configured per-runtime, not specified by LIRAQ. The Rabbit
integration layer consults the trust configuration when accepting DCS handshakes.

---

## 7 Federation Topology

### 7.1 Hub-Spoke

One runtime acts as the primary, others sync state:

```
Secondary ──Rabbit──► Primary Runtime ◄──Rabbit── Secondary
```

### 7.2 Mesh

All runtimes are peers, each with their own DCS:

```
Runtime A ◄──Rabbit──► Runtime B ◄──Rabbit──► Runtime C
```

State sync keeps shared paths consistent. Each runtime manages its own
surfaces, attention, and local DCS interactions.

### 7.3 Broadcast

One DCS controls multiple runtimes (e.g., fleet-wide alert):

```
        ┌──Rabbit──► Runtime A (bridge)
DCS ────┼──Rabbit──► Runtime B (engineering)
        └──Rabbit──► Runtime C (medical)
```

Each runtime receives the same mutations but projects onto its own local
surfaces with its own presentation profiles and accommodation settings.

---

## 8 Implementation Notes

- The Rabbit adapter is a platform adapter (see [09-dcs-api §8](09-dcs-api.md)).
  It translates between LCF messages and Rabbit bundles.
- Rabbit integration is optional. A conforming LIRAQ runtime is NOT required to
  support Rabbit.
- State sync is eventually consistent, not strongly consistent. Applications
  requiring strong consistency should implement their own coordination protocol
  on top of Rabbit.
- Rabbit lane membership can be reflected in the state tree under
  `_rabbit.lanes` for DCS inspection, but this path is implementation-specific
  and not standardized.
