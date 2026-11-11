# GOS RFC-0003: Identity & Persistence

- **Status:** Draft v1.1 (proposed revision of Stable v1.0)
- **Category:** Foundation
- **Author:** Ghost System Community
- **Scope:** Technical Specification
- **Applies to:** All Ghost-Compatible Systems

## 1. Introduction & Context

### 1.1 Purpose

This RFC defines the identity primitives of the Ghost Open Standard (GOS): the Typed Ghost MAC, the Global Unique Identifier (GUID), the Serial Number and the Hardware ID. It mandates where they are stored, how they survive firmware updates and factory resets, and how a device gets, keeps and recovers its identity.

### 1.2 Motivation

Interoperability in a distributed IoT mesh requires immutable, collision-free device identification. Version 1.0 fixed the format of the Ghost MAC and the factory reset contract. It left three questions open, which field work on the first Ghost product made concrete:

- **Replacement.** When a device is replaced, the installation (a place, a zone, its configuration) must stay and only the device must change. This needs an identity that belongs to the device and nothing else.
- **Reflash.** A firmware update must never change who the device is, and one firmware image must fit every device of a product.
- **Loss.** A chip without a silicon unique ID (for example ATmega328P) cannot re-derive its identity after its storage is erased. Something outside the device must remember it.

Version 1.1 answers them with an identity record kept apart from the firmware image and a Device Registry (RFC-0007) that issues and remembers identities.

### 1.3 Terminology

| Term | Meaning |
|---|---|
| **Ghost MAC** | 6-byte protocol address of a node (Section 3.1). |
| **Instance ID** | Bytes 2 to 5 of the Ghost MAC. Called "Serial Number" in v1.0; renamed to avoid confusion with the printed serial number. |
| **GUID** | 128-bit UUID of the device. Primary key of the device in the Device Registry. |
| **Serial Number** | Human-readable number printed on the device label. |
| **Hardware ID** | Unique identifier burnt in silicon by the chip manufacturer, when one exists (for example the ESP32 base MAC address in eFuse). Read-only. |
| **Identity Record** | The persistent copy of the identity held by the device (Section 3.3). |
| **Device Registry** | The service that issues identities and keeps the link Ghost MAC, GUID, Hardware ID, Serial Number (RFC-0007). |
| **Provisioning** | Giving a device its identity for the first time. |

## 2. Core Specification

### 2.1 The Four Identity Primitives

| Primitive | Role | Source | Changes over the life of the device |
|---|---|---|---|
| Ghost MAC | Protocol address on every Ghost transport | Section 2.3 | Never |
| GUID | Key of the device in the Device Registry; arbiter of every conflict | Generated once at provisioning | Never |
| Serial Number | Label for people | Assigned at provisioning | May be requalified by the registry (Section 2.5) |
| Hardware ID | Physical anchor, where the chip has one | Silicon | Never |

- Every Ghost node MUST have a Ghost MAC.
- Every provisioned device MUST have a GUID and a Serial Number.
- A device whose chip has a Hardware ID MUST report it at provisioning. The Hardware ID is never used as a protocol address.

### 2.2 The Ghost MAC Structural Invariance

Every hardware node participating in a Ghost network MUST possess a 6-byte unique identifier known as the **Ghost MAC**, with the layout of Section 3.1: a mandatory leading token, a hardware type code and a 32-bit Instance ID.

### 2.3 Source of the Instance ID

The Instance ID MUST come from the first source that applies:

1. **Stored.** If the Identity Record holds a Ghost MAC, it MUST be used unchanged, whatever the other sources would give.
2. **Derived.** If the chip has a Hardware ID, the Instance ID MUST be derived from it with the deterministic function of Section 4.2, so that it can be computed again after a loss.
3. **Assigned.** If the chip has no Hardware ID, a production device MUST receive its Instance ID from the Device Registry at provisioning.
4. **Self-minted.** A device that is not provisioned MAY mint an Instance ID from hardware entropy so that it can run on a bench. Such a device is UNREGISTERED (Section 2.8).

The Device Registry MUST refuse to record two devices with the same Ghost MAC. When a derived Instance ID collides, the registry assigns another one, which is stored and used under rule 1.

### 2.4 GUID

- The GUID MUST be a UUID (RFC 9562) generated once, at provisioning, and recorded in the Device Registry.
- The GUID MUST NOT be derived from the Ghost MAC, the Serial Number or the Hardware ID.
- Two records that disagree are reconciled on the GUID: the GUID is never changed to resolve a conflict on another field.

### 2.5 Serial Number

- The Serial Number is a label, not a key. Its format is defined by the manufacturer; it MUST be printable ASCII and at most 16 characters.
- A Serial Number collision MUST be resolved in the Device Registry, on the GUID, without changing the GUID of either device.
- The label of a device SHOULD carry the GUID in machine-readable form next to the printed Serial Number.

### 2.6 Persistence

- The identity MUST be stored in non-volatile data storage that is separate from the firmware image (EEPROM, NVS, FRAM, secure element).
- A firmware image MUST NOT contain a device identity. One image MUST fit every device of a product.
- A firmware update MUST NOT erase, regenerate or modify the Identity Record. A flashing tool MUST NOT erase the storage that holds it on a provisioned device, except on an explicit re-provisioning request.

**Factory reset contract.** When a Ghost node undergoes a factory reset:

- **Identity immutability:** the reset MUST NOT erase, regenerate or modify the Ghost MAC, the GUID, the Serial Number or the registration state. These fields are bound to the hardware lifecycle.
- **State volatility:** the reset MUST erase cryptographic keys, network configurations, broker credentials, local caches, logical bus addresses and device pairings.

### 2.7 Integrity of the Identity Record

- The Identity Record MUST carry a format version and a checksum, and MUST be stored in two copies.
- A write MUST be ordered so that a power cut leaves at least one valid copy or an unprovisioned record, never a half-written identity that reads as valid.
- At boot, a device MUST use a valid copy and SHOULD repair the other one from it.
- If no copy is valid, the device MUST NOT silently create a new production identity. It behaves as UNREGISTERED until it is recovered (Section 2.9).

### 2.8 Registration States

| State | Meaning |
|---|---|
| `UNREGISTERED` | No identity issued by the Device Registry. Bench and laboratory use only. |
| `PROVISIONED` | Identity issued by the Device Registry and confirmed after read-back. |

- The registration state MUST be part of the Identity Record and MUST be reported in the identity payload (Section 3.4).
- Commissioning tools and installation servers MUST NOT bind an `UNREGISTERED` device to an installation.

### 2.9 Provisioning, Reflash and Recovery

- **Provisioning.** A device without a valid identity can only be provisioned while the provisioning tool is connected to the Device Registry. Without a connection the tool MUST refuse.
- **Reflash.** Updating the firmware of a `PROVISIONED` device does not require a connection to the Device Registry. The tool reads the identity before and after the update and MUST report any difference as a failure.
- **Recovery with a Hardware ID.** When the Identity Record of such a device is lost, the tool MUST look the device up in the registry by its Hardware ID and restore the same Ghost MAC, GUID and Serial Number. A new identity MUST NOT be issued.
- **Recovery without a Hardware ID.** The label is the recovery anchor: the tool looks the device up by the GUID or Serial Number on the label and restores the recorded identity.
- **Reissue.** Only when recovery is impossible for a device without a Hardware ID (unreadable label, no registry record), a new Ghost MAC, GUID and Serial Number are issued. The registry records that the new identity replaces the old one and retires the old one.

The registry operations behind these rules are specified in RFC-0007.

## 3. Architecture & Data Structures

### 3.1 Typed Ghost MAC Byte Layout

The Ghost MAC consists of exactly 6 bytes (48 bits), represented in text as six groups of two hexadecimal digits separated by colons (e.g., `22:01:00:AF:12:34`). The byte layout MUST follow Big-Endian (Network Byte Order) convention:

|**Byte Position**|**Field Name**|**Type / Value**|**Description**|
|---|---|---|---|
|**Byte 0**|Protocol Prefix|`uint8_t` / `0x22`|Mandatory token identifying a native Ghost device.|
|**Byte 1**|Type Code|`uint8_t`|Type-code registry token identifying the hardware family.|
|**Byte 2 to 5**|Instance ID|`uint32_t`|Unique instance number (Section 2.3). MUST NOT be `0x00000000` or `0xFFFFFFFF`.|

### 3.2 Standard Type-Code Registry (v1.0)

Byte 1 dictates the node archetype, as listed in `dictionaries/devices.json`:

- `0x00`: Reserved / Unspecified Node
- `0x01`: `FIELD_SENSOR` (e.g., Parking Radar Sensor)
- `0x02`: `FIELD_ACTUATOR` (e.g., Relay, Valve Controller)
- `0x03`: `FIELD_GATEWAY` (Gateway / Edge Runtime Unit)
- `0x04`: `FIELD_HMI` (Interactive Kiosk / Presentation HMI)
- `0x05` to `0xFF`: Reserved for future GOS expansions

### 3.3 Identity Record (logical content)

The byte layout is defined by each runtime library; the content is normative.

| Field | Size | Content |
|---|---|---|
| Format version | 1 byte | Version of the record layout |
| Registration state | 1 byte | `UNREGISTERED` or `PROVISIONED` |
| Ghost MAC | 6 bytes | Section 3.1 |
| GUID | 16 bytes | Present when `PROVISIONED` |
| Serial Number | up to 16 bytes | Present when `PROVISIONED` |
| Checksum | 2 bytes or more | Over all the fields above |

The logical bus address of a node is not part of the identity. It is volatile state (Section 2.6) and MUST be stored outside the checksummed record.

### 3.4 Representation in Identity Contract

The network-facing identity payload MUST ONLY present identification, firmware versioning and low-level physical traits. High-level capabilities are strictly forbidden from leaking into this payload.

```json
{
  "guid": "f81d4fae-7dec-11d0-a765-00a0c91e6bf6",
  "ghost_mac": "22:01:00:AF:12:34",
  "serial_number": "GPR-26-000123",
  "registration": "PROVISIONED",
  "device_type": "FIELD_SENSOR",
  "hardware_revision": "v2.1.0",
  "firmware_version": "1.0.4"
}
```

`guid` and `serial_number` are absent while the device is `UNREGISTERED`. The Hardware ID is exchanged with the Device Registry only and is not part of this payload.

Transports that cannot carry this payload (for example a register-based field bus) MUST expose at least the Ghost MAC, and SHOULD expose the GUID, the Serial Number, the registration state and the firmware version in a read-only identity block defined by the transport specification.

## 4. Backwards Compatibility & Edge Cases

### 4.1 Collision and Conflict Handling

- If two nodes present the same Ghost MAC on one network, the Edge Automation Runtime layer MUST reject the second registration and flag a duplicate identity conflict.
- Security layers MUST treat a changing GUID attached to an identical Ghost MAC as identity spoofing and isolate the node. This is why a device with a Hardware ID is always recovered with its original GUID (Section 2.9) and why a reissue always changes the Ghost MAC as well.
- A device with a Hardware ID whose stored identity does not match the registry record for that Hardware ID MUST be reported as a conflict.

### 4.2 Deriving the Instance ID from a Hardware ID

For microcontrollers with a silicon unique ID (ESP32 eFuse base MAC, STM32 96-bit UID, RP2040 flash unique ID), the runtime firmware MUST compute a deterministic 32-bit hash of that ID and use it as the Instance ID, with the `0x22` prefix and the appropriate Type Code. Values `0x00000000` and `0xFFFFFFFF` are remapped.

Version 1.0 named SHA-256 truncated to 4 bytes. The reference implementation (`ghost-field-arduino`) has always used FNV-1a 32-bit. See open point O-1.

### 4.3 Devices Deployed Before v1.1

A device holding only a v1.0 identity (a Ghost MAC, no GUID) is `UNREGISTERED`. Provisioning it keeps its Ghost MAC, provided the registry has no other device with that address, and adds the GUID, the Serial Number and the registration state. Its logical address and configuration are not touched.

## 5. Changes from v1.0

- Added the Serial Number, the Hardware ID, the Identity Record, the registration states and the Device Registry.
- Renamed bytes 2 to 5 of the Ghost MAC from "Serial Number" to "Instance ID". The byte layout is unchanged.
- Defined the source of the Instance ID, including assignment by the registry for chips without a Hardware ID.
- Required the identity to live outside the firmware image and to survive firmware updates.
- Added integrity rules for the stored identity.
- Added provisioning, reflash, recovery and reissue rules.
- Aligned the type-code names with `dictionaries/devices.json`.

## 6. Open Points

- **O-1** Hash used to derive the Instance ID: keep SHA-256 as written in v1.0 and change the implementation, or standardize FNV-1a 32-bit as implemented. Changing it changes the Ghost MAC of every device already derived.
- **O-2** Serial Number format (product prefix, year, counter) and the exact content of the label.
- **O-3** UUID version of the GUID (v4 random or v7 time-ordered).
- **O-4** Whether the registry signs the Identity Record, and which layer verifies the signature.
