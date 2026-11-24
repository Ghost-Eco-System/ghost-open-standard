# Architecture

## Layers of a site

| Layer | Role |
|---|---|
| **Field devices** | Sensors and actuators where things happen: a detector above a parking space, a relay, a display. They are small, robust and simple. |
| **Gateways** | Connect a group of field devices to the site network. They discover new devices, give them a local address, collect their readings and pass on commands. |
| **Site server** | The brain of the site. It knows every device, where it is installed and what it reports. It accepts or refuses devices, keeps the history, raises alarms and serves the screens of the site. |
| **Screens and tools** | Operator consoles, displays for the public, tools for installers and maintenance staff. They talk to the site server. |
| **Optional hub** | Above one or several sites, for owners who want it: several sites side by side, notifications, remote reports. |

A field device talks only to its gateway, a gateway only to its site server. Each layer can be replaced without changing the others.

## Deployment choices

The same system can be deployed in three ways, chosen by the owner of the site:

- **Isolated:** the site has no Internet access at all. Everything works locally. Updates and reports travel on physical media.
- **National:** the site may connect to a hub hosted in its own country, and nowhere else.
- **Cloud:** the site may connect to a hub operated as a service.

A site never depends on the hub to run. If the link is lost, the site keeps working and catches up later.

## Manufacturing and field work

Devices are produced in workshops that may belong to different manufacturers. A workshop flashes the firmware, gives each device its identity and prints its label. Installers then mount and wire the devices, and the site server takes them in. Maintenance staff replace or update devices on site. Each step is recorded so that the history of a device can be followed.
