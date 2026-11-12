# Ghost Open Standard (GOS)

A lightweight, deterministic, hardware-agnostic communication standard for IoT, edge computing, and distributed automation. GOS defines the foundational protocol, data structures, identity primitives, and interaction contracts for the Ghost Ecosystem — from resource-constrained microcontrollers to cloud platforms and AI agents.

## Philosophy

- **Core Minimalism** — The standard specifies only the minimum needed for interoperability; everything else is application-defined.
- **Sovereign Node Principle** — Each physical node is the sole authoritative source of truth for its identity and capabilities.
- **Layered Architecture** — Dependencies flow downward: Business → Platform → Runtime → GOS Core.
- **Priority Ordering** — Correctness > Resilience > Observability > Performance > Features.

## Repository Structure

```
rfcs/          — Specification documents (RFC-style)
  rfc-0001*    — Core principles (the "constitution")
  rfc-0002*    — JSON message envelope
  rfc-0003*    — Identity & persistence (Ghost MAC, GUID, hardware ID, label, identity record)
  rfc-0004*    — Capabilities grammar & interface matrix
  rfc-0005*    — Capability discovery & node lifecycle
  rfc-0006*    — Error contract
  rfc-0007*    — Device registry (draft)
  transport/   — Transport-layer RFCs
    rfc-0100*  — Ghost Binary Protocol (36-byte frame)
    rfc-0101*  — LAN server discovery (mDNS, UDP fallback), draft
    rfc-0102*  — USB identity console (provisioning), draft
schemas/       — Machine-validatable JSON Schema files
dictionaries/  — Registry of types, devices, and capabilities
```

*Foundation RFCs are **Stable v1.0**, except RFC-0003, under revision (Draft v1.1), and RFC-0007 (Draft v0.1).*

## Wire Formats

GOS defines two wire representations of the same logical model:

| Format | Target | Overhead |
|---|---|---|
| **JSON** | Cloud/edge platforms, SaaS, AI agents | Variable |
| **Ghost Binary Protocol (GBP)** | Field microcontrollers (ESP32, AVR, ARM Cortex) | Fixed 36-byte frame |

Both are governed by the same RFCs and validated against the same schemas.

## Identity

Each device is identified by a **6-byte Ghost MAC** (`0x22` prefix + type code + 4-byte instance ID), and a **GUID**. The last four bytes of the Ghost MAC are the manufacturing serial number, issued by the registry; the label carries the Ghost MAC only. They are stored outside the firmware image and survive firmware updates and factory reset; keys and configuration do not ([RFC-0003](rfcs/rfc-0003-identity-persistence.md)). Identities are issued and remembered by the **Device Registry** ([RFC-0007](rfcs/rfc-0007-device-registry.md), contract in `schemas/device-registry.openapi.yaml`).

## Capabilities

Capabilities are named using a dot-separated namespace under `cap.*`:

```
cap.distance    — Integer, CM, READ, STATEFUL
cap.occupancy   — Boolean, READ, STATEFUL
```

Defined by a three-dimensional interface matrix ([RFC-0004](rfcs/rfc-0004-capabilities-grammar.md)):
- **Access Mode:** READ / WRITE / READ_WRITE
- **Interaction Mode:** EVENT_ONLY / COMMAND_ONLY
- **State Model:** STATEFUL / STATELESS

## Getting Started

1. **Read the RFCs** — Start with [RFC-0001](rfcs/rfc-0001-core-principles.md) for architectural doctrine, then proceed through RFC-0002 through RFC-0006.
2. **Validate messages** — Use the JSON Schemas in `schemas/` to validate envelope, capability, and error payloads against the standard.
3. **Consult the registries** — `dictionaries/` contains the canonical mappings for data types, device classifications, and capabilities.
4. **Implement GBP** — See [RFC-0100](rfcs/transport/rfc-0100-field-binary-protocol.md) for the 36-byte field binary protocol specification.

## License

Ghost Open Standard is:

- specification-first
- tool-agnostic
- implementation-neutral
- auditable
- portable

Implementations may be open or proprietary.  
The contracts remain open.
