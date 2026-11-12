# GOS RFC-0102: USB Identity Console

- **Status:** Draft v0.1
- **Category:** Transport
- **Author:** Ghost System Community
- **Scope:** Technical Specification
- **Applies to:** Field nodes and gateways with a USB serial console; provisioning tools

## 1. Introduction & Context

### 1.1 Purpose

This RFC defines the text commands a provisioning tool uses over a device's USB serial console to read the Identity Record of the device and to write the identity issued by the Device Registry (RFC-0003 v1.1 Section 2.9, RFC-0007).

### 1.2 Motivation

The identity is never compiled into the firmware (RFC-0003 Section 2.6). A tool therefore writes it after flashing, through a channel every Ghost device already has on the bench: its USB console. The field bus only reads identities; it never writes them.

### 1.3 Terminology

The key words MUST, MUST NOT, SHOULD and MAY are to be interpreted as described in RFC 2119.

- **Tool:** the provisioning tool, connected to the Device Registry and to the device.
- **Reply line:** a line written by the device that starts with `[ID] `.

## 2. Core Specification

### 2.1 Framing

- Lines are ASCII, ended by `\n`. A device MUST ignore `\r`. A device MUST accept command lines of at least 96 characters.
- A device MAY write other lines (logs) at any time. The tool MUST ignore every line that does not start with `[ID] `.
- The serial settings are those of the device's console (115200 8N1 for the reference devices).

### 2.2 Commands

| Command | Effect |
|---|---|
| `id` | The device writes its identity reply line. |
| `id write <ghost_mac> <guid>` | The device writes the identity as `PROVISIONED` (Section 2.4), then the result. |
| `id write <ghost_mac> <guid> force` | Same, and replaces a different `PROVISIONED` identity. |

- `<ghost_mac>` is the canonical text form `22:TT:XX:XX:XX:XX` (hexadecimal, either case on input).
- `<guid>` is the canonical 36-character form of RFC 4122 (either case on input).
- Tokens are separated by one or more spaces.

### 2.3 Identity reply

```
[ID] state=<STATE> type=<TT> mac=<ghost_mac> guid=<guid> hwid=<hardware_id>
```

| Field | Content |
|---|---|
| `state` | `PROVISIONED` or `UNREGISTERED` (RFC-0003 Section 2.8) |
| `type` | Device type code, two upper-case hexadecimal digits (RFC-0003 Section 3.2) |
| `mac` | Ghost MAC, upper case |
| `guid` | GUID, lower case, or `-` when `UNREGISTERED` |
| `hwid` | Hardware ID (the physical MAC of an ESP32, upper case with colons), or `-` when the chip has none |

### 2.4 Write

A device receiving `id write` MUST:

1. Refuse a Ghost MAC whose prefix is not `0x22`, whose Instance ID is in the laboratory range or is `00000000` (`lab_mac`), or whose device type is not its own (`type_mismatch`), and an all-zero GUID (`bad_guid`).
2. Refuse to replace a different `PROVISIONED` identity unless `force` is given (`provisioned`). Writing the identity it already holds is accepted.
3. Write the record in two copies as required by RFC-0003 Section 2.7, read both back and compare them with what was asked (`storage` on any difference).
4. On success write `[ID] written`, then the identity reply line built from the record read back.
5. On failure write `[ID] error <reason>`, where `<reason>` is one of `syntax`, `bad_mac`, `lab_mac`, `type_mismatch`, `bad_guid`, `provisioned`, `storage`, and leave the stored identity unchanged.

A node whose Ghost MAC changes MUST forget its logical bus address, so that its gateway onboards it again under the new identity. A gateway MAY restart.

### 2.5 Tool behaviour

- Before `allocate`, the tool reads the identity. It MUST NOT provision a `PROVISIONED` device (reflash and recovery are separate flows, RFC-0003 Section 2.9).
- The tool sends the `device_type` and, when present, the Hardware ID read from the device in the allocation request.
- The tool commits the allocation only after an identity reply whose `state`, `mac` and `guid` equal the allocation. Otherwise it cancels the allocation.
- A tool waits at least 3 seconds for a reply line before it reports a timeout.

## 3. Example

```
> id
< [ID] state=UNREGISTERED type=01 mac=22:01:FF:3A:91:0C guid=- hwid=-
> id write 22:01:00:00:00:2A 0b8e5f0e-3f3c-4d3e-9a51-2f4b6c7d8e9f
< [ID] written
< [ID] state=PROVISIONED type=01 mac=22:01:00:00:00:2A guid=0b8e5f0e-3f3c-4d3e-9a51-2f4b6c7d8e9f hwid=-
```

## 4. Backwards Compatibility & Edge Cases

- Firmware without this console answers nothing that starts with `[ID] `: the tool reports a timeout and does not allocate.
- A lost reply after a successful write: the tool's `id` read shows the new identity, and the write can be retried with the same values.

## 5. Reference Implementation

`ghost-field-core` `ghost_identity_record.h` (record, parsing, reply), used by `ghost-field-arduino` (`GhostModbusFieldNode::handleIdentityCommand`) and by the parking gateway firmware. The tool is `ghost-provisioning-tool` (Ghost-Eco-System).

## 6. Open Points

- **O-1** Authentication of the console (a provisioning PIN or a signed write) for devices left on site.
