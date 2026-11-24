# Identity

## One device, one permanent identity

Every Ghost device receives an identity when it is manufactured. The identity:

- is unique: no two devices share it;
- never changes during the life of the device, through firmware updates and factory resets;
- is stored in the device, apart from its firmware, so that one firmware fits every device of a product;
- is created by the manufacturing workshop, not by the device itself, and never computed from the electronic components.

## Three ways to recognise a device

| Form | For whom | Use |
|---|---|---|
| A **global identifier** | Systems | The identity itself, unique in the world. It settles every conflict. |
| A **network address** | Protocols | How the device is addressed on its site. It is derived from the global identifier. |
| A **readable code** | People | A short code of letters, with a check letter, that an installer reads on the label and compares with a screen. |

The label of a device shows the readable code in large letters, a code a phone or scanner can read, and the network address in small letters. The label is enough to restore the identity of a device whose memory was damaged. A label carries no secret: reading it gives no right over the device or the site.

## Origin of an identity

An identity can be **certified**: the manufacturer's identity authority vouches that it issued it. The site server checks this before it accepts a device, so that a device made outside the authorised chain is not taken in. Each site decides how strict this check is; production sites accept only certified devices.

## Uniqueness on a site

A site refuses a second device that presents an identity it already knows. Devices can move between sites (spare parts, after-sales), so identities are unique everywhere, not only on one site.
