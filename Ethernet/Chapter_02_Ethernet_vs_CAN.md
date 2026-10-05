# Chapter 2 — Teach Ethernet by Comparing with CAN

> Goal: Use everything you already know about CAN to learn Ethernet fast. For each concept: **CAN idea → Ethernet equivalent → key difference**. This is your fastest path to fluency. 🧩

---

## 2.0 The One Big Mental Shift

| | CAN | Ethernet |
|---|---|---|
| Topology | **Shared bus** — one wire, everyone listens | **Switched star** — each device has its own link to a switch |
| Addressing | **Message-based** (CAN ID = content) | **Device-based** (MAC/IP) + **app-based** (port) |
| Who talks | **Arbitration** (priority decides) | **Switch forwarding** (no contention, full duplex) |
| Data size | 8 B (CAN) / 64 B (CAN FD) | up to 1500 B (jumbo more) |
| Speed | ≤ 1 Mbit/s (8 in FD data) | 10 Mbit/s – 10 Gbit/s |

**In one sentence:** CAN organizes traffic by *message priority on a shared wire*; Ethernet organizes traffic by *addresses on dedicated switched links*.

---

## 2.1 CAN ID vs MAC / IP 🟢

| CAN | Ethernet |
|---|---|
| **CAN ID** identifies the *message/content* and its *priority*. It does **not** say who sent it or who should receive it. | **MAC** identifies the *device* (Layer 2). **IP** identifies the *host/network* (Layer 3). **Port** identifies the *application* (Layer 4). |

- In CAN, receivers use **acceptance filters** on the CAN ID to decide "is this message for me?"
- In Ethernet, the **switch** already sends the frame only to the correct port (by dest MAC), and the host checks the IP/port to route it to the right app.

**Example:**
- CAN: `0x123` = "EngineSpeed" — any node that cares filters for `0x123`.
- Ethernet: "send EngineSpeed via SOME/IP" → target IP `192.168.0.20`, UDP port `30501`, delivered to MAC `AA:BB:...:20`.

> The publish-subscribe *feel* of CAN (many listeners to one message) is recreated in Ethernet using **multicast + SOME/IP Service Discovery** (Ch 5).

---

## 2.2 CAN Frame vs Ethernet Frame 🟢

```
CAN (Classic) frame:
[SOF][ Identifier ][RTR][Control][ Data 0-8B ][CRC-15][ACK][EOF]

Ethernet II frame:
[Preamble][SFD][Dst MAC 6B][Src MAC 6B][EtherType 2B][ Payload 46-1500B ][FCS CRC-32]
```

| Aspect | CAN | Ethernet |
|---|---|---|
| Identifier | 11/29-bit CAN ID | 6-byte Dst + 6-byte Src MAC |
| "What's inside" marker | — (implicit by ID) | **EtherType** (0x0800 IPv4, 0x0806 ARP, 0x86DD IPv6) |
| Max payload | 8 B / 64 B (FD) | 1500 B |
| CRC | 15-bit | 32-bit (FCS) |
| Acknowledge | In-frame **ACK slot** (bit-level) | **No L2 ack** — reliability handled higher (TCP) |
| Framing sync | SOF bit + bit stuffing | Preamble + SFD |

**Big difference:** CAN acknowledges every frame at the bit level on the bus. Ethernet has **no per-frame acknowledgment at Layer 2** — if you need guaranteed delivery you use **TCP** at Layer 4; UDP is fire-and-forget (like CAN's "just put it on the bus").

---

## 2.3 CAN Arbitration vs Ethernet Switching 🟢

**CAN arbitration:** When two nodes transmit simultaneously, the CAN ID bits arbitrate — the lower ID (more dominant bits) wins, the loser backs off and retries. This is **non-destructive bitwise arbitration** on a shared bus.

**Ethernet switching:** There is *no arbitration* in modern Ethernet. Each device has a **dedicated full-duplex link** to the switch. The switch:
1. Learns which MAC is on which port (**MAC address table**).
2. Forwards each frame **only** to the port where the destination lives.
3. **Buffers** frames if a port is busy (queuing), so there are no collisions.

```
CAN: everyone shares one wire, priority decides
   Node1 ─┐
   Node2 ─┼──── one shared bus ── arbitration
   Node3 ─┘

Ethernet: switch gives each its own link
   ECU1 ──┐
   ECU2 ──┼── [ SWITCH ] ── forwards by MAC, buffers, no collisions
   ECU3 ──┘
```

**Consequence:** CAN guarantees the highest-priority message gets through first (great for control). Plain Ethernet does **not** guarantee timing — a burst can fill a switch queue and delay a critical frame. **TSN** (Ch 5) adds priority/scheduling to bring CAN-like determinism to Ethernet.

---

## 2.4 CAN Bus vs Ethernet Network

| CAN bus | Ethernet network |
|---|---|
| One physical shared medium, all nodes tapped on | A **collection of point-to-point links** joined by switches |
| Adding a node = tap onto the bus (loads the bus) | Adding a device = plug into a free switch port |
| Bus length/speed tradeoff (propagation for arbitration) | Each link is independent; segment length limited by PHY |
| One "domain" = one bus | Segmented into **VLANs**/subnets for isolation |

🧩 A single car may have **many CAN buses** bridged by a gateway. Modern architectures replace those with **one (or few) Ethernet backbones** and switches, dramatically simplifying wiring (Ch 5, zonal architecture).

---

## 2.5 CAN Node vs Ethernet Host / ECU

| CAN node | Ethernet host / ECU |
|---|---|
| Has a CAN controller + transceiver | Has a MAC + PHY, plus a full **TCP/IP stack** |
| Identified only by the messages it sends/filters | Identified by **MAC**, **IP**, and offers services on **ports** |
| Stateless bus participant | Can hold connections (TCP), subscriptions (SOME/IP), sessions (DoIP) |

An Ethernet ECU is "heavier" — it runs an IP stack, may run TCP, SOME/IP, DoIP servers, etc. That's more capability but also more configuration.

---

## 2.6 CAN Controller vs Ethernet MAC

| CAN controller | Ethernet MAC (Media Access Controller) |
|---|---|
| Builds/parses CAN frames, handles arbitration, CRC, ACK, error states | Builds/parses Ethernet frames, handles MAC addressing, CRC (FCS), (historically CSMA/CD) |
| Usually inside the microcontroller | Usually inside the MCU/SoC; connects to PHY via **MII/RMII/RGMII/SGMII** |

Both are the **Layer 2 "brain."** The MAC does not touch the wire directly — it hands bits to the PHY.

---

## 2.7 CAN Transceiver vs Ethernet PHY

| CAN transceiver | Ethernet PHY (Physical Layer transceiver) |
|---|---|
| Converts controller's TX/RX logic ↔ differential CAN_H/CAN_L bus levels | Converts MAC's digital data ↔ line signaling on twisted pair |
| Handles bus dominant/recessive states | Handles encoding (PAM, echo cancellation for T1), link-up, auto-negotiation |
| Simple, 1 chip | More complex; T1 PHYs do full-duplex on a single pair |

🧩 **Perfect mental map:** `MAC : PHY` (Ethernet) = `controller : transceiver` (CAN). The MAC/controller is digital+logic; the PHY/transceiver is analog+wire.

---

## 2.8 CAN Gateway vs Ethernet Gateway / Router

| CAN gateway | Ethernet router / gateway ECU |
|---|---|
| Routes/translates messages **between CAN buses** (and to CAN FD, LIN, etc.) | **Router** forwards IP packets between IP networks/subnets; **Gateway ECU** bridges CAN/LIN/FlexRay ↔ Ethernet |
| Works on CAN IDs / signal mapping | Works on IP addresses (routing) and can translate protocols |

In a real car the **Gateway ECU** does both: it routes IP traffic *and* translates legacy CAN/LIN signals into SOME/IP or DoIP over Ethernet. This is a core interview topic for a CAN-to-Ethernet engineer.

---

## 2.9 CAN Bandwidth vs Ethernet Bandwidth

| | CAN | CAN FD | Automotive Ethernet |
|---|---|---|---|
| Speed | ≤ 1 Mbit/s | ≤ 8 Mbit/s (data) | 10 / 100 / 1000 Mbit/s (up to 10 Gbit/s) |
| Payload | 8 B | 64 B | up to 1500 B |
| Best for | Control signals | Faster control / small data | Cameras, radar, lidar, OTA, service comm |

Ethernet's bandwidth is what enables **ADAS sensors, infotainment, and OTA software updates** — things CAN physically cannot carry.

---

## 2.10 CAN Error Handling vs Ethernet Error Handling 🟢

| CAN | Ethernet |
|---|---|
| Rich, built-in: bit error, stuff error, CRC error, form error, ACK error | Frame error detection via **FCS (CRC-32)** — bad frames are **silently dropped** |
| **Error frames** actively signal problems on the bus | No error frames; no bus-wide signaling |
| **Error counters (TEC/REC)** → error-active / passive / **bus-off** | No node bus-off concept; link is point-to-point |
| Automatic retransmission by controller | **No L2 retransmission**; recovery is by **TCP** (L4) or the application |
| Fault confinement protects the whole bus | A faulty link only affects that one link/port |

**Key interview point:** CAN pushes error handling *down* into the controller (deterministic, automatic). Ethernet pushes reliability *up* the stack — Layer 2 only detects & drops; **TCP** provides retransmission/ordering, **UDP** provides none. So "reliable Ethernet communication" = choosing TCP or building app-level checks.

---

## 2.11 CAN Communication Flow vs Ethernet Communication Flow 🟢

**CAN (AUTOSAR) send flow:**
```
SWC → RTE → Com → PduR → CanIf → CAN Driver → CAN Transceiver → BUS
```

**Ethernet (AUTOSAR) send flow:**
```
SWC → RTE → Com / SOME-IP → PduR → SoAd → TcpIp → EthIf → Eth Driver → PHY → WIRE
```

Side-by-side (memorize this — it's the heart of Ch 6):

| Step | CAN stack | Ethernet stack |
|---|---|---|
| App | SWC | SWC |
| Middleware | RTE | RTE |
| Signal packing | Com | Com **or** SOME/IP |
| PDU routing | PduR | PduR |
| Interface | CanIf | **SoAd** (Socket Adaptor) + **TcpIp** |
| Driver | CAN Driver | **EthIf** + **Eth Driver** |
| Physical | CAN Transceiver | **PHY** |

Notice: **RTE, Com, PduR are reused** — your AUTOSAR knowledge transfers directly. The new pieces are **SoAd, TcpIp, EthIf, Eth**, plus **SOME/IP, DoIP, Sd** on top.

---

## 2.12 Master Comparison Table (Print This) 🟢

| Concept | CAN | Ethernet |
|---|---|---|
| Topology | Shared bus | Switched star, full-duplex |
| Address | CAN ID (message) | MAC / IP / port (device/host/app) |
| Media access | Bitwise arbitration | Switching (no arbitration) |
| Frame data | 8 B / 64 B FD | 46–1500 B |
| Speed | ≤ 1 (8 FD) Mbit/s | 10–10000 Mbit/s |
| Duplex | Half | Full |
| ACK | In-frame bit ACK | None at L2 (TCP at L4) |
| Error handling | Built-in, counters, bus-off | FCS detect+drop; recovery higher up |
| Retransmit | Automatic (controller) | TCP (L4) or app |
| Determinism | Native (priority) | Needs TSN |
| Controller | CAN controller | Ethernet MAC |
| Transceiver | CAN transceiver | Ethernet PHY |
| Multi-net link | CAN gateway | Router / gateway ECU |
| Config artifact | DBC / LDF | ARXML (SOME/IP, network config), fibex/SD config |
| Tooling | CANoe/CANalyzer | Wireshark, CANoe.Ethernet |

---

## 2.13 Chapter Summary

- The core shift: **shared-bus + arbitration + message-IDs (CAN)** → **switched star + addressing + full-duplex (Ethernet)**.
- **MAC ≈ CAN controller**, **PHY ≈ CAN transceiver**, **switch = new concept**, **router ≈ IP-level gateway**.
- Ethernet L2 only **detects** errors and **drops** bad frames; **reliability moves up to TCP**.
- The AUTOSAR flow **reuses RTE/Com/PduR** and adds **SoAd/TcpIp/EthIf/Eth** + **SOME/IP/DoIP/Sd**.
- CAN's "many listeners" is recreated with **multicast + SOME/IP Service Discovery**.

**Next:** [Chapter 3 — Networking Fundamentals](Chapter_03_Networking_Fundamentals.md) (IP, subnet, DHCP, ARP, DNS, TCP/UDP, VLAN, NAT).
