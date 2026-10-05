# Chapter 1 — Ethernet Fundamentals (Start from Zero)

> Goal: By the end of this chapter you understand what Ethernet is, the vocabulary (MAC, IP, port, PHY, switch), the OSI/TCP-IP models, and the Ethernet frame — **without assuming any prior Ethernet knowledge**.

---

## 1.1 What is Ethernet and Why It Is Used

**Simple explanation:**
Ethernet is a set of rules (a *standard*) for connecting computers/devices with cables so they can exchange data. Think of it as the "language + wiring rules" that lets many devices talk on the same network.

When you plug a laptop into an office wall socket or a router with a cable, you are using Ethernet.

**Technical explanation:**
Ethernet is defined by the **IEEE 802.3** family of standards. It specifies:
- The **physical layer** (cables, connectors, signaling, voltage levels).
- The **data link layer** (how bits are grouped into *frames*, addressing via MAC, error detection via FCS).

**Why it is used:**
- **High speed** — from 10 Mbit/s up to 400 Gbit/s (automotive: 10 Mbit/s to 10 Gbit/s).
- **Scalable** — connect 2 devices or thousands.
- **Cheap & standardized** — same tech in PCs, servers, and now cars.
- **Full-duplex** — send and receive at the same time (unlike CAN).

🧩 **CAN comparison:** CAN was designed for *short, prioritized control messages* on a shared bus at up to 1 Mbit/s (8 Mbit/s with CAN FD data phase). Ethernet is designed for *high-bandwidth data* (cameras, radar, software updates, service-oriented communication) that CAN simply cannot carry.

---

## 1.2 Ethernet vs Internet vs TCP/IP

Beginners mix these three up constantly. Here is the clean distinction:

| Term | What it is | Scope | Analogy |
|---|---|---|---|
| **Ethernet** | Wiring + framing rules to move data between devices on a **local** link | One local network (LAN) | The road inside your city |
| **TCP/IP** | The family of protocols that let data travel **across networks** (addressing, routing, reliability) | End-to-end, across networks | The postal rules: addresses, delivery, tracking |
| **Internet** | The **global** interconnection of millions of networks using TCP/IP | Worldwide | The entire global road + postal system |

**Key point:** Ethernet is *how* devices are physically connected and framed locally. TCP/IP is *how* the data is addressed and delivered, possibly across many networks. The Internet is the giant result of connecting everything with TCP/IP.

> In a car, you almost never touch "the Internet." You use **Ethernet** (the wiring/framing) plus **TCP/IP** (IP, UDP, TCP) plus **automotive protocols** (SOME/IP, DoIP) on top.

---

## 1.3 LAN, WAN, Network, Node, Host

- **Network** — any group of devices that can communicate with each other.
- **LAN (Local Area Network)** — a *small, local* network (home, office, one car). Ethernet is the classic LAN technology.
- **WAN (Wide Area Network)** — a *large* network spanning cities/countries (the Internet is the biggest WAN).
- **Node** — any device connected to a network (PC, switch, ECU, printer).
- **Host** — a node that *runs applications* and has an IP address (a PC, a server, an ECU). A pure switch is a node but not usually called a host.

🧩 **CAN comparison:**
- A CAN **bus** ≈ a LAN (one local network segment).
- A CAN **node** ≈ an Ethernet **host/ECU**.
- CAN has no real WAN concept; Ethernet naturally scales from LAN to WAN.

```
LAN (one car's Ethernet backbone)
┌─────────────────────────────────────────┐
│  [ECU A]───┐                             │
│            ├──[ Ethernet Switch ]──[ECU C]│
│  [ECU B]───┘                             │
└─────────────────────────────────────────┘
Each ECU = node/host.  Switch = node (not a host).
```

---

## 1.4 MAC Address, IP Address, Port Number 🟢

These three together answer: *which device, on which network, for which application?*

### MAC Address (Layer 2 — Data Link)
- A **48-bit (6-byte)** hardware address burned into the network interface.
- Written as `AA:BB:CC:DD:EE:FF` (hex).
- First 3 bytes = **OUI** (manufacturer), last 3 = device-unique.
- Used to deliver a frame **within one local network** (one link/switch domain).

### IP Address (Layer 3 — Network)
- A **logical** address used to route data **across networks**.
- IPv4: `192.168.1.10` (32-bit). IPv6: `2001:db8::1` (128-bit).
- Can change; assigned by config or DHCP.

### Port Number (Layer 4 — Transport)
- A **16-bit** number (0–65535) identifying *which application/service* on the host.
- Example: web server = port 80/443, SOME/IP often uses a configured UDP/TCP port, DoIP uses **TCP/UDP 13400**.

**Putting it together — the "postal" analogy:**

| Layer | Answers | Analogy |
|---|---|---|
| MAC | Which device on this local link | The specific mailbox on this street |
| IP | Which host, possibly on another network | The full postal address (city, street, house) |
| Port | Which app/service on that host | The person/department in the house |

```
Send data to  IP 192.168.0.20 : port 13400
                 │              │
                 │              └── application (DoIP server)
                 └── host (target ECU)
Delivered locally using the target's MAC address.
```

🧩 **CAN comparison:** In CAN there is **no device address at all**. The 11/29-bit **CAN ID identifies the message/content**, not the sender or receiver. In Ethernet, addressing is explicit: MAC = device, IP = host, port = app.

---

## 1.5 The Physical Building Blocks: Cable, Connector, PHY, MAC, Switch, Router

| Block | Simple meaning | Technical role |
|---|---|---|
| **Ethernet cable** | The wire | Carries electrical signals (twisted pair copper) |
| **Connector** | The plug | IT: RJ45. Automotive: rugged connectors (not RJ45) |
| **PHY** | Physical layer chip | Converts digital bits ↔ analog signals on the wire |
| **MAC** | Media Access Controller | Builds/parses frames, handles addressing at Layer 2 |
| **Switch** | Local traffic director | Forwards frames to the correct port using MAC table |
| **Router** | Between-networks director | Forwards packets between different IP networks |

**How PHY and MAC relate (very important for automotive):**
```
[ Software / IP stack ]
        │
     [ MAC ]   ← Layer 2: framing, addressing (often inside the microcontroller/SoC)
        │  (MII / RMII / RGMII / SGMII interface)
     [ PHY ]   ← Layer 1: signaling on the wire (separate chip or integrated)
        │
   [ Ethernet cable ]
```

🧩 **CAN comparison:**
- Ethernet **MAC** ≈ CAN **controller** (framing, addressing/arbitration logic).
- Ethernet **PHY** ≈ CAN **transceiver** (drives the physical wire).
- Ethernet **switch** — *no CAN equivalent*; CAN is a shared bus with no switching.
- Ethernet **router** ≈ conceptually like a **CAN gateway** but at the IP layer.

**Switch vs Router in one line:**
- **Switch** = works with **MAC** addresses, connects devices **within one network** (Layer 2).
- **Router** = works with **IP** addresses, connects **different networks** (Layer 3).

---

## 1.6 OSI Model and TCP/IP Model 🟢

The **OSI model** is a 7-layer conceptual reference. The **TCP/IP model** is the 4-layer practical model actually used. Interviewers love this.

```
OSI (7 layers)              TCP/IP (4 layers)         Automotive example
────────────────────────────────────────────────────────────────────────
7 Application  ┐
6 Presentation ├─────────►  Application              SOME/IP, DoIP, UDS, DHCP
5 Session      ┘
4 Transport    ──────────►  Transport                TCP, UDP
3 Network      ──────────►  Internet (Network)       IP, ICMP, ARP*
2 Data Link    ┐
1 Physical     ┴─────────►  Network Access / Link    Ethernet MAC + PHY
```
\*ARP sits between L2 and L3 in practice.

**Remember each OSI layer (mnemonic):** *All People Seem To Need Data Processing* (App, Presentation, Session, Transport, Network, Data-link, Physical).

| Layer | Job | Data unit | Address used |
|---|---|---|---|
| 7 Application | App protocols | Data | — |
| 4 Transport | End-to-end delivery, ports | Segment (TCP) / Datagram (UDP) | Port |
| 3 Network | Routing across networks | Packet | IP |
| 2 Data Link | Local delivery, framing | Frame | MAC |
| 1 Physical | Bits on the wire | Bits | — |

🧩 **CAN comparison:** CAN only really defines **Layer 1 (physical)** and **Layer 2 (data link)**. Everything above (like ISO-TP/UDS) is added by higher-layer standards. Ethernet gives you a full, layered stack up to the application.

---

## 1.7 Ethernet Frame Structure 🟢🧩

This is the single most important diagram to memorize. An **Ethernet II frame** looks like this:

```
┌──────────┬─────┬──────────────┬─────────────┬───────────┬───────────────┬─────┐
│ Preamble │ SFD │ Destination  │ Source MAC  │ EtherType │   Payload     │ FCS │
│  7 bytes │ 1 B │  MAC (6 B)   │   (6 B)     │  (2 B)    │  46–1500 B    │ 4 B │
└──────────┴─────┴──────────────┴─────────────┴───────────┴───────────────┴─────┘
   └── L1 (sync) ──┘└──────────────── L2 Frame (what MAC handles) ──────────────┘
```

Field-by-field:

| Field | Size | Purpose |
|---|---|---|
| **Preamble** | 7 bytes | `10101010...` pattern to let the receiver **synchronize** its clock |
| **SFD** (Start Frame Delimiter) | 1 byte | `10101011` — marks "the real frame starts now" |
| **Destination MAC** | 6 bytes | Who should receive this frame |
| **Source MAC** | 6 bytes | Who sent it |
| **EtherType** | 2 bytes | What's inside the payload (e.g. `0x0800` = IPv4, `0x0806` = ARP, `0x86DD` = IPv6, `0x88A8`/`0x8100` = VLAN) |
| **Payload** | 46–1500 bytes | The actual data (an IP packet, ARP, etc.). Min 46 (padded if smaller), max 1500 = **MTU** |
| **FCS** (Frame Check Sequence) | 4 bytes | **CRC-32** for error detection |

**Key numbers to memorize:**
- **MTU (Maximum Transmission Unit)** = 1500 bytes payload.
- **Minimum frame** = 64 bytes (from Dest MAC through FCS).
- **Jumbo frames** = >1500 bytes (used in some high-bandwidth cases, e.g. camera data).

🧩 **CAN comparison — side by side:**

| Aspect | CAN frame | Ethernet frame |
|---|---|---|
| Identifier | 11/29-bit CAN ID (content-based) | 48-bit source & dest MAC (device-based) |
| Max data | 8 bytes (CAN) / 64 bytes (CAN FD) | 1500 bytes (up to jumbo) |
| Error check | 15-bit CRC | 32-bit CRC (FCS) |
| Who receives | Everyone (filtered by ID) | Only the addressed device (via switch) |
| Direction | Half-duplex shared bus | Full-duplex point-to-point |

---

## 1.8 Unicast, Multicast, Broadcast 🟢

These describe **how many receivers** a frame targets — decided by the **Destination MAC**.

| Type | Meaning | Destination MAC | Automotive use |
|---|---|---|---|
| **Unicast** | One-to-one | A specific device MAC | DoIP diagnostics, TCP streams |
| **Multicast** | One-to-many (a group) | Group MAC (`01:00:5E:...` for IPv4) | **SOME/IP events / service discovery**, AVB audio/video |
| **Broadcast** | One-to-all | `FF:FF:FF:FF:FF:FF` | ARP requests, DHCP discovery |

```
Unicast:    A ───────────► C           (only C gets it)

Multicast:  A ──┬────────► C            (only group members C & D)
                └────────► D

Broadcast:  A ──┬────────► B
                ├────────► C            (everyone on the network)
                └────────► D
```

🧩 **CAN comparison:** CAN is **inherently broadcast** — every node sees every frame and filters by ID. Ethernet lets you *choose* unicast/multicast/broadcast, which is far more bandwidth-efficient. Automotive Ethernet uses **multicast heavily** (SOME/IP publish-subscribe) to mimic CAN's "many listeners" model without flooding everyone.

---

## 1.9 Full Duplex vs Half Duplex 🟢🧩

- **Half duplex** — devices share one channel; only one can transmit at a time (like a walkie-talkie). Collisions possible → old Ethernet used **CSMA/CD** to handle them.
- **Full duplex** — separate send and receive paths; both directions simultaneously (like a phone call). **No collisions.**

**Modern Ethernet (including all Automotive Ethernet) is full duplex.** CSMA/CD is effectively obsolete.

🧩 **CAN comparison:** CAN is **half-duplex** — one shared bus, and **arbitration** (not collision detection) decides who wins when two nodes start at once. Ethernet full-duplex + switching removes contention entirely: each device has its own dedicated link to the switch.

> **Automotive Ethernet twist:** 100BASE-T1 / 1000BASE-T1 achieve **full duplex over a single twisted pair** using echo cancellation — a big deal for weight/cost (fewer wires). More on this in Chapter 5.

---

## 1.10 Bandwidth, Data Rate, Latency 🟢

| Term | Meaning | Example |
|---|---|---|
| **Data rate / speed** | Bits per second the link can signal | 100BASE-T1 = 100 Mbit/s |
| **Bandwidth** | Practical capacity (often used interchangeably with data rate) | ~100 Mbit/s minus overhead |
| **Throughput** | Actual useful data delivered per second | e.g. 94 Mbit/s after headers |
| **Latency** | Time for data to travel end-to-end | switch adds µs; TSN bounds it |
| **Jitter** | Variation in latency | critical for audio/video/control |

**Naming decode:** `100BASE-T1`
- `100` = 100 Mbit/s
- `BASE` = baseband (signal uses the whole wire, no modulation onto a carrier)
- `T1` = twisted pair, **1 pair** (automotive single-pair). (`T` alone/`TX` = IT twisted pair, typically 2/4 pairs.)

🧩 **CAN comparison:**
- CAN: up to **1 Mbit/s** (classic), CAN FD data phase up to ~**8 Mbit/s**.
- Automotive Ethernet: **10 / 100 / 1000 Mbit/s** and beyond — **100–1000× more bandwidth**.
- CAN latency is very deterministic due to priority arbitration; standard Ethernet latency is low but *not guaranteed* — which is exactly why **TSN** (Time-Sensitive Networking, Ch 5) exists to make Ethernet deterministic for control traffic.

---

## 1.11 Chapter Summary

- **Ethernet** = local wiring + framing rules (IEEE 802.3); **TCP/IP** = addressing/delivery on top; **Internet** = global TCP/IP network.
- Addressing has 3 levels: **MAC (device)**, **IP (host/network)**, **port (application)**.
- **MAC ↔ CAN controller**, **PHY ↔ CAN transceiver**, **switch = no CAN equivalent**.
- Memorize the **Ethernet II frame**: Preamble, SFD, Dst MAC, Src MAC, EtherType, Payload (46–1500), FCS (CRC-32).
- **Unicast / Multicast / Broadcast** chosen by destination MAC; automotive loves multicast.
- Modern Ethernet is **full-duplex, switched** — no arbitration, no collisions.
- Ethernet gives **100–1000× the bandwidth of CAN**; TSN adds determinism.

**Next:** [Chapter 2 — Teach Ethernet by Comparing with CAN](Chapter_02_Ethernet_vs_CAN.md), where we map every CAN concept you know to its Ethernet counterpart.
