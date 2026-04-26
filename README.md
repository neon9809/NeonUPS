# NeonUPS
## Open-Source Universal Modular Power Management System

[![中文文档](https://img.shields.io/badge/文档-中文-brightgreen?style=for-the-badge&logo=read-the-docs)](docs/README_zh-CN.md)

## 1. Background and Problem

Today, DC power supply in daily life is extremely fragmented. Every device comes with its own dedicated adapter, leading to cluttered power strips, messy cables, energy waste, and an inability to implement centralized management or ride-through during power outages. When a device is replaced, its power supply is often discarded entirely. The spread of smart homes has only magnified this pain point—curtain motors, sensors, and lights each trail their own independent power supply, turning wiring, maintenance, and emergency backup into constant headaches.

## 2. Core Philosophy

We envision a **highly modular, open-source intelligent power management system**. It functions as a power-supply building-block platform: a base mainframe with expandable slots that can be freely configured with standardized function modules—input, output, battery, IoT control, and more. Users can assemble their own dedicated power center like building with LEGO bricks, and manage everything intelligently via the network.

**Two fundamental principles:**

- **Sustainability-driven open source**: Communication protocols and specification documents are fully open. Anyone can manufacture compatible modules, eliminating wasteful replacement cycles.
- **On-demand modularity**: Every function is an independent hot-swappable module. If you need more power, parallel multiple mainframes; if you need more functions, add more modules. The system never becomes obsolete.

## 3. Typical Scenarios

- **Home energy center**: A shoebox-sized mainframe simultaneously powers the optical modem, router, NAS, smart speaker, DC lighting system, curtains, robot vacuum, and security devices. It centrally manages and distributes energy from the grid, photovoltaic system, and battery storage. During a power outage, the built-in battery provides seamless backup, keeping the network online.
- **Maker workbench**: Goodbye, messy bench power supplies. By inserting different modules, you quickly obtain multiple programmable outputs with adjustable voltage, USB fast charging, and battery storage. Every output is software-controllable.
- **Remote equipment hosting**: For devices left in a hometown home or a small server room, use a smartphone to remotely monitor the power status of each channel. When an anomaly occurs, remotely power-cycle or switch supplies.

## 4. Extended Preliminary Vision

### 4.1 Physical Electrical Architecture

**Bus Voltage and Topology**

The system uses **36 V DC** as its internal bus voltage, sitting below the 60 V safety extra-low voltage (SELV) limit while balancing transmission efficiency with user safety. The bus uses mainframe units as nodes, supporting star or multi-mainframe parallel topologies. Mainframes are interconnected through dedicated high-current DC connectors. When more power is needed, simply add another mainframe—capacity stacks like server power supplies.

**Connector Specification**

Power circuits and signal circuits are physically separated, each adopting mature, low-cost connectors:

- **Power input/output**: 5.5 mm DC jacks are used. Input and output ports are clearly differentiated by physical form or color (input uses a male plug, output uses a female socket) to prevent misconnection during hot-swap. A single connector is rated for 8 A continuous; high-power modules reserve a dual-port parallel design.
- **Communication bus**: The CAN bus is carried over an RJ45 (8P8C) physical connector with the locking tab removed to allow tool-free quick disconnect. Only one twisted pair inside the RJ45 is used for the CAN_H / CAN_L differential signal; the remaining pins are assigned to module presence detection, address coding, and reserved functions. **Although this connector looks like Ethernet, it is exclusively used for CAN communication in this system and must never be plugged into standard network equipment.**
- **Module-to-mainframe interface**: Each slot provides both power contacts and CAN signal contacts, supporting hot-swap. Critical power modules feature a software-locked retention mechanism—you must first “request removal” through the software, and the lock releases only after authorization, preventing accidental disconnection from disrupting operations.

**Module Form Factors**

- **Local modules**: Inserted directly into standard slots on the mainframe. Examples include input modules (AC-DC, solar MPPT), output modules (multi-channel adjustable voltage, USB PD), battery modules, and communication modules.
- **Remote modules**: Built as standard 86-type wall panels, installed at endpoints and connected back to the mainframe via 36 V DC wiring inside the walls. For instance, a Type-C PD panel module embedded beside a bed or sofa delivers fast charging while completely eliminating the need for a traditional dangling power adapter.

**Hardware Safety**

The system incorporates multiple layers of hardware protection at the electrical level: every module has built-in over-current, over-temperature, and short-circuit protection; the interconnection module (used when linking multiple mainframes) monitors the real-time current flowing through the link and enforces a hardware current limit; the carbon dioxide emergency module uses a **slow-release** mechanism—a micro solenoid valve and pressure regulator control the cylinder. When it simultaneously detects a high-temperature threshold and smoke particulates, it judges a fire risk, disconnects the affected circuit, and continuously releases carbon dioxide to suppress the fire, avoiding the shock and false-trigger risks of pyrotechnic discharge. The module contains a supercapacitor backup supply to ensure the valve can be opened even after the main power is cut.

**Ecosystem Bridging**

A series of converter modules will bring vast existing stocks of mature power supply systems onto the 36 V bus:

- **ATX power supply converter module**: Leverages the massive number of ATX power supplies to provide a stable, high-power source for the system.
- **Server power supply converter module**: Adapts the huge secondary-market inventory of hot-swappable server power modules, drastically lowering system power costs and giving makers an inexhaustible, cost-effective power source.

### 4.2 Communication Architecture

**Bus Protocol**

The system adopts **CAN bus** as the core communication protocol between modules and the mainframe, exploiting its native multi-master capability, non-destructive arbitration, and real-time performance. In the latest design, the **CAN controller is embedded in the mainframe**, while the module side only needs a CAN transceiver and minimal logic, dramatically reducing the development barrier and bill of materials for third-party modules. The mainframe is responsible for bus arbitration, module address assignment, heartbeat management, and policy execution.

**Module Addressing and Multi-Mainframe Interconnection**

Each module is automatically assigned a CAN ID by the mainframe via hardware address pins upon insertion, with no factory pre-configuration required. Multiple mainframes can operate in parallel on the same CAN bus. The CAN protocol natively supports the coexistence of multiple controllers, achieving conflict-free arbitration through frame ID priority.

Flexible redundancy collaboration among mainframes can be implemented at the application layer:

- **Hot standby mode**: One mainframe acts as the active controller sending commands, while another listens. If the active mainframe’s heartbeat is lost, the standby automatically takes over.
- **Load sharing**: Each mainframe manages its own group of modules without interfering with others.
- **Fully distributed**: Every mainframe can issue control commands, and modules respond according to their addresses.

**Distributed Watchdog Architecture**

Watchdog functionality is no longer confined to the mainframe; it is designed as a tiered service that users can configure on demand:

- **Hardware tier (module self-protection)**: Every module leaves the factory with built-in pure-hardware over-current and over-temperature protection that does not rely on any external signal, serving as the ultimate safety fallback.
- **Link tier (direct mainframe connection)**: The mainframe sends periodic heartbeat frames to the modules in its slots. If a module fails to receive a heartbeat within the timeout, it executes a pre-defined safe action. This is the fastest local software-level protection.
- **Service tier (programmable remote watchdog)**: Through a communication module, users can deploy custom watchdog policies from a server—defining the timeout period, retry count, and triggered action (restart, power off, switch supply). The communication module sends heartbeat frames on behalf of the server to the designated modules, making this ideal for remote hosting and maintenance scenarios.
- **Redundant cross-feeding**: The system can simultaneously host multiple communication modules (cellular, Wi‑Fi, Ethernet, LoRa), each independently maintaining watchdog heartbeats. Multiple network paths back each other up; a single link failure does not kill the watchdog. In extreme scenarios, you can configure multiple watchdog sources to watch over each other, requiring several to simultaneously declare failure before executing a final action, preventing a single false judgment from causing an erroneous power-down.

**Full-Module Observability**

All modules report their operating parameters in real time via the CAN bus:

- **Input/output modules**: Voltage, current, power, and switching state for each channel.
- **Battery module**: Total pack voltage as well as internal resistance, temperature, and state of health for each cell, so users can predict remaining runtime and overall life.
- **Interconnection module**: Real-time current, providing load awareness for multi-mainframe parallel arrays.

**Network Access and Redundancy**

Communication modules bridge the CAN bus data to external networks, enabling remote management. The system encourages inserting multiple communication modules of different types that back each other up, automatically switching when one network fails, ensuring continuous control for critical infrastructure.

## 5. About the Name "NeonUPS"

The project's full name is **Nimble Energy Operating Node**, abbreviated as **NEON**, combined with the common acronym **UPS** (Uninterruptible Power Supply) to form **NeonUPS**. This name captures the project's core identity across four dimensions:

- **Nimble** — Every function is implemented as an independent, hot-swappable module — users can freely assemble their power center like building blocks. The host is as compact as a shoebox, yet scales flexibly by paralleling multiple hosts, rejecting bulk and bloat.
- **Energy** — The project focuses not merely on supplying power, but on fine-grained management of multi-form energy — from grid AC, solar PV, to battery storage; from inputs, outputs, to the health of every single cell — all brought under unified, modular management.
- **Operating** — It is far more than a passive collector or object — it is a real-time, actively orchestrating energy hub. The system continuously executes output switching control, multi-source energy changeover, distributed watchdog strategies, and remote maintenance actions. Whether the grid fails or a device acts up, it runs, decides, and protects, keeping critical loads online.
- **Node** — Each host is an intelligent node in an energy network, achieving peer-to-peer communication and redundant cooperation between modules and hosts via CAN bus. Multiple nodes can flexibly form hot-standby, load-sharing, or fully distributed arrays, providing an infinitely stackable power foundation for homes, labs, and remote sites.

NeonUPS — more than power supply; it is an ever-online energy operating network built from nimble nodes.
