# GOS RFC-0004: Capabilities Grammar

- **Status:** Stable v1.0
- **Category:** Foundation    
- **Author:** Ghost System Community    
- **Scope:** Protocol Syntax    
- **Applies to:** All Ghost-Compatible Systems    

## 1. Introduction & Context

### 1.1 Purpose

This RFC defines the formal grammar, structural path conventions, and taxonomy of taxonomies for functional primitives within the Ghost Open Standard (GOS), known as **Capabilities**.

### 1.2 Motivation

Distributed IoT ecosystems require a unified mechanism to declare and discover the exact functional footprint of a node. Previous definitions restricted capability identifiers to a rigid single-suffix layout (`cap.domain.type`), which failed to scale for nested technical domains (e.g., separating active power from reactive power). Furthermore, older versions mixed data access, interaction paradigms, and state properties into a single chaotic "interface mode" list. This RFC resolves these issues by introducing an expandable namespaced path grammar and a clean, three-dimensional taxonomy matrix.

## 2. Core Specification

### 2.1 The Namespaced Path Grammar

Every Capability within a Ghost-compliant system MUST be declared using a lower-case alphanumeric dot-separated string prefix.

- The declaration MUST strictly match the following formal Backus-Naur Form (BNF) grammar:

    ```
    <capability_id> ::= "cap." <segment> [ "." <segment> ]*
    <segment>       ::= [a-z0-9_]+
    ```

- Universal top-level domains (e.g., `cap.temperature`, `cap.energy`) are managed by the GOS registry, while custom sub-segments allow granular engineering extensions without breaking root-level structural integrity.    

### 2.2 The Three-Dimensional Interface Matrix

To eliminate impossible semantic configurations, the behavioral profile of a Capability MUST NOT be mixed into a single property. It MUST be explicitly declared across three independent, orthogonal dimensions:

#### 1. Access Mode (`access_mode`)

Dictates data authorization boundaries:

- `READ`: The consumer can only fetch or subscribe to the value.    
- `WRITE`: The consumer can only push state modifications to the node.    
- `READ_WRITE`: Full bidirectional access allowed.    

#### 2. Interaction Paradigm (`interaction_mode`)

Dictates network messaging behaviors:

- `EVENT_ONLY`: The node asynchronously broadcasts state updates when a physical delta occurs; it does not process synchronous read polls.    
- `COMMAND_ONLY`: The node acts solely as a transaction processor, handling execution requests and returning an immediate execution receipt.    

#### 3. State Model (`state_model`)

Dictates local persistence semantics:

- `STATEFUL`: The capability retains its last known operating value in memory, representing a persistent physical condition.    
- `STATELESS`: The capability represents an ephemeral trigger, pulse, or execution hook with no persistent history.    

## 3. Architecture & Data Structures

### 3.1 Grammatical Hierarchy Mapping

Capabilities scale natively through nesting. Below is the structural hierarchy mapping for common field applications:

```
cap (Root Prefix)
 └── energy (Domain Segment)
      └── power (Sub-Domain Segment)
           ├── active   --> cap.energy.power.active
           └── reactive --> cap.energy.power.reactive
```

### 3.2 JSON Schema Profile Declaration

When a node self-announces its functional profile during discovery, it MUST format its capability descriptors using the following JSON Schema structure:

```JSON
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "GOSCapabilityDefinition",
  "type": "object",
  "patternProperties": {
    "^cap\\.[a-z0-9_]+(\\.[a-z0-9_]+)*$": {
      "type": "object",
      "required": ["access_mode", "interaction_mode", "state_model", "type"],
      "properties": {
        "type": {
          "type": "string",
          "enum": ["boolean", "integer", "float", "string", "object"]
        },
        "access_mode": {
          "type": "string",
          "enum": ["READ", "WRITE", "READ_WRITE"]
        },
        "interaction_mode": {
          "type": "string",
          "enum": ["EVENT_ONLY", "COMMAND_ONLY"]
        },
        "state_model": {
          "type": "string",
          "enum": ["STATEFUL", "STATELESS"]
        },
        "unit": {
          "type": "string",
          "description": "Standardized SI unit string (e.g., 'Celsius', 'Hz', 'm')."
        }
      },
      "additionalProperties": false
    }
  },
  "additionalProperties": false
}
```

### 3.3 Concrete Wire-Format Declaration Example

An operational payload declaring a Parking Radar sensor capability profile:

```JSON
{
  "cap.distance": {
    "type": "float",
    "access_mode": "READ",
    "interaction_mode": "EVENT_ONLY",
    "state_model": "STATEFUL",
    "unit": "m"
  }
}
```

## 4. Backwards Compatibility & Edge Cases

### 4.1 Strict Filtering of Illegal Permutations

The platform orchestration engine MUST reject capability registrations containing logically flawed profiles. For example, registering a capability with:

```
access_mode      = WRITE
interaction_mode = EVENT_ONLY
```

is structurally illegal because a write-only channel cannot generate asynchronous outbound state updates. Such profiles MUST be intercepted and dropped at the registration boundary.

### 4.2 Namespace Masking

If a vendor creates an application-specific extension, they MUST prefix their custom segment using the `vendor_` indicator (e.g., `cap.parking.vendor_kiosk_led_pulse`). This avoids path collisions with future official GOS core allocations while ensuring full parsing compatibility for standard agentic consumers.