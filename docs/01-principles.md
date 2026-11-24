# Principles

## Why a standard

Buildings, car parks and industrial sites use many devices from many suppliers. Each supplier tends to bring its own way of naming devices, describing data and reporting faults. The Ghost Open Standard gives them one common way, so that a site can grow, change supplier or add a new kind of device without being rebuilt.

## Guiding rules

- **Minimal core.** The standard defines only what is needed for systems to understand each other. Business rules (prices, schedules, how a car park is run) belong to applications, never to the standard.
- **The device is the source of truth about itself.** A device knows its own identity and what it can do, and tells it. Other systems learn it from the device; they do not guess it.
- **Layers depend downwards only.** Business applications rely on platforms, platforms on runtimes, runtimes on the standard; never the reverse.
- **Priorities, in order:** correctness, then resilience, then observability, then performance, then features. A feature that weakens correctness is refused.
- **Local first and sovereign.** Everything a site needs runs on the site. Data stays with the owner of the site unless the owner decides otherwise.
- **Nothing hidden in the field.** A device never carries a secret that a person could read on it, and a label never grants any right.
- **Auditable.** Every device and every firmware can be traced back to where and how it was made.
- **Implementation neutral.** The standard does not impose a language, a vendor or a tool. Implementations may be open or proprietary.

## What the standard is not

It is not a product, a cloud service or a programming framework. It does not define user interfaces, business processes or prices.
