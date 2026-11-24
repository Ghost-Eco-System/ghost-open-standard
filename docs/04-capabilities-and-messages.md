# Capabilities and messages

## Capabilities

A **capability** is one thing a device can sense or do: measure a distance, detect presence, switch a light, show a number. A device describes its capabilities itself, and every system reads the same description.

Each capability says:

- whether it can be **read**, **written**, or both;
- whether it **reports by itself** when something changes, or **acts on command**;
- whether it has a **state** that can be asked at any time, or only produces events;
- the kind of value it carries and its unit.

New kinds of devices bring new capabilities without changing the standard: the description is enough for systems to use them.

## Messages

Systems exchange **messages**. Every message says who sends it, to whom, when, and which version of the standard it follows. Its content is either a reading, an event, a command, a status or an error.

Two forms of the same messages exist: a compact form for small field devices with little memory, and a readable form for servers, applications and tools. Both carry the same meaning.

## Discovery and lifecycle

- A device **announces** itself when it starts or reconnects: its identity and its capabilities.
- A system may **ask** devices to announce themselves again.
- A device sends a regular **sign of life**. A device that stops sending it is marked as no longer present.
- A gateway finds its site server on the local network by itself; an installer only types an address when the network requires it.

## Errors

When something goes wrong, a device or a system reports an **error** with a code from a shared family of codes, a message for people, and a level of severity. A system that does not know a detailed code still understands its family.
