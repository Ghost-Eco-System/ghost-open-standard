# GOS RFC-0001: Core Principles

- **Status:** Stable v1.0
- **Category:** Foundation
- **Author:** Ghost System Community
- **Scope:** Architectural Doctrine
- **Applies to:** All Ghost-Compatible Systems

## 1. Introduction & Context

### 1.1 Purpose

The Ghost Open Standard (GOS) provides a lightweight, deterministic, and hardware-agnostic communication standard for the Internet of Things (IoT), edge computing, and distributed automation. As the foundational document of the GOS, this RFC establishes the immutable engineering doctrine, architectural constraints, and philosophical priorities that govern all subsequent Ghost specifications and implementations.

### 1.2 Motivation

Traditional IoT specifications frequently suffer from architectural drift, combining low-level transport mechanisms, high-level business logic, and device management protocols into unified, complex frameworks. This blending introduces tight coupling, rigid schemas, and severe backward-compatibility debt.

GOS breaks away from this approach by enforcing a strict separation between syntax (the protocol) and semantics (the business logic). This RFC serves as the "constitution" to prevent scope creep, ensuring that the core standard remains minimal, indestructible, and highly optimized for autonomous agentic engineering workflows.

## 2. Core Specification

### 2.1 The Axiom of Core Minimalism

The Ghost Open Standard (GOS) MUST remain strictly minimal. It standardizes _mechanisms of exchange_, never _policies of business_.

- The core standard provides universal primitives for identity, messaging envelopes, capability structures, discovery lifecycles, and error reporting.    
- The core standard MUST remain completely blind to the commercial or operational meaning of the data it transports. It handles structural validation but ignores business interpretation.



### 2.2 The Sovereign Node Principle (Source of Truth)

The physical producing node itself MUST be treated as the sole authoritative source of truth concerning its identity and capabilities.

- Centralized registries, platform brokers, or edge caches MUST be treated as non-authoritative projections or temporary cached views.    
- A node’s state and functional capabilities are dynamically declared by the node itself via live network interfaces, eliminating hardcoded centralized dependency chains.    


### 2.3 The Architectural Priority Ordering

When technical requirements conflict during the design or implementation of any Ghost-compatible system, engineers and autonomous agents MUST resolve trade-offs by strictly adhering to the following hierarchy:

$$\text{Correctness} > \text{Resilience} > \text{Observability} > \text{Performance} > \text{Features}$$

1. **Correctness (Safety):** System behavior MUST strictly match the formal specification. Data integrity and state predictability can never be compromised for speed or scope.    

2. **Resilience:** Systems MUST gracefully handle failures, network partitions, and invalid payloads without collapsing. A failure in a single sub-system must be isolated.

3. **Observability:** Systems MUST expose transparent, unambiguous state indicators, standard error contracts, and trace headers to ensure deep operational visibility.    

4. **Performance:** Execution speed, low latency, and memory footprint are highly valued, but only after correctness, resilience, and observability are fully guaranteed.

5. **Features:** Functional scope and application-specific capabilities are secondary to the underlying stability and correctness of the protocol engine.


## 3. Architecture & Data Structures

### 3.1 Layer Separation Contract

The Ghost Ecosystem divides responsibilities into distinct layers. The GOS public standard is strictly isolated at the bottom of the stack, serving as the interface contract for all proprietary or custom implementations above it.

```
┌────────────────────────────────────────────────────────┐ (Proprietary)
│                   BUSINESS LAYER                       │ e.g., Custom SaaS,
│       Application Logic, Contexts, Workflows           │ Custom Device,
└──────────────────────────┬─────────────────────────────┘ 
                           ▼
┌────────────────────────────────────────────────────────┐
│                   PLATFORM LAYER                       │ (Proprietary)
│      Shared Infrastructure, Brokers, Registries        │ e.g., mqtt
└──────────────────────────┬─────────────────────────────┘
                           ▼
┌────────────────────────────────────────────────────────┐ 
│                   RUNTIME LAYER                        │ (Proprietary)
│     Target-Specific, Engines, Component, Libraries     │ e.g., framework,
└──────────────────────────┬─────────────────────────────┘ 
                           ▼
==========================================================
┌────────────────────────────────────────────────────────┐
│              GOS (GHOST OPEN STANDARD)                 │ (OPEN SOURCE & PUBLIC)
│  RFCs, Core Schemas, Identity Primitives, Compliance   │
└────────────────────────────────────────────────────────┘
```

### 3.2 Data Flow Directionality Rule

To prevent hidden coupling, architectural dependencies MUST only flow downward toward the Core:

- **Business** components may depend on Platform, Runtime, and Core.    
- **Platform** components may depend on Runtime and Core.    
- **Runtime** components may depend on Core.    
- **Core (GOS)** MUST NOT depend on any external application, platform wrapper, or business entity. It is entirely self-contained.    

### 3.3 Network Topology

Throughout every level of the network topology—from the Cloud down to the Field—Ghost enforces a strict separation between the **Immutable Core** (which handles mechanisms, syntax, and transport) and the **Business Layer** (which handles proprietary logic, semantics, and user experience).

This separation is governed by the **Data Flow Directionality Rule**, which mandates that architectural dependencies must only flow downward: Business components can depend on the Core, but the Core must remain entirely self-contained and blind to any operational or business meaning.

Here is how this Core vs. Business separation manifests at each level of mapping:

**1. Field Level: Core <-> Business**

- **Core:** At the physical hardware layer, the core provides the indestructible C/C++ runtime. It handles the deterministic 36-byte Ghost Binary Protocol (GBP) frames, local hardware watchdogs, and the native Auto-Addressing lifecycle.

- **Business:** The business layer here is the physical sensing or actuating logic. For example, an engineer injects custom distance calculation logic into the field node's runtime without ever having to write or touch the underlying network code.

**2. Gateway Level: Core <-> Business**

- **Core:** The Gateway acts as the ultimate agnostic bridge. The core runs the Master Execution Loop, manages the Hardware Abstraction Layer (HAL) for physical transports (like Modbus or CANbus), and maintains a real-time, low-latency "Tactical Digital Twin" in memory. Crucially, it runs the Ghost Universal Mapping (GUM) engine to deterministically translate raw binary frames into rich JSON payloads.

- **Business:** At the Gateway edge, business logic no longer deals with bytes, registers, or watchdogs. It interacts exclusively with the translated "Semantic Digital Twin," allowing edge processing routines to operate purely on standardized JSON events.

**3. Edge & Cloud Level: Platform (Core) <-> SaaS (Business)**

- **Platform (Core):** In the platform or brain layer, the core is represented by the Ghost Open Standard (GOS). It provides the universal JSON message envelopes, identity persistence mechanisms (Ghost MAC), standard error contracts, and explicitly structured capability namespaces (e.g., `cap.distance`).

- **SaaS (Business):** The business layer is the high-level application, SaaS dashboard, or Autonomous AI Agent. Because the data arrives pre-formatted with rich semantic context, frontend teams or AI agents can immediately consume it. For example, a team can build a Node-RED dashboard to display "Occupied Space" warnings directly from the JSON envelope without needing to understand the underlying industrial protocols.

By sustaining this decoupled architecture from top to bottom, Ghost achieves what it calls **"simplexity"**—delivering the deterministic, low-power performance of raw binary on the hardware side, while presenting a perfectly clean, self-describing semantic interface to the cloud and AI agents on the business side.

### 3.4 Architectural Separation

At the macro scale of cloud and enterprise architectures, the divide between the **Core Vertical (Infrastructure)** and the **SaaS Business (Multi-tenant)** strictly adheres to Ghost's philosophical mandate: standardizing the _mechanisms of exchange_ while ignoring the _policies of business_.

Here is how this architectural separation manifests at the enterprise and cloud scale:

**1. Core Vertical (Infrastructure)**

- **The Immutable Engine:** At this level, the infrastructure acts as the foundational, agnostic engine that enforces the Ghost Open Standard (GOS). It is responsible for universal primitives like routing envelopes, capability structures, discovery lifecycles, and standard error contracts.

- **Business Blindness:** The core infrastructure must remain completely blind to the commercial, multi-tenant, or operational meaning of the data it transports. It cares only about structural validation and data invariance.

- **Decoupled Governance:** Standard compliance lives strictly in the open-source domain, ensuring that the foundational infrastructure remains vendor-independent and highly stable without being bloated by application-specific features or localized bug fixes.

**2. SaaS Business (Multi-tenant)**

- **The Proprietary Vertical:** This is where commercial value, multi-tenant architectures, and specific industry verticals. All proprietary and use-case-specific logic is isolated exclusively in the `/business` layer.

- **Semantic Consumption:** Because the core infrastructure guarantees that data arrives as rich, contract-validated JSON payloads bound to explicit namespaces (like `cap.distance`), SaaS applications and Autonomous AI Agents can instantly consume and act on the information. Software engineers can build multi-tenant applications or dashboards (using tools like Node-RED) to display real-world conditions without needing any knowledge of the underlying 36-byte binary frames or hardware watchdogs.

**The Bridge Between Them** This separation is rigidly enforced by Ghost's **Data Flow Directionality Rule**. Dependencies must only flow downward: the SaaS Business applications can depend on the Core Infrastructure, but the Core must remain entirely self-contained and free of any business entities or platform wrappers.

Ultimately, this allows SaaS vendors to scale complex, multi-tenant environments with complete creative freedom, relying on the core infrastructure's guarantee of uncorrupted data invariance from the physical field all the way up to the cloud.

### 3.5 Deployment Example

In the Ghost Ecosystem, the edge is entirely agnostic to the form factor or deployment method you choose. Whether you are running a **Docker container, a native application, a hybrid app, or a web application**, it fits perfectly into the architecture because Ghost dictates _how components communicate_, not the specific technologies used to build them.

Here is how those different edge applications integrate into the Ghost architecture:

**1. Agnostic Runtime Environments** The Ghost ecosystem dictates that regardless of the language or runtime environment, all software implementations must strictly adhere to a logic separation between the "Immutable Core" and the "Business Extend" layer. The Edge Gateway itself is fundamentally an **architectural abstraction**; it is designed to execute on anything from a resource-constrained ESP32 to a Raspberry Pi or a heavy industrial server. If industrial server at the edge is running Docker or orchestrating hybrid apps, it simply acts as the host for the Business layer.

**2. All Apps Live in the Business Layer** Docker containers, web apps, and native apps represent proprietary use-case logic. Under Ghost's ecosystem governance, all of this application-specific code must live exclusively in the `/business` layer. This ensures that specific edge applications can evolve, scale, or be swapped out completely without ever forcing modifications to the underlying core routing or protocol engine.

**3. Seamless Consumption via the Semantic Digital Twin** The true power of this setup is how easily edge apps can consume hardware data. Legacy edge architectures often forced applications to directly parse cryptic Modbus registers or CANbus frames. In Ghost, the Gateway's core runs the **Ghost Universal Mapping (GUM) engine**, which automatically translates raw 36-byte binary hardware frames into a rich, JSON-based **Semantic Digital Twin**.

Because of this, Dockerized microservice, web app, or Node-RED dashboard at the edge can simply subscribe to clean JSON namespaces like `cap.distance` or `cap.temperature`. Web developers do not need to know anything about the underlying hardware watchdogs, physical bus addresses, or binary protocols.

**4. The Only Constraint: Directionality** The only rule Docker or Web apps must follow is the **Data Flow Directionality Rule**. The dependencies must only flow downward: business applications can depend on the Core infrastructure to feed them data, but the Core must remain entirely self-contained and blind to the existence of specific Docker container or Web App.


## 4. Backward Compatibility & Evolution

### 4.1 Immutable Core Rules

Modifications to the `Core` MUST NOT be introduced to resolve application-specific or localized hardware bugs. Once a GOS feature version is marked as `Stable`, its fundamental primitives are locked.

### 4.2 Version Representation

- **Standard Evolution:** The Ghost Open Standard specification versioning uses a strict `MAJOR.MINOR` format (e.g., `1.0`, `1.1`).    
- **Document Presentation:** The prefix `v` (e.g., `v1.0`) is strictly reserved for presentation layers, documentation titles, and repository tagging. It MUST NOT be used within raw data structures or network schemas.