# GOS RFC-0003: Identity & Persistence

- **Status:** Draft v1.1 (proposed revision of Stable v1.0)
- **Category:** Foundation
- **Author:** Ghost System Community
- **Scope:** Technical Specification
- **Applies to:** All Ghost-Compatible Systems

## 1. Introduction & Context

### 1.1 Purpose

This RFC defines the identity primitives of the Ghost Open Standard (GOS): the Typed Ghost MAC, the Global Unique Identifier (GUID) and the Hardware ID. The last four bytes of the Ghost MAC are the manufacturing serial number of the device. The RFC mandates where the identity is stored, how it survives firmware updates and factory resets, how a device gets, keeps and recovers it, and what its physical label carries.

### 1.2 Motivation

Interoperability in a distributed IoT mesh requires immutable, collision-free device identification. Version 1.0 fixed the format of the Ghost MAC and the factory reset contract. It left three questions open, which field work on the first Ghost product made concrete:

- **Replacement.** When a device is replaced, the installation (a place, a zone, its configuration) must stay and only the device must change. This needs an identity that belongs to the device and nothing else.
- **Reflash.** A firmware update must never change who the device is, and one firmware image must fit every device of a product.
- **Loss.** A chip without a silicon unique ID (for example ATmega328P) cannot re-derive its identity after its storage is erased, and no chip can re-derive an address that a registry issued. Something outside the device must remember it.

Version 1.1 answers them with an identity record kept apart from the firmware image and a Device Registry (RFC-0007) that issues and remembers identities.

### 1.3 Terminology

| Term | Meaning |
|---|---|
| **Ghost MAC** | 6-byte protocol address of a node (Section 3.1). The only identifier printed on a label. |
| **Instance ID** | Bytes 2 to 5 of the Ghost MAC. It is the manufacturing serial number of the device: unique across all device types. Called "Serial Number" in v1.0. |
| **Serial Number** | The Instance ID as people read it, for example `00:1E:3A:1D`. It is not a separate identifier. |
| **GUID** | 128-bit UUID of the device. Primary key of the device in the Device Registry. |
| **Hardware ID** | Unique identifier burnt in silicon by the chip manufacturer, when one exists (for example the ESP32 base MAC address in eFuse, also called its physical MAC). Read-only. |
| **Identity Record** | The persistent copy of the identity held by the device (Section 3.3). |
| **Device Registry** | The service that issues identities and keeps the link Ghost MAC, GUID, Hardware ID (RFC-0007). |
| **Provisioning** | Giving a device its identity for the first time. |

## 2. Core Specification

### 2.1 The Identity Primitives

| Primitive | Role | Source | Changes over the life of the device |
|---|---|---|---|
| Ghost MAC | Protocol address on every Ghost transport; label identifier | Issued by the Device Registry (Section 2.3) | Never |
| GUID | Key of the device in the Device Registry; arbiter of every conflict | Generated once at provisioning | Never |
| Hardware ID | Physical anchor, where the chip has one | Silicon | Never |

- Every Ghost node MUST have a Ghost MAC.
- Every provisioned device MUST have a GUID.
- A device whose chip has a Hardware ID MUST report it, at provisioning and in its identity payload (Section 3.4). The Hardware ID is never used as a protocol address and never used to compute one.

### 2.2 The Ghost MAC Structural Invariance

Every hardware node participating in a Ghost network MUST possess a 6-byte unique identifier known as the **Ghost MAC**, with the layout of Section 3.1: a mandatory leading token, a hardware type code and a 32-bit Instance ID.

### 2.3 Source of the Instance ID

The Instance ID MUST come from the first source that applies:

1. **Stored.** If the Identity Record holds a Ghost MAC, it MUST be used unchanged.
2. **Issued.** A production device MUST receive its Ghost MAC from the Device Registry at provisioning, whatever its chip. The registry draws the Instance ID from a single counter shared by all device types and all namespaces, so two devices never share an Instance ID even when their type codes differ. The type code is a classification and takes no part in the numbering. The Instance ID is never computed from a Hardware ID.
3. **Self-minted (laboratory).** A device that is not provisioned MAY mint an Instance ID at random so that it can run on a bench, only in the laboratory range `0xFF000000` to `0xFFFFFFFE`. Such a device is `UNREGISTERED` (Section 2.8). The Device Registry never issues a value from that range.

The range `0x00000001` to `0xFEFFFFFF` (more than 4.2 billion values) belongs to the registry. Values are never reused, including after a device is retired; gaps are normal. The Device Registry MUST refuse to record two devices with the same Ghost MAC.

### 2.4 GUID

- The GUID MUST be a UUID (RFC 9562) generated once, at provisioning, and recorded in the Device Registry.
- The GUID MUST NOT be derived from the Ghost MAC or the Hardware ID.
- Two records that disagree are reconciled on the GUID: the GUID is never changed to resolve a conflict on another field.

### 2.5 Serial Number and Label

- The Serial Number is the Instance ID. There is no other manufacturing number: it is unique by construction and needs no collision rule.
- The physical label of a device MUST carry the Ghost MAC, and only the Ghost MAC, as data. It carries it three times, always in the same canonical text form (uppercase hexadecimal, colon-separated, for example `22:01:00:1E:3A:1D`):
  - as human-readable text;
  - as a Code 128 barcode;
  - as a QR code.
- A label MAY show a product name or a logo for the eye. It is decoration: tools MUST NOT read it and MUST NOT depend on it.
- A label MUST NOT carry a secret (key, pairing code, network credential) or any data that changes during the life of the device (state, firmware version, configuration).
- A device with a display SHOULD show its Ghost MAC in the same canonical form and its registration state, so that the display can be compared with the label.
- The label is the recovery anchor of a device without a Hardware ID (Section 2.9).

### 2.6 Persistence

- The identity MUST be stored in non-volatile data storage that is separate from the firmware image (EEPROM, NVS, FRAM, secure element).
- A firmware image MUST NOT contain a device identity. One image MUST fit every device of a product.
- A firmware update MUST NOT erase, regenerate or modify the Identity Record. A flashing tool MUST NOT erase the storage that holds it on a provisioned device, except on an explicit re-provisioning request.

**Factory reset contract.** When a Ghost node undergoes a factory reset:

- **Identity immutability:** the reset MUST NOT erase, regenerate or modify the Ghost MAC, the GUID or the registration state. These fields are bound to the hardware lifecycle.
- **State volatility:** the reset MUST erase cryptographic keys, network configurations, broker credentials, local caches, logical bus addresses and device pairings.

### 2.7 Integrity of the Identity Record

- The Identity Record MUST carry a format version and a checksum, and MUST be stored in two copies.
- A write MUST be ordered so that a power cut leaves at least one valid copy or an unprovisioned record, never a half-written identity that reads as valid.
- At boot, a device MUST use a valid copy and SHOULD repair the other one from it.
- If no copy is valid, the device MUST NOT silently create a new production identity. It behaves as `UNREGISTERED`, with a laboratory Ghost MAC, until it is recovered (Section 2.9).

### 2.8 Registration States

| State | Meaning |
|---|---|
| `UNREGISTERED` | No identity issued by the Device Registry. Bench and laboratory use only; the Ghost MAC is in the laboratory range. |
| `PROVISIONED` | Identity issued by the Device Registry and confirmed after read-back. |

- The registration state MUST be part of the Identity Record and MUST be reported in the identity payload (Section 3.4).
- Commissioning tools and installation servers MUST NOT bind an `UNREGISTERED` device to an installation.

### 2.9 Provisioning, Reflash and Recovery

- **Provisioning.** A device without a valid identity can only be provisioned while the provisioning tool is connected to the Device Registry. Without a connection the tool MUST refuse.
- **Reflash.** Updating the firmware of a `PROVISIONED` device does not require a connection to the Device Registry. The tool reads the identity before and after the update and MUST report any difference as a failure.
- **Recovery with a Hardware ID.** When the Identity Record of such a device is lost, the tool MUST look the device up in the registry by its Hardware ID and restore the same Ghost MAC and GUID. A new identity MUST NOT be issued.
- **Recovery without a Hardware ID.** The label is the recovery anchor: the tool reads the Ghost MAC on the label (barcode, QR code or text), looks the device up in the registry by that Ghost MAC and restores the recorded identity.
- **Reissue.** Only when recovery is impossible for a device without a Hardware ID (unreadable label, no registry record), a new Ghost MAC and GUID are issued. The registry records that the new identity replaces the old one and retires the old one. The label MUST be reprinted.

The registry operations behind these rules are specified in RFC-0007.

## 3. Architecture & Data Structures

### 3.1 Typed Ghost MAC Byte Layout

The Ghost MAC consists of exactly 6 bytes (48 bits), represented in text as six groups of two hexadecimal digits separated by colons (e.g., `22:01:00:AF:12:34`). The byte layout MUST follow Big-Endian (Network Byte Order) convention:

|**Byte Position**|**Field Name**|**Type / Value**|**Description**|
|---|---|---|---|
|**Byte 0**|Protocol Prefix|`uint8_t` / `0x22`|Mandatory token identifying a native Ghost device.|
|**Byte 1**|Type Code|`uint8_t`|Type-code registry token identifying the hardware family.|
|**Byte 2 to 5**|Instance ID|`uint32_t`|Manufacturing serial number (Section 2.3). MUST NOT be `0x00000000` or `0xFFFFFFFF`.|

`0xFFFFFFFF` stands for "no Instance ID" in tools and displays that have no identity to show yet. It is never an address.

### 3.2 Standard Type-Code Registry (v1.0)

Byte 1 dictates the node archetype, as listed in `dictionaries/devices.json`:

- `0x00`: Reserved / Unspecified Node
- `0x01`: `FIELD_SENSOR` (e.g., Parking Radar Sensor)
- `0x02`: `FIELD_ACTUATOR` (e.g., Relay, Valve Controller)
- `0x03`: `FIELD_GATEWAY` (Gateway / Edge Runtime Unit)
- `0x04`: `FIELD_HMI` (Interactive Kiosk / Presentation HMI)
- `0x05` to `0xFE`: Reserved for future GOS expansions
- `0xFF`: Unknown / not set. Never assigned to a device; tools and displays use it when no type is known.

### 3.3 Identity Record (logical content)

The byte layout is defined by each runtime library; the content is normative.

| Field | Size | Content |
|---|---|---|
| Format version | 1 byte | Version of the record layout |
| Registration state | 1 byte | `UNREGISTERED` or `PROVISIONED` |
| Ghost MAC | 6 bytes | Section 3.1 |
| GUID | 16 bytes | Present when `PROVISIONED` |
| Checksum | 2 bytes or more | Over all the fields above |

The Hardware ID is read from silicon and is not stored in the record. The logical bus address of a node is not part of the identity. It is volatile state (Section 2.6) and MUST be stored outside the checksummed record.

### 3.4 Representation in Identity Contract

The network-facing identity payload MUST ONLY present identification, firmware versioning and low-level physical traits. High-level capabilities are strictly forbidden from leaking into this payload.

```json
{
  "guid": "f81d4fae-7dec-11d0-a765-00a0c91e6bf6",
  "ghost_mac": "22:03:00:00:01:7C",
  "hardware_id": "AC:67:B2:11:22:33",
  "registration": "PROVISIONED",
  "device_type": "FIELD_GATEWAY",
  "hardware_revision": "v2.1.0",
  "firmware_version": "1.0.4"
}
```

`guid` is absent while the device is `UNREGISTERED`. `hardware_id` is present when the chip has one: it is the physical MAC address of the chip (for an ESP32, the base MAC address from eFuse), written as colon-separated hexadecimal pairs. It identifies the silicon only and is never a protocol address.

Transports that cannot carry this payload (for example a register-based field bus) MUST expose at least the Ghost MAC, and SHOULD expose the GUID, the registration state and the firmware version in a read-only identity block defined by the transport specification.

## 4. Backwards Compatibility & Edge Cases

### 4.1 Collision and Conflict Handling

- If two nodes present the same Ghost MAC on one network, the Edge Automation Runtime layer MUST reject the second registration and flag a duplicate identity conflict.
- Security layers MUST treat a changing GUID attached to an identical Ghost MAC as identity spoofing and isolate the node. This is why a device with a Hardware ID is always recovered with its original GUID (Section 2.9) and why a reissue always changes the Ghost MAC as well.
- A device with a Hardware ID whose stored identity does not match the registry record for that Hardware ID MUST be reported as a conflict.

### 4.2 No Derivation from a Hardware ID

Version 1.0 required a deterministic 32-bit hash of the Hardware ID as Instance ID (SHA-256 truncated in the text, FNV-1a in the reference implementation). Version 1.1 removes the derivation: every production Ghost MAC is issued by the registry, so the Hardware ID is a lookup key and nothing more. An implementation MUST NOT compute a production Ghost MAC from a Hardware ID.

### 4.3 Devices Built Before v1.1

No production device holds a definitive identity before the Device Registry exists. Prototypes and bench units, including those carrying a v1.0 Ghost MAC, are `UNREGISTERED`; their addresses have no standing and are replaced when the device is provisioned.

## 5. Changes from v1.0

- Added the GUID, the Hardware ID report, the Identity Record, the registration states and the Device Registry.
- The Instance ID (bytes 2 to 5, called "Serial Number" in v1.0) is the manufacturing serial number, issued by the registry from one counter for all device types. The byte layout is unchanged.
- Removed the derivation of the Instance ID from a Hardware ID; added a laboratory range for unregistered devices.
- Defined the label: the Ghost MAC as text, Code 128 and QR code, nothing else.
- Required the identity to live outside the firmware image and to survive firmware updates.
- Added integrity rules for the stored identity.
- Added provisioning, reflash, recovery and reissue rules.
- Reserved type code `0xFF` as "unknown".
- Aligned the type-code names with `dictionaries/devices.json`.

## 6. Open Points

- **O-1** UUID version of the GUID (v4 random or v7 time-ordered).
- **O-2** Whether the registry signs the Identity Record, and which layer verifies the signature.
