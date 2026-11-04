---
id: "01HMCXW4001E6CHQ8AA7GC0ES1"
title_en: "Crossbar Bus Architecture: From Shared Bus to Full Matrix"
summary_en: "Notes on the crossbar bus that connects multiple masters and slaves. Covers the evolution from shared bus to hierarchical bus to full matrix, the PL301 structure and arbitration schemes, pipelined and burst transfers, QoS, scaling to NoC, and clock domain crossing."
---
## Chapter 4, Section 2: PL Crossbar Bus Overview

A **crossbar bus** is a high-performance interconnect that connects multiple masters to multiple slaves. It switches point-to-point connections dynamically, which enables **multiple simultaneous transfers**.

## How Buses Evolved

### Shared Bus

- All devices share a single bus
- Only one transfer at a time → bottleneck
- Simple to implement, low bandwidth

### Hierarchical Bus

- Layered into a system bus → local bus
- The AMBA AHB + APB structure

### Crossbar (Full Matrix)

- M masters × N slaves → **M×N switch matrix**
- Every master can talk to a different slave **at the same time**
- Bandwidth ↑, area and complexity ↑

![Bus evolution|full](/img/01HMCXW4001E6CHQ8AA7GC0ES1/98ecc750a0f8.webp)

## PL Crossbar Structure

**PL (Programmable Logic) crossbar**: the switch array is configured programmably.

- **Master ports**: CPU, DMA, GPU, etc.
- **Slave ports**: memory controller, peripherals, I/O
- A switch at each crosspoint → arbitration + routing

![PL crossbar structure|full](/img/01HMCXW4001E6CHQ8AA7GC0ES1/f5320a7b2172.webp)

## Arbitration

When several masters request the same slave, the arbiter picks one:

- **Fixed priority**: priorities are fixed
- **Round-robin**: fairness
- **Weighted**: by bandwidth share
- **QoS (Quality of Service)**: real-time requirements

![Arbitration|full](/img/01HMCXW4001E6CHQ8AA7GC0ES1/0aa7f27680a6.webp)

## Pipelined Transaction

A bus transaction is split into an **address phase** and a **data phase** and pipelined:
- Address and data overlap → 2x throughput
- The basis of AMBA AHB and AXI

![Pipelined transaction|full](/img/01HMCXW4001E6CHQ8AA7GC0ES1/cc37756c2bfa.webp)

## Burst Transfer

Several consecutive words are transferred with a single address phase:
- INCR4, INCR8, INCR16 (incrementing)
- WRAP4, WRAP8 (wrap-around, for cache line fills)

Essential for cache line fills.

![Burst transfer|full](/img/01HMCXW4001E6CHQ8AA7GC0ES1/fbb9ad97f74a.webp)

## QoS (Quality of Service)

**Latency guarantees** for real-time systems (video, audio):
- Deadline-based scheduling
- Bandwidth reservation
- Priority queues

In AMBA AXI this is implemented with the **AxQOS** signals.

![QoS|full](/img/01HMCXW4001E6CHQ8AA7GC0ES1/f8a1851b5c67.webp)

## Scaling the Crossbar: NoC (Network-on-Chip)

With many masters and slaves, a full crossbar becomes impractical → **Network-on-Chip**:
- Topologies such as mesh, torus, ring, and tree
- Router + link structure
- Packet-based routing

![Scaling to NoC|full](/img/01HMCXW4001E6CHQ8AA7GC0ES1/0bee37276607.webp)

## Clock Domain Crossing

If a master and a slave sit in different clock domains, an **asynchronous bridge** or a **FIFO** is required:
- Prevents metastability
- Handshake protocol (req/ack)

![CDC|full](/img/01HMCXW4001E6CHQ8AA7GC0ES1/26b82bb454ed.webp)

## Summary

| Concept | Description |
|------|------|
| **Shared Bus** | One common bus, simple |
| **Crossbar** | M×N switch matrix, multiple simultaneous transfers |
| **Arbitration** | Picks a master on contention |
| **Pipelined/Burst** | Techniques for higher bandwidth |
| **QoS** | Real-time guarantees |
| **NoC** | Interconnect for large-scale systems |

The crossbar is at the heart of modern SoCs. The AMBA AXI, ACE, and CHI protocols are all built on crossbars or NoCs.

![Crossbar overview|full](/img/01HMCXW4001E6CHQ8AA7GC0ES1/bea51ec818ce.webp)
