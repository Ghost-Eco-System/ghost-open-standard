# GOS RFC-0002: Message Envelope

- **Status:** Stable v1.0    
- **Category:** Foundation
- **Author:** Ghost System Community    
- **Scope:** Protocol Syntax    
- **Applies to:** All Ghost-Compatible Systems    

## 1. Introduction & Context

### 1.1 Purpose

This RFC defines the standardized, universal communication envelope for the Ghost Open Standard (GOS). Every payload exchanging state, commands, configuration, or telemetry within the Ghost Ecosystem MUST be wrapped in this atomic structure, regardless of the underlying transport layer.

### 1.2 Motivation

To achieve true transport independence (MQTT, HTTP, BLE, Raw Serial), the messaging contract must be distinct from the transmission medium. By defining a strict envelope structure at the core level, consumers can safely parse routing details, protocol version compliance, and execution timestamps without needing any awareness of the high-level payload's functional intent. This decoupling ensures long-term schema evolution and absolute predictability for agentic code parsers.

## 2. Core Specification

### 2.1 Envelope Immutability & Extensibility

All structural fields defined at the root level of the Ghost Envelope MUST be treated as mandatory or strictly constrained optional primitives.

- Consumers MUST silently ignore unknown or experimental fields appended to the envelope root, rather than throwing compilation or runtime errors. This rule guarantees seamless forward and backward compatibility during minor version updates.
    
- The processing of any Ghost envelope MUST fail immediately if the root structure fails to pass the core schema validation defined in Section 3.
    

### 2.2 Strict Structural Rules

- **Routing Assertions:** The envelope MUST explicitly state its source identifier (`from`) and target scope (`to`). Broad multicasting or pub/sub topics do not waive this requirement; the envelope itself must carry its topological context.
    
- **Temporal Integrity:** The timestamp field MUST be populated at the exact moment of message creation by the producing node. It MUST use the standardized ISO 8601 extended UTC format.
    
- **Payload Isolation:** The actual data payload MUST be encapsulated entirely within a distinct root key named `payload`. Application logic or capability states are forbidden from leaking into the envelope root.
    

## 3. Architecture & Data Structures

### 3.1 JSON Schema Specification

The normative wire-format for the GOS Message Envelope is JSON. Below is the strict JSON Schema definition located in `core/schemas/envelope.json`:


```JSON
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "GOSMessageEnvelope",
  "type": "object",
  "required": ["gos", "from", "to", "timestamp", "payload"],
  "properties": {
    "gos": {
      "type": "string",
      "description": "The GOS specification version enforcing this message structure.",
      "pattern": "^[0-9]+\\.[0-9]+$"
    },
    "from": {
      "type": "string",
      "description": "The unique logical identity of the sending entity (GUID or Ghost MAC).",
      "pattern": "^[a-zA-Z0-9_\\-:\\.]+$"
    },
    "to": {
      "type": "string",
      "description": "The target identity, broadcast group, or logical service address.",
      "pattern": "^[a-zA-Z0-9_\\-:\\.\\*]+$"
    },
    "timestamp": {
      "type": "string",
      "format": "date-time",
      "description": "ISO 8601 UTC timestamp reflecting message generation time."
    },
    "meta": {
      "type": "object",
      "description": "Optional operational metadata. Reserved for trace IDs, routing telemetry, or security nonces.",
      "additionalProperties": true
    },
    "payload": {
      "type": "object",
      "description": "The isolated transactional container housing operational capabilities, commands, or telemetry.",
      "additionalProperties": true
    }
  },
  "additionalProperties": true
}
```

### 3.2 Wire-Format Example

A standard runtime compliance message adhering to GOS RFC-0002:

JSON

```
{
  "gos": "1.0",
  "from": "22:01:00:AF:12:34",
  "to": "platform.registry",
  "timestamp": "2026-06-20T21:56:12Z",
  "meta": {
    "trace_id": "9f8a7b6c-5e4d-3c2b-1a09-f8e7d6c5b4a3"
  },
  "payload": {
    "cap.distance": {
      "value": 0.42
    }
  }
}
```

## 4. Backwards Compatibility & Edge Cases

### 4.1 Strict Constraints on Malformed Inputs

If an incoming message violates the structural rules of this envelope (e.g., a missing `from` property or an invalid `gos` version layout):

- The receiving node MUST drop the packet immediately.
    
- It MUST NOT attempt to parse the embedded `payload`.
    
- If the transport layer supports bidirectionality, the receiver SHOULD emit a standardized GOS error payload (defined in RFC-0006) targeted at the offending entity, signaling a structural rejection.
    

### 4.2 Version Mismatch Handling

When a node running `gos` version `1.0` receives a valid envelope stamped with a higher version (e.g., `"gos": "1.1"`), it MUST parse the envelope using the rules of `1.0`. It will extract the known fields (`from`, `to`, `payload`) and safely ignore any new structural additions introduced by the newer minor version.