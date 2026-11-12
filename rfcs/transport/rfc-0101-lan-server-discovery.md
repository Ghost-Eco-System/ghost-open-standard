# GOS RFC-0101: LAN Server Discovery

- **Status:** Draft
- **Category:** Transport
- **Author:** Ghost System Community
- **Scope:** Protocol
- **Applies to:** IP gateways (Ethernet) and the local servers (desktop apps) they connect to

## 1. Introduction & Context

### 1.1 Purpose

This RFC defines how a gateway on a local IP network finds the WebSocket endpoint of its server without any manual configuration. It is the IP-network counterpart of RFC-0005: RFC-0005 covers how a node announces itself once it reaches its registry, and this RFC covers how the gateway finds that registry in the first place.

### 1.2 Motivation

RFC-0005 states that the ecosystem must eliminate manual device provisioning. Before this RFC, an installer had to type the server URL (`ws://<pc-ip>:8765/ghost`) on the gateway's USB console. With this RFC, a factory-fresh gateway plugged into the LAN by Ethernet reaches the server by itself ("plug & play").

## 2. Core Specification

### 2.1 Roles

- **Server:** the process that accepts gateway WebSocket sessions (for example the Ghost Parking desktop app).
- **Gateway:** the device that opens the WebSocket session.

### 2.2 Server Obligations

- The server MUST advertise a DNS-SD service over mDNS (RFC 6762/6763) with:
  - service type `_ghost-parking._tcp.local`
  - an SRV record pointing to `<hostname>.local` and the WebSocket port (default `8765`)
  - an A record for each LAN IPv4 address
  - TXT records `path=/ghost` and `v=1`
- The server MUST answer the UDP fallback probe (§3.2) on UDP port `8766`.
- The server SHOULD exclude loopback, link-local (169.254/16) and virtual adapters from the addresses it advertises.
- When the server has several LAN interfaces, a UDP reply MUST carry the address of the interface on the prober's subnet.

### 2.3 Gateway Obligations

- **The Manual-Override Rule:** a server URL configured by an operator MUST always win. The gateway MUST run discovery only when no manual URL is configured.
- The gateway MUST try mDNS first: a PTR query for `_ghost-parking._tcp.local` sent to `224.0.0.251:5353`, up to 3 times, 1 s apart.
- If mDNS gives no usable answer, the gateway MUST fall back to the UDP probe: up to 3 broadcasts to `255.255.255.255:8766`, 1 s apart.
- If both fail, the gateway MUST back off (5 s, doubling up to 60 s) and retry.
- The gateway MAY cache the discovered URL across reboots. After 3 consecutive failed connections to a cached or discovered URL, it MUST discover again; this covers a server whose IP address changed.
- After the WebSocket session opens, the RFC-0005 lifecycle applies unchanged.

## 3. Architecture & Data Structures

### 3.1 mDNS Service

```
_ghost-parking._tcp.local.        PTR  ghost-pc._ghost-parking._tcp.local.
ghost-pc._ghost-parking._tcp.local. SRV  0 0 8765 ghost-pc.local.
ghost-pc._ghost-parking._tcp.local. TXT  "path=/ghost" "v=1"
ghost-pc.local.                   A    192.168.1.20
```

The gateway builds the URL as `ws://<A>:<SRV port><TXT path>`. If `path` is missing, it MUST use `/ghost`.

### 3.2 UDP Fallback Probe (port 8766)

The gateway sends to the broadcast address:

```JSON
{"type":"discover","v":1,"gateway":"22:02:00:AF:12:34"}
```

The server answers by unicast to the sender's address and port:

```JSON
{"type":"server","v":1,"ws":"ws://192.168.1.20:8765/ghost","name":"GHOST-PC"}
```

- Both datagrams MUST be UTF-8 JSON, 512 bytes or less.
- A receiver MUST silently ignore datagrams that are not valid JSON, have another `type`, or have a `v` it does not support.

## 4. Backwards Compatibility & Edge Cases

- **Gateways without discovery** keep working with a manual URL; servers implementing this RFC change nothing for them.
- **Multicast blocked** (managed switches, Windows firewall): the UDP broadcast fallback covers it inside the subnet. Cross-subnet installs still need a manual URL.
- **Several servers on one LAN:** the gateway uses the first valid answer. Installations with several servers SHOULD set a manual URL. A future version MAY add a `site` TXT key to select a server.
- **Security:** discovery is unauthenticated, and any host on the LAN can answer. This matches the current trust model (plain `ws://`, no authentication). Pairing and TLS are left to a future RFC.
- **Firewall:** the server host MUST allow inbound TCP 8765, UDP 8766 and UDP 5353 on private networks.
