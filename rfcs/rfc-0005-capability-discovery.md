# GOS RFC-0005: Capability Discovery

- **Status:** Stable v1.0    
- **Category:** Foundation    
- **Author:** Ghost System Community    
- **Scope:** Protocol Interaction    
- **Applies to:** All Ghost-Compatible Systems    

## 1. Introduction & Context

### 1.1 Purpose

This RFC defines the standardized discovery protocol, network lifecycle states, and orchestration behaviors of the Ghost Open Standard (GOS). It formalizes how decentralized nodes announce their presence, broadcast their internal capability matrices, and maintain active status across an agnostic network mesh.

### 1.2 Motivation

A robust distributed ecosystem must eliminate manual device provisioning. Intermediate brokers or registries must be able to dynamically discover and adapt to any physical device upon connection. Previous lifecycle designs relied on tight transport coupling (MQTT-specific Retain flags) and mixed the device registration lifecycle with operational business telemetry. This RFC decouples discovery from the transport layer, standardizing explicit self-announcements and explicit discovery requests to maximize network agility and support autonomous agent orchestration.

## 2. Core Specification

### 2.1 The Lifecyle State Machine

Every Ghost-compliant node MUST transition through three distinct operational network states:

Plaintext

```
┌──────────────┐      Boot / Reconnect
│   OFFLINE    ├─────────────────────────────┐
└──────▲───────┘                             ▼
       │                              ┌──────────────┐
       │ Device Clean Disconnect      │  ANNOUNCING  │ (Transmits full capability profile)
       │                              └──────┬───────┘
       │                                     │
       │                                     ▼
┌──────┴───────┐      Interval Tick   ┌──────────────┐
│    ONLINE    │◄─────────────────────┤    ONLINE    │ (Enters operational loop)
└──────┬───────┘                      └──────▲───────┘
       │                                     │
       └─────────────────────────────────────┘
                 Emits Alive Heartbeat
```

### 2.2 Normative Network Behaviors

- **The Self-Announce Rule:** Upon boot, hardware initialization, or reconnection to a network interface, a node MUST immediately broadcast a self-announcement payload exactly once. This payload contains the node's identity and its full capability profile.
    
- **The Discovery Query Response:** A node MUST monitor incoming network channels for a standardized discovery trigger command (`cmd.node.discover_capabilities`). Upon receipt of this targeted or wildcard command, the node MUST reply with its full profile within a randomized window (0 to 500ms) to avoid network collision storms.
    
- **The Alive Heartbeat Constancy:** While online, a node MUST emit a lightweight heartbeat message at a strict, pre-configured time interval (the default is every 60 seconds) to maintain its active lease within platform registries.
    

## 3. Architecture & Data Structures

### 3.1 Self-Announcement Payload Structure

The envelope payload for a self-announcement MUST use the `cmd.node.announce` keyword. It encapsulates both the identity metadata (RFC-0003) and the capability schemas (RFC-0004):

```JSON
{
  "gos": "1.0",
  "from": "22:01:00:AF:12:34",
  "to": "platform.registry",
  "timestamp": "2026-06-20T22:30:00Z",
  "payload": {
    "cmd.node.announce": {
      "identity": {
        "guid": "f81d4fae-7dec-11d0-a765-00a0c91e6bf6",
        "ghost_mac": "22:01:00:AF:12:34",
        "device_type": "FIELD_SENSOR",
        "hardware_revision": "v2.1.0",
        "firmware_version": "1.0.4"
      },
      "capabilities": {
        "cap.distance": {
          "type": "float",
          "access_mode": "READ",
          "interaction_mode": "EVENT_ONLY",
          "state_model": "STATEFUL",
          "unit": "m"
        }
      }
    }
  }
}
```

### 3.2 Alive Heartbeat Payload Structure

The periodic heartbeat MUST omit the heavy capability definitions to optimize bandwidth. It utilizes the `status.node.alive` container:


```JSON
{
  "gos": "1.0",
  "from": "22:01:00:AF:12:34",
  "to": "platform.registry",
  "timestamp": "2026-06-20T22:31:00Z",
  "payload": {
    "status.node.alive": {
      "uptime_seconds": 60,
      "rssi_dbm": -65,
      "free_heap_bytes": 184320
    }
  }
}
```

## 4. Backwards Compatibility & Edge Cases

### 4.1 Missing Heartbeat Lease Expiry

Platform engines and central brokers MUST maintain a dynamic lease timer for each discovered node. If a node fails to emit an `status.node.alive` message for two consecutive heartbeat intervals (120 seconds by default), the platform MUST flag the node as `STALE` or `OFFLINE` and notify connected business layers.

### 4.2 Dynamic Capability Mutations

If a hot-pluggable modular node alters its physical capability profile during runtime (e.g., adding an extension board), the node MUST execute a new `cmd.node.announce` sequence dynamically without requiring a hard hardware reboot. Registries MUST completely overwrite the prior capability cache for that node upon receiving the new announcement.