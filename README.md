# Ghost Open Standard

The Ghost Open Standard describes how devices, gateways and site servers of the Ghost ecosystem work together: what they are, what they promise each other, and the principles behind them.

This repository is **descriptive**. It explains the ideas of the standard in plain words. It does not contain formats, byte layouts, protocol commands, addresses, codes, schemas, dictionaries or examples: those are specified in the Ghost Extended Standard, which is available to licensed manufacturers and partners.

## Contents

| Document | What it describes |
|---|---|
| [Principles](docs/01-principles.md) | Why the standard exists and the rules that guide every decision |
| [Architecture](docs/02-architecture.md) | The layers of a Ghost system and how a site works with or without the Internet |
| [Identity](docs/03-identity.md) | How every device gets a permanent identity, and how people and systems recognise it |
| [Capabilities and messages](docs/04-capabilities-and-messages.md) | How a device says what it can do, and how systems talk to each other |
| [Trust and lifecycle](docs/05-trust-and-lifecycle.md) | How a device is made, installed, kept up to date and retired, and why a site can trust it |
| [Glossary](docs/glossary.md) | The words of the standard |
| [Governance](GOVERNANCE.md) | How the standard evolves and how to take part |

## In one paragraph

A Ghost system is local first: a site runs on its own network, without needing the Internet. Every device carries a permanent identity created at manufacturing, which people can read on its label and systems can verify. Devices describe their own capabilities; gateways connect field devices to a site server; the site server decides which devices it accepts. Firmware and identities come from a manufacturing chain that can be audited from end to end. Anything beyond the site (several sites, remote access, a cloud service) is optional and chosen by the owner of the site.
