# GOS RFC-0003: Identity & Persistence

- **Status:** Stable v1.0    
- **Category:** Foundation    
- **Author:** Ghost System Community    
- **Scope:** Technical Specification    
- **Applies to:** All Ghost-Compatible Systems    

## 1. Introduction & Context

### 1.1 Purpose

This RFC defines the unique identity primitives of the Ghost Open Standard (GOS) and mandates their lifecycles and persistence criteria across hardware factory resets. It formalizes the structure of the Typed Ghost MAC address and the Global Unique Identifier (GUID).

### 1.2 Motivation

Interoperability in a distributed IoT mesh requires immutable, collision-free device identification. Previous implementations allowed inconsistent generation of identity formats and ambiguous behavior during factory resets. By underdefining these mechanisms, security pairings could accidentally survive resets while core identities were wiped, causing severe routing collisions. This specification establishes a deterministic binaire layout and an absolute behavioral contract for hardware state resets.

## 2. Core Specification

### 2.1 The Ghost MAC Structural Invariance

Every hardware node participating in a Ghost network MUST possess or generate a 6-byte unique identifier known as the **Ghost MAC**.

- The Ghost MAC MUST strictly follow the 6-byte structural layout specified in Section 3.
    
- The structure enforces a mandatory leading token, an explicit hardware type classification code, and a manufacturer-defined unique serial block.
    

### 2.2 Factory Reset Persistence Contract

When a Ghost node undergoes a factory reset procedure, the device state transitions MUST strictly respect the following boundaries:

- **Identity Immutability:** A factory reset MUST NOT erase, regenerate, or modify the GUID or the Ghost MAC. These fields are bound to the hardware lifecycle.
    
- **State Volatility:** A factory reset MUST completely erase cryptographic keys, network configurations, broker credentials, local caches, and device pairings.
    

## 3. Architecture & Data Structures

### 3.1 Typed Ghost MAC Byte Layout

The Ghost MAC consists of exactly 6 bytes (48 bits), represented in text format as six groups of two hexadecimal digits separated by colons (e.g., `22:01:00:AF:12:34`). The byte layout MUST follow Big-Endian (Network Byte Order) convention:

|**Byte Position**|**Field Name**|**Type / Value**|**Description**|
|---|---|---|---|
|**Byte 0**|Protocol Prefix|`uint8_t` / `0x22`|Mandatory token identifying a native Ghost device.|
|**Byte 1**|Type Code|`uint8_t`|Type-code registry token identifying the hardware family.|
|**Byte 2 to 5**|Serial Number|`uint32_t`|Manufacturer-defined unique hardware serial sequence.|

### 3.2 Standard Type-Code Registry (v1.0)

Byte 1 dictates the node archetype. The initial GOS v1.0 allocation is defined below:

- `0x00`: Reserved / Unspecified Node
    
- `0x01`: Field Sensor Node (e.g., Parking Radar Sensor)
    
- `0x02`: Field Actuator Node (e.g., Relay, Valve Controller)
    
- `0x03`: Gateway / Edge Runtime Unit
    
- `0x04`: Interactive Kiosk / Presentation HMI
    
- `0x05` to `0xFF`: Reserved for future GOS expansions
    

### 3.3 Representation in Identity Contract

The network-facing Identity payload MUST ONLY present identification, firmware versioning, and low-level physical traits. High-level capabilities are strictly forbidden from leaking into this payload.

JSON

```
{
  "guid": "f81d4fae-7dec-11d0-a765-00a0c91e6bf6",
  "ghost_mac": "22:01:00:AF:12:34",
  "device_type": "FIELD_SENSOR",
  "hardware_revision": "v2.1.0",
  "firmware_version": "1.0.4"
}
```

## 4. Backwards Compatibility & Edge Cases

### 4.1 Collision Handling on Custom Serials

If a manufacturer implements a non-unique serial sequence in Bytes 2-5, the Edge Automation Runtime layer MUST reject the node registration and flags a duplicate identity conflict. Security layers MUST treat a changing GUID attached to an identical Ghost MAC as an identity spoofing attack and isolate the node instantly.

### 4.2 Legacy MAC Aliasing

For microcontrollers lacking a native 48-bit hardware identifier or hardcoded serial, the runtime firmware MUST compute a cryptographic hash (SHA-256) of the chip's internal unique silicon ID (e.g., ESP32 Efuse MAC), extract the lower 4 bytes to populate Bytes 2-5, and append the `0x22` prefix and appropriate Type Code. This ensures deterministic generation across updates.