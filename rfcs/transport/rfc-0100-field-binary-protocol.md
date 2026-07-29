# GOS RFC-0100: Field Binary Protocol

- **Status:** Stable v1.0
- **Category:** Transport
- **Author:** Ghost System Community
- **Scope:** Protocol Syntax (Binary)
- **Applies to:** Field Edge Networks Only

#### 1. Introduction & Context

##### 1.1 Purpose

This RFC defines the Ghost Binary Protocol (GBP), a highly optimized, fixed-width binary frame specification for resource-constrained microcontrollers at the Field layer of the Ghost Ecosystem. It provides a deterministic mapping contract for identities, commands, and capability states without requiring high-overhead text serialization (JSON).

##### 1.2 Motivation

While JSON (RFC-0002) is the standard for high-level platforms and agentic orchestration, parsing strings on low-power microcontrollers (e.g., small 8-bit AVR or 32-bit ARM/ESP32 architectures) introduces memory fragmentation (heap allocation) and processing latency. To satisfy the _Minimalism_ and _Performance_ priorities of RFC-0001, this specification collapses the Ghost envelope and capability concepts into a compact, fixed 36-byte network frame, maximizing payload availability while optimizing field bus bandwidth.

#### 2. Core Specification

##### 2.1 Fixed-Width Invariance

Every native Ghost Field binary frame transmitted over a local field interface MUST be exactly **36 bytes (288 bits)** in length.

- Partial packets or frames exceeding 36 bytes MUST be treated as corrupt and discarded at the link layer interface.
- Padding bytes (0x00) MUST be appended to the end of the payload if the actual operational data does not fully occupy the allocated payload space.

##### 2.2 Network Byte Order (Endianness)

All multi-byte numeric fields within the frame structure MUST be encoded using **Big-Endian** convention (Network Byte Order). Microcontrollers utilizing Little-Endian architecture internally MUST perform byte-swapping operations before transmitting or immediately after receiving a Ghost binary frame.

##### 2.3 The MAC Deferral Contract (Local Addressing)

To minimize message overhead, standard telemetry and operational frames MUST NOT include the 6-byte Ghost MAC identifier.

- Communication relies entirely on a localized 2-byte logical routing tuple: `ID_BUS` (Segment/Channel) and `ID` (Node Address).
- The binding of a physical Ghost MAC (RFC-0003) to a local logical address MUST occur exactly once during the initialization sequence via a specialized network registration frame.
- Prior to gateway assignment, a node MUST use the default unconfigured logical address `0xF7` (247).

#### 3. Architecture & Frame Structures

##### 3.1 Normative 36-Byte Frame Layout

The 36-byte block is strictly divided into three functional segments: Routing (4 bytes), Protocol Header (3 bytes), and Extended Payload (29 bytes).

|**Byte Index**|**Field Name**|**Data Type**|**Functional Description**|
|---|---|---|---|
|byte|**TX_ID**|uint8_t|Logical address of the emitting field node.|
|byte|**TX_ID_BUS**|uint8_t|Identifier of the local bus segment or radio channel of the transmitter.|
|byte|**RX_ID**|uint8_t|Logical address of the target receiver or local edge gateway.|
|byte|**RX_ID_BUS**|uint8_t|Identifier of the target bus segment or radio channel.|
|byte|**CAP_ID**|uint8_t|Numeric index mapped to a specific capability (e.g., 0x01 $\rightarrow$ cap.distance).|
|byte|**CMD**|uint8_t|Operation code / transaction type (e.g., Report, Trigger, Error).|
|byte|**CAP_TYPE**|uint8_t|Structural type classification indicator matching RFC-0004.|
|byte[7..35]|**PAYLOAD**|uint8_t|Extended raw data container for capability states, values, or diagnostics.|
#### 4. Backwards Compatibility & Edge Cases

##### 4.1 Interface Independent Abstraction

This RFC strictly isolates the frame layout from the physical wire layer. The bytes defined herein MUST remain identical whether streamed sequentially over an asynchronous UART link (RS-485), wrapped inside an SPI transaction, or embedded as a payload inside a wireless radio transmission (nRF24/BLE). Media-dependent framing (such as adding a Start-of-Frame delimiter byte 0x02, an End-of-Frame token, or a CRC16 checksum) is a local runtime responsibility and MUST occur outside this 36-byte boundary.

##### 4.2 Local Address Collision Recovery

If two physical nodes accidentally emit frames using identical `TX_ID` and `TX_ID_BUS` coordinates simultaneously, the upstream edge gateway MUST trigger an address reassignment sequence. The gateway broadcasts an identity query command forcing all nodes on the target segment to declare their immutable 6-byte Ghost MAC addresses, resetting the logical routing map without interrupting higher-level software components.