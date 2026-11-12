# GOS RFC-0007: Device Registry

- **Status:** Draft v0.1
- **Category:** Interface
- **Author:** Ghost System Community
- **Scope:** Technical Specification
- **Applies to:** Provisioning tools, flashing tools and installation servers

## 1. Introduction & Context

### 1.1 Purpose

This RFC defines the Device Registry: the online service that issues device identities and keeps, for every Ghost device, the link between its Ghost MAC, GUID and Hardware ID (RFC-0003). It specifies the data kept for a device, the operations offered to tools and the rules these operations obey.

The machine-readable contract is `schemas/device-registry.openapi.yaml`.

### 1.2 Motivation

RFC-0003 requires an identity that survives firmware updates and can be restored after a loss. A device cannot do that alone, and two provisioning stations cannot guarantee unique addresses without a shared authority. A single registry gives atomic allocation, recovery and an audit trail for every product built on GOS, whatever its vertical.

### 1.3 Scope

The registry says **what a device is**. It does not say where a device is installed, what it is paired with or how it is configured: that belongs to the installation server of each site.

## 2. Core Specification

### 2.1 Out of the Runtime Path

- Field nodes, gateways and installation servers MUST NOT need the registry to operate. A site MUST keep working without Internet access.
- The registry is used for provisioning, recovery, after-sales and audit. An installation server MAY query it, when online, to resolve a device it does not know.

### 2.2 Neutrality

- The registry MUST NOT depend on any product or vertical. The kind of a device is its GOS device type (`dictionaries/devices.json`); product, hardware revision and batch are plain attributes.
- Every device belongs to one **namespace** (one per product line or vertical). Access rights are granted per namespace.

### 2.3 Uniqueness

- GUID: unique in the whole registry.
- Ghost MAC: unique in the whole registry. An allocation that would duplicate one MUST be refused.
- Instance ID (bytes 2 to 5 of the Ghost MAC, the manufacturing serial number): unique in the whole registry, whatever the device type or namespace.
- Hardware ID: unique in the whole registry when present.

### 2.4 Issuing the Instance ID

- The registry draws Instance IDs from **one counter** shared by all device types and namespaces, in ascending order, in the range `0x00000001` to `0xFEFFFFFF`. The range `0xFF000000` to `0xFFFFFFFF` is never issued (laboratory range, RFC-0003 Section 2.3).
- A value is never issued twice and never reused, including when a reservation expires, is cancelled or the device is retired. Gaps are normal.
- The registry issues a Ghost MAC for every device, whatever its chip. It never computes one from a Hardware ID and never accepts one proposed by a tool.

### 2.5 Two-Phase Allocation

Provisioning is a reservation followed by a confirmation, so that a failed flash leaves no orphan identity:

1. **Allocate.** The tool sends the namespace, device type, product, hardware revision and, when the chip has one, the Hardware ID. The registry reserves a GUID and a Ghost MAC in a single atomic operation and returns them with an expiry time. A request whose Hardware ID is already recorded is refused: the device already has an identity and is recovered instead.
2. **Write and read back.** The tool writes the Identity Record to the device and reads it back.
3. **Commit.** The tool confirms. The device record becomes `PROVISIONED`.

A reservation that is not committed before it expires is released. A tool MAY cancel a reservation. While a device is `RESERVED`, `lookup` returns its `allocation_id` and `expires_at` to the station that holds the reservation, and to no other station, so that a tool that lost its state can still commit or cancel (Section 4).

### 2.6 Operations

| Operation | Effect |
|---|---|
| `allocate` | Reserve an identity (Section 2.5). |
| `commit` | Confirm a reservation after read-back. |
| `cancel` | Release a reservation. |
| `lookup` | Find a device by GUID, Ghost MAC or Hardware ID. The Ghost MAC is the value printed on the label. |
| `recover` | Return the recorded identity of a device so that it can be written again; recorded as an event. The device is found by its Hardware ID or by the Ghost MAC read on its label. |
| `reissue` | Issue a new identity that replaces a lost one (RFC-0003 Section 2.9); the old device is retired and linked. Refused for a device that has a Hardware ID. |
| `retire` | Mark a device as permanently out of service. Its identifiers are never reused. |
| `record event` | Append a flash or lifecycle event. |

### 2.7 Offline Tools

- `allocate`, `commit`, `recover` and `reissue` require a live connection.
- A tool that reflashes a `PROVISIONED` device while offline MUST queue the flash event and send it when the connection returns. Events carry an idempotency key so that a retry is recorded once.

### 2.8 Security and Audit

- Every provisioning station MUST authenticate with its own credential. Rights to allocate, recover, reissue and retire are granted separately, per namespace.
- Every operation MUST be appended to an audit log that cannot be edited through the API: who, when, which station, which device, what changed.
- The registry holds no secret of a device (no key, no network credential).
- The Ghost MAC on a label is not a secret. Possessing it grants nothing: every operation needs a station credential with the right for the namespace.

### 2.9 Availability

The registry is a single point for provisioning. Its operator MUST back it up and SHOULD publish a periodic read-only export of the device records. The export is a safety copy, never a second source of truth.

## 3. Architecture & Data Structures

### 3.1 Device Record

| Field | Content |
|---|---|
| `guid` | UUID, primary key |
| `ghost_mac` | `22:TT:XX:XX:XX:XX`; its last four bytes are the manufacturing serial number |
| `hardware_id` | Silicon unique ID (the physical MAC for an ESP32), absent when the chip has none |
| `namespace` | Product line or vertical |
| `device_type` | GOS device type |
| `product`, `hardware_revision`, `batch` | Attributes |
| `state` | `RESERVED`, `PROVISIONED`, `INSTALLED`, `FAULTY`, `RETIRED` |
| `firmware_version` | Last version reported by a flash event |
| `provisioned_at`, `provisioned_by` | Date and provisioning station |
| `allocation_id`, `expires_at` | Only while `RESERVED`, and only in answers to the station that holds the reservation |
| `replaces`, `replaced_by` | GUID links created by `reissue` |

### 3.2 Lifecycle

```
RESERVED ──commit──► PROVISIONED ──► INSTALLED ◄──► FAULTY
    │                     │              │            │
  expiry / cancel         └──────────────┴────────────┴──► RETIRED
```

`INSTALLED` and `FAULTY` are reported by tools as lifecycle events; they inform after-sales and never drive the behaviour of a site.

### 3.3 Errors

Errors follow the GOS error contract (RFC-0006), with codes in the `ERR_REGISTRY` domain: `DUPLICATE_HARDWARE_ID`, `RESERVATION_EXPIRED`, `UNKNOWN_DEVICE`, `NOT_ALLOWED`, `REISSUE_REFUSED_HARDWARE_ID`, `INVALID_REQUEST`, `INTERNAL_ERROR`. A missing, unknown or revoked station credential and a missing right are both `NOT_ALLOWED`. A device the station has no right to look up is reported as `UNKNOWN_DEVICE`, so that its existence is not disclosed.

## 4. Backwards Compatibility & Edge Cases

- **No imported addresses.** No production device exists before the registry. A prototype or bench unit, whatever Ghost MAC it carries, is provisioned like a blank device and receives a new identity; the registry never imports an existing address.
- **Lost reservation.** If the tool crashes between write and commit, the device holds an identity whose reservation has not expired yet. The next provisioning attempt MUST read the device first, look the Ghost MAC up (the answer carries `allocation_id` for the reserving station) and commit the existing reservation, or cancel it and start over after an explicit erase. After the expiry the identity is released and never issued again; the device is then provisioned as a blank one. Another station never sees the `allocation_id`.

## 5. Open Points

- **O-1** Hosting and operator of the reference instance.
- **O-2** Reservation lifetime.
- **O-3** Format of the periodic export.
- **O-4** Whether the registry signs identities (RFC-0003 open point O-2).
