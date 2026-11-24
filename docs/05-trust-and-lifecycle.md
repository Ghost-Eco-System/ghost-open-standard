# Trust and lifecycle

## Life of a device

1. **Made.** A workshop assembles the board, flashes an approved firmware, gives the device its identity and prints its label. The device is recorded by its manufacturer.
2. **Installed.** An installer mounts and wires it. Its gateway discovers it; the site server checks its identity and accepts it.
3. **In service.** The device reports; the site server watches it and raises alarms.
4. **Maintained.** A device can be updated, repaired, have its identity restored, or be replaced. Every intervention is recorded.
5. **Retired.** A retired identity is never given to another device.

## Firmware

- Firmware is built by an automated process from reviewed code, never on a personal computer.
- Every released firmware is **signed**, and the tools that install it check the signature and the integrity of every file before writing anything to a device.
- Updating the firmware never erases the identity of a device.

## Trust

- Sites trust a small set of **authorities**: those that sign firmware and those that certify identities. The list of authorities, and of anything withdrawn, is delivered to sites even when they have no Internet access.
- Nothing that could create a trusted identity is ever kept in a workshop computer, a device or a site.
- People get access according to what they do: making devices, installing them, operating a site or developing firmware. Access is given per person and can be removed at any time.

## Records

Each manufacturer keeps a record of every device it produced: what it is, when and where it was made, and what happened to it since. The record says what a device is; the site server says where it is installed. Records never contain data about customers or sites.
