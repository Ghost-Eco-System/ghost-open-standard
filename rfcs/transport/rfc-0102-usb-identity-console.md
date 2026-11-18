# GOS RFC-0102: USB Identity Console

- **Status:** Draft v0.2 (aligned with RFC-0003 v1.2)
- **Category:** Transport
- **Author:** Ghost System Community
- **Scope:** Technical Specification
- **Applies to:** Field nodes and gateways with a USB serial console; provisioning tools

## 1. Introduction & Context

### 1.1 Purpose

This RFC defines the text commands a provisioning tool uses over a device's USB serial console to read the Identity Record of the device and to write the identity the tool created (RFC-0003 v1.2 Sections 2.4 and 2.11).

### 1.2 Motivation

The identity is never compiled into the firmware (RFC-0003 Section 2.9). A tool therefore writes it after flashing, through a channel every Ghost device already has on the bench: its USB console. The field bus only reads identities; it never writes them.

### 1.3 Terminology

The key words MUST, MUST NOT, SHOULD and MAY are to be interpreted as described in RFC 2119.

- **Tool:** the provisioning tool, connected to the device.
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

- `<ghost_mac>` is the canonical text form `22:MM:XX:XX:XX:XX` (hexadecimal, either case on input).
- `<guid>` is the canonical 36-character form of RFC 9562 (either case on input).
- Tokens are separated by one or more spaces.
- The device type is not sent: a device knows its own type and stores it in the record (RFC-0003 Section 3.3).

### 2.3 Identity reply

```
[ID] state=<STATE> type=<TT> mac=<ghost_mac> guid=<guid> hwid=<hardware_id>
```

| Field | Content |
|---|---|
| `state` | `PROVISIONED` or `UNREGISTERED` (RFC-0003 Section 2.10) |
| `type` | Device type code of the device, two upper-case hexadecimal digits (RFC-0003 Section 3.2). It is never read from the Ghost MAC. |
| `mac` | Ghost MAC, upper case |
| `guid` | GUID, lower case, or `-` when `UNREGISTERED` |
| `hwid` | Hardware ID (the physical MAC of an ESP32, upper case with colons), or `-` when the chip has none |

### 2.4 Write

A device receiving `id write` MUST:

1. Refuse a Ghost MAC whose prefix is not `0x22` or whose manufacturer code is `00` or `FF` (`bad_mac`), whose Instance ID is in the laboratory range or is `00000000` (`lab_mac`), and an all-zero GUID (`bad_guid`). A device that can compute SHA-256 SHOULD also refuse a Ghost MAC that does not derive from the GUID (RFC-0003 Section 2.4) (`not_derived`).
2. Refuse to replace a different `PROVISIONED` identity unless `force` is given (`provisioned`). Writing the identity it already holds is accepted.
3. Write the record, with its own device type, in two copies as required by RFC-0003 Section 2.10, read both back and compare them with what was asked (`storage` on any difference).
4. On success write `[ID] written`, then the identity reply line built from the record read back.
5. On failure write `[ID] error <reason>`, where `<reason>` is one of `syntax`, `bad_mac`, `lab_mac`, `bad_guid`, `not_derived`, `provisioned`, `storage`, and leave the stored identity unchanged.

A node whose Ghost MAC changes MUST forget its logical bus address, so that its gateway onboards it again under the new identity. A gateway MAY restart.

### 2.5 Tool behaviour

- Before provisioning, the tool reads the identity. It MUST NOT provision a `PROVISIONED` device: reflash and recovery are separate flows (RFC-0003 Section 2.11).
- The tool MUST check, before it sends `id write`, that the Ghost MAC derives from the GUID and carries its own manufacturer and station codes.
- The tool records the identity in its station journal, and prints the label, only after an identity reply whose `state`, `mac` and `guid` equal what it wrote. Otherwise it discards the identity; the Instance ID is not reused.
- A tool waits at least 3 seconds for a reply line before it reports a timeout.

## 3. Example

```
> id
< [ID] state=UNREGISTERED type=01 mac=22:07:FF:3A:91:0C guid=- hwid=-
> id write 22:07:3A:22:49:A5 e99d7d12-bfc8-4abd-af48-bb7d5baf06a4
< [ID] written
< [ID] state=PROVISIONED type=01 mac=22:07:3A:22:49:A5 guid=e99d7d12-bfc8-4abd-af48-bb7d5baf06a4 hwid=-
```

## 4. Backwards Compatibility & Edge Cases

- Firmware without this console answers nothing that starts with `[ID] `: the tool reports a timeout and does not provision.
- A lost reply after a successful write: the tool's `id` read shows the new identity, and the write can be retried with the same values.
- Version 0.1 refused a Ghost MAC whose byte 1 differed from the device type (`type_mismatch`). Since RFC-0003 v1.2 byte 1 is the manufacturer code: that check and its reason are removed.

## 5. Reference Implementation

`ghost-field-core` `ghost_identity_record.h` (record, parsing, reply), used by `ghost-field-arduino` (`GhostModbusFieldNode::handleIdentityCommand`) and by the parking gateway firmware. The tool is `ghost-provisioning-tool` (Ghost-Eco-System).

## 6. Open Points

- **O-1** Authentication of the console (a provisioning PIN or a signed write) for devices left on site.
