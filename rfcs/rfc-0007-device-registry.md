# GOS RFC-0007: Device Records

- **Status:** Draft v0.2 (replaces Draft v0.1, the Device Registry service)
- **Category:** Interface
- **Author:** Ghost System Community
- **Scope:** Technical Specification
- **Applies to:** Provisioning tools, flashing tools and site servers
- **Decision record:** Ghost handbook ADR 0008

The key words MUST, MUST NOT, SHOULD and MAY are to be interpreted as described in RFC 2119.

## 1. Introduction & Context

### 1.1 Purpose

This RFC defines the **records repository** of a manufacturer: the versioned archive that keeps, for every device the manufacturer provisioned, the link between its GUID, Ghost MAC and Hardware ID (RFC-0003), and its history. It specifies the layout of the repository, the content of a device record and the rules tools follow to write and read it.

The machine-readable format of a record is `schemas/device-record.json`.

### 1.2 Motivation

Since RFC-0003 v1.2 a provisioning station creates a unique identity on its own: the manufacturer code, the station code and the station journal rule out duplicates. What a station cannot do alone is keep the records of a manufacturer together, safe and auditable, for recovery and after-sales. A git repository per manufacturer does that with no service to run, works with tools that are offline most of the time, and its history is the audit trail.

The records repository is an **archive, not an allocation authority**. Uniqueness is settled before a record reaches it.

### 1.3 Scope

The records say **what a device is**. They do not say where a device is installed, what it is paired with or how it is configured: that belongs to the site server of each installation.

## 2. Core Specification

### 2.1 Out of the Runtime Path

- Field nodes, gateways and site servers MUST NOT need the records repository to operate. A site MUST keep working without Internet access.
- The repository is used for recovery, after-sales and audit. A site server MAY read it, when online and allowed, to resolve a device it does not know.

### 2.2 One Repository per Manufacturer

- Each manufacturer that holds a manufacturer code (RFC-0003 Section 2.5) keeps **one private git repository** of records. The reference hosting is GitHub, in the manufacturer's organisation.
- The repository MUST NOT depend on any product or vertical. The kind of a device is its GOS device type (`dictionaries/devices.json`); product, hardware revision and batch are plain attributes.
- The default branch is the record. It MUST be protected against force pushes and deletion.

### 2.3 Layout

```
manufacturer.json                         manufacturer code and name
stations/<SS>.json                        one file per station code
devices/22-<MM>/<SS>/<B3>/<ghost-mac>.json one file per device
hwid/<H0>/<H1>/<hardware-id>.json         index: Hardware ID -> Ghost MAC
```

- `<MM>`, `<SS>` and `<B3>` are bytes 1, 2 and 3 of the Ghost MAC, as two upper-case hexadecimal digits. `<ghost-mac>` is the canonical Ghost MAC with `-` instead of `:`, for example `devices/22-07/3A/22/22-07-3A-22-49-A5.json`.
- `<hardware-id>` is the canonical Hardware ID with `-` instead of `:`; `<H0>` and `<H1>` are its first two bytes. An index file holds `{"ghost_mac": "<canonical Ghost MAC>"}`.
- The split into folders keeps every folder small.

### 2.4 Stations

- A manufacturer assigns a station code by creating `stations/<SS>.json` (Section 3.2) **before** the station provisions any device.
- A station code file is created once. A tool MUST refuse a station code whose file exists for another station, and MUST NOT provision with a code whose file is missing or whose state is `retired`.
- A retired code is never assigned again.

### 2.5 Writing Records

- A station keeps its journal (RFC-0003 Section 2.11) and sends every new identity to the repository as **one commit** that creates the device file and, when the device has a Hardware ID, its index file. It sends it at once, or when the network returns, in journal order.
- A new device file MUST be created, never overwritten. If the path already exists:
  - with the same GUID, the record was already sent: the tool treats the push as done;
  - with another GUID, two stations share a station code or a journal was lost. The tool MUST stop provisioning and report `DUPLICATE_GHOST_MAC`.
- An existing Hardware ID index file that names another Ghost MAC means the chip was provisioned twice: the tool MUST report `DUPLICATE_HARDWARE_ID` and recover the first identity instead (RFC-0003 Section 2.11).
- A record is never deleted. A change of state, an event or a link is a new commit that modifies the device file.
- The author of every commit identifies the operator; the record names the station (Section 3.1).

### 2.6 Access and Security

- Operators of a manufacturer have write access to its records repository and to nothing else (ADR 0008). Reading it is enough for recovery and after-sales.
- The repository holds no secret of a device (no key, no network credential) and no data of a customer or a site.
- A Ghost MAC, a Human ID or a GUID on a label is not a secret. Possessing one grants nothing: writing needs the right on the repository.

### 2.7 Operations

| Operation | In the repository |
|---|---|
| `provision` | Create the device file (state `PROVISIONED`) and its Hardware ID index (Section 2.5). |
| `lookup` | Read the device file at the path of its Ghost MAC. From a Hardware ID: read the index first. From a GUID (the QR code of a label): derive the Ghost MAC (RFC-0003 Section 2.4) first. |
| `recover` | Read the record, write its identity to the device again, append a `recovered` event. |
| `reissue` | Create the record of the new identity with `replaces`; set `replaced_by` and the state `RETIRED` on the old record, when it exists. |
| `retire` | Set the state `RETIRED`. Its identifiers are never reused. |
| `record event` | Append a `flashed`, `installed`, `faulty` or `recovered` event to the record. |

### 2.8 Offline Work

- Every operation first works on the station journal. A tool that is offline queues its commits and sends them when the network returns.
- A station that lost its journal MUST rebuild it from the device files of its own station code before it provisions again.
- `lookup` offline only sees the station journal; a QR code read from a label is always enough to restore an identity (RFC-0003 Section 2.11).

## 3. Architecture & Data Structures

### 3.1 Device Record

A device file is a JSON document that follows `schemas/device-record.json`.

| Field | Content |
|---|---|
| `schema_version` | `1` |
| `guid` | UUID of the device |
| `ghost_mac` | Canonical Ghost MAC `22:MM:SS:XX:XX:XX` |
| `human_id` | Human ID, computed from the Ghost MAC |
| `hardware_id` | Hardware ID, absent when the chip has none |
| `device_type` | GOS device type name (`dictionaries/devices.json`) |
| `product`, `hardware_revision`, `batch` | Attributes; `batch` is optional |
| `firmware_version` | Version written at provisioning, then the last one reported by a `flashed` event |
| `state` | `PROVISIONED`, `INSTALLED`, `FAULTY`, `RETIRED` |
| `provisioned_at` | Date and time, RFC 3339 |
| `provisioned_by` | `{"station": "<SS>", "operator": "<login>"}` |
| `replaces`, `replaced_by` | Ghost MACs linked by `reissue` |
| `events` | List of `{"type", "at", "station", "operator", "detail"}` |

Tools MUST check, before they write a record and when they read one, that `ghost_mac` derives from `guid` (RFC-0003 Section 2.4) and that `human_id` matches `ghost_mac`.

### 3.2 Station File

`stations/<SS>.json` holds `{"code": "<SS>", "name": "<station name>", "created_at": "<RFC 3339>", "state": "active" | "retired"}`.

### 3.3 Lifecycle

```
PROVISIONED ──► INSTALLED ◄──► FAULTY
     │              │            │
     └──────────────┴────────────┴──► RETIRED
```

`INSTALLED` and `FAULTY` are reported by tools as events; they inform after-sales and never drive the behaviour of a site.

### 3.4 Errors

Tools report errors with the GOS error contract (RFC-0006), in the `ERR_RECORDS` domain: `DUPLICATE_GHOST_MAC`, `DUPLICATE_HARDWARE_ID`, `NOT_DERIVED`, `UNKNOWN_DEVICE`, `UNKNOWN_STATION`, `NOT_ALLOWED`.

## 4. Backwards Compatibility & Edge Cases

- **No imported addresses.** No production device exists before RFC-0003 v1.2. A prototype or bench unit, whatever Ghost MAC it carries, is provisioned like a blank device.
- **The Device Registry service of v0.1** (`ghost-device-registry`, contract `schemas/device-registry.openapi.yaml`) is frozen. It is not needed to provision. A later service MAY import the records repositories as they are; the layout and the record schema are designed for that.
- **Two stations with the same code** are found at the first push of a duplicate path (Section 2.5); no record is overwritten.

## 5. Changes from v0.1

- The online Device Registry service is replaced by one git repository of records per manufacturer.
- Identities are created by the provisioning station (RFC-0003 v1.2), not allocated: no reservation, no commit, no `RESERVED` state.
- Station codes are files of the repository.
- Recovery can start from a GUID read on a label.
- Operators write to the repository of their manufacturer.

## 6. Open Points

- **O-1** A consolidated view of the devices of several manufacturers, for the standard's owner.
- **O-2** Signed records (RFC-0003 open point O-1).
