# GOS RFC-0006: Error Contract

- **Status:** Stable v1.0    
- **Category:** Foundation    
- **Author:** Ghost System Community    
- **Scope:** Protocol Syntax    
- **Applies to:** All Ghost-Compatible Systems    

## 1. Introduction & Context

### 1.1 Purpose

This RFC defines the standardized, universal contract for anomaly signaling, diagnostic reporting, and fault telemetry within the Ghost Open Standard (GOS). It establishes a rigid structural contract for the root-level error container while ensuring flexible metadata extension properties for hardware-specific diagnostics.

### 1.2 Motivation

In large-scale distributed IoT deployments, silent failures and inconsistent error formats are primary drivers of system downtime. Previous iterations of the protocol either omitted error states from the core schema or enforced a rigid, over-specified error dictionary that could not adapt to diverse hardware constraints (e.g., distinguishing an ultrasonic sensor blind-zone fault from an ESP32 low-voltage brownout). This specification resolves these limitations by guaranteeing that a standardized `error` block is present during failures, while leaving specific diagnostic telemetry open to vendor extension.

## 2. Core Specification

### 2.1 The Mandatory Error Root Rule

Whenever a Ghost-compliant node encounters a functional exception, a hardware fault, or a malformed input processing failure, its response message or outbound telemetry envelope MUST contain a distinct root key within the `payload` named `error`.

- If the `error` key is present, the message MUST be interpreted by all consumers as an anomalous or failed state transaction.    
- The presence of the `error` block overrides any operational capabilities transmitted within the same transaction envelope.    

### 2.2 Structural Rigidity vs. Diagnostic Flexibility

To maintain compatibility with automated agents and platform monitoring tools, the core error object MUST provide three universal primitives: a standardized domain-scoped error code (`code`), a human-readable diagnostic description (`message`), and an error classification level (`severity`).

- Beyond these three mandatory properties, the error schema MUST allow an optional, open-ended metadata object (`meta`). This object provides a safe area for custom hardware-specific diagnostics, raw register dumps, or stack traces without breaking core protocol parsers.    

## 3. Architecture & Data Structures

### 3.1 JSON Schema Error Specification

The normative structure for anomaly signaling within the GOS is defined below in the schema file `core/schemas/error.json`:


```JSON
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "GOSErrorContract",
  "type": "object",
  "required": ["error"],
  "properties": {
    "error": {
      "type": "object",
      "required": ["code", "message", "severity"],
      "properties": {
        "code": {
          "type": "string",
          "pattern": "^ERR_[A-Z0-9_]+(\\.[A-Z0-9_]+)*$",
          "description": "Standardized uppercase dot-separated error identifier code."
        },
        "message": {
          "type": "string",
          "description": "Human-readable, clear diagnostic message detailing the exception."
        },
        "severity": {
          "type": "string",
          "enum": ["WARN", "CRITICAL", "FATAL"],
          "description": "The operational impact level of the reported anomaly."
        },
        "meta": {
          "type": "object",
          "description": "Vendor-specific telemetry payload for raw debugging logs, component IDs, or hardware traces.",
          "additionalProperties": true
        }
      },
      "additionalProperties": false
    }
  },
  "additionalProperties": true
}
```

### 3.2 Wire-Format Compliance Examples

#### Example A: Standard Core-Compliant Error (Hardware Fault)

An example of an ESP32 node reporting an out-of-bounds read error on its distance capability:


```JSON
{
  "gos": "1.0",
  "from": "22:01:00:AF:12:34",
  "to": "platform.registry",
  "timestamp": "2026-06-20T22:45:00Z",
  "payload": {
    "error": {
      "code": "ERR_SENSOR.HARDWARE.BLIND_ZONE",
      "message": "Ultrasonic echo received before minimum hardware timing threshold.",
      "severity": "WARN",
      "meta": {
        "raw_echo_duration_us": 120,
        "hardware_pin_trig": 12,
        "hardware_pin_echo": 14
      }
    }
  }
}
```

#### Example B: Core-Compliant Protocol Error (Malformed Input)

An example of an edge gateway rejecting a message due to a broken signature or missing envelope element:


```JSON
{
  "gos": "1.0",
  "from": "22:03:00:BC:78:90",
  "to": "22:01:00:AF:12:34",
  "timestamp": "2026-06-20T22:45:02Z",
  "payload": {
    "error": {
      "code": "ERR_PROTOCOL.ENVELOPE.MISSING_FIELD",
      "message": "The incoming transaction was dropped due to a missing root field: 'from'.",
      "severity": "CRITICAL"
    }
  }
}
```

## 4. Backwards Compatibility & Edge Cases

### 4.1 Severity Level Enforcement Boundaries

Receiving layers (Platform and Business) MUST trigger distinct routing workflows based on the reported `severity` value:

- `WARN`: The capability is experiencing transient degradation. Transactions may continue, but anomalies must be logged.
    
- `CRITICAL`: The targeted capability or sub-system has failed entirely. The platform MUST invalidate the current capability lease state.
    
- `FATAL`: The entire physical node is collapsing or entering an unrecoverable panic state. The platform MUST immediately declare the node offline and trigger system alerts.
    

### 4.2 Handling Unknown Custom Error Codes

When an orchestration agent or business dashboard processes an error payload containing a highly specific, nested custom error code (e.g., `ERR_SENSOR.HARDWARE.BLIND_ZONE.VEND_TRIP_A`), it MUST gracefully fallback to evaluating the root-level segments (`ERR_SENSOR.HARDWARE`). This fallback guarantees universal error visibility across all tiers of the network.