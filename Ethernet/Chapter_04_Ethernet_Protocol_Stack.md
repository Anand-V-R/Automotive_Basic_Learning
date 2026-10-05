# Chapter 4 — Ethernet Protocol Stack

> Goal: Understand the full layered stack and **exactly what happens, step by step, when one ECU sends data to another ECU**. We cover Ethernet II, ARP, IP, ICMP, TCP, UDP, and the automotive protocols SOME/IP and DoIP.

---

## 4.1 The Layered Picture (Encapsulation) 🟢

Each layer **wraps** the layer above with its own header. This is **encapsulation**.

```
Application data (e.g. SOME/IP message)
        │  + SOME/IP header
   ┌────▼─────────────────────────────────────────┐
   │ Transport (UDP or TCP)  + UDP/TCP header      │  → Segment/Datagram
   ├───────────────────────────────────────────────┤
   │ Network (IP)            + IP header            │  → Packet
   ├───────────────────────────────────────────────┤
   │ Data Link (Ethernet)    + Eth header + FCS     │  → Frame
   ├───────────────────────────────────────────────┤
   │ Physical (PHY)          bits on the wire       │  → Bits
   └───────────────────────────────────────────────┘
```

**On receive, it's the reverse (decapsulation):** each layer strips its header and hands the payload up.

**Full nesting of a SOME/IP-over-UDP frame:**
```
[ Ethernet header ][ IP header ][ UDP header ][ SOME/IP header ][ payload ][ FCS ]
   L2                 L3            L4             L7 (app)                   L2
```

🧩 CAN has essentially one header (ID + control) and 8/64 bytes of data. Ethernet nests **multiple headers** — this layering is what gives it flexibility and scale.

---

## 4.2 Layer 1 — Physical

- The **PHY** encodes bits into signals on twisted pair and manages **link-up**, speed, and (automotive) **master/slave** timing roles.
- IT Ethernet PHYs auto-negotiate; **automotive T1 PHYs** are configured (one side master, one slave).
- Delivers a raw bitstream framed by **Preamble + SFD**.

---

## 4.3 Layer 2 — Data Link (Ethernet II) 🟢

Recall the frame from Chapter 1:
```
[Preamble][SFD][Dst MAC][Src MAC][EtherType][Payload 46-1500][FCS]
```
- The **MAC** builds this frame, fills in addresses, computes **FCS (CRC-32)**.
- **EtherType** tells the receiver what's inside:

| EtherType | Payload |
|---|---|
| `0x0800` | IPv4 |
| `0x0806` | ARP |
| `0x86DD` | IPv6 |
| `0x8100` | VLAN-tagged (802.1Q) |

- On receive, the MAC checks FCS; **bad frames are dropped silently** (no L2 retransmission).

---

## 4.4 ARP — Bridging L2 and L3 🟢

Before sending an IP packet on the local subnet, the sender needs the destination's **MAC**. ARP resolves `IP → MAC` (see Ch 3.5).

```
Need to send to 192.168.0.20
  MAC known?  ── no ──► ARP request (broadcast) ──► ARP reply ──► cache MAC
              ── yes ─► build frame with dest MAC and send
```
ARP entries are cached; automotive ECUs with static config sometimes have **static ARP** to avoid runtime ARP delays.

---

## 4.5 Layer 3 — IP (Internet Protocol) 🟢

IP adds **logical addressing and routing**. Key IPv4 header fields:

| Field | Purpose |
|---|---|
| Version | 4 or 6 |
| Source IP / Destination IP | End-to-end addressing |
| TTL (Time To Live) | Hop limit; decremented per router (prevents loops) |
| Protocol | What's inside: **6 = TCP, 17 = UDP, 1 = ICMP** |
| Header checksum | Detects header corruption |
| Fragment fields | Support fragmentation if packet > MTU |

- IP is **best-effort** — no delivery guarantee, no ordering (that's TCP's job).
- **Fragmentation:** if a packet exceeds the link MTU (1500), it may be split and reassembled at the destination.

---

## 4.6 ICMP (Internet Control Message Protocol) 🔵

ICMP carries **diagnostics and error messages** for IP (it rides inside IP, protocol = 1).
- **Echo request/reply** = the basis of `ping`.
- **Destination unreachable**, **time exceeded** (used by `traceroute`).

🚗 Useful for basic reachability testing between ECUs and testers, but production traffic uses UDP/TCP.

---

## 4.7 Layer 4 — Transport: UDP and TCP 🟢

### UDP header (8 bytes — tiny)
```
[ Src Port (2B) ][ Dst Port (2B) ][ Length (2B) ][ Checksum (2B) ][ data... ]
```
- Connectionless, no retransmission, no ordering. Low latency.
- 🚗 Used for **SOME/IP events, notifications, and Service Discovery**, and periodic data.

### TCP header (20+ bytes)
Key fields: Src/Dst port, **Sequence number**, **Acknowledgment number**, **flags (SYN, ACK, FIN, RST, PSH)**, **window size** (flow control), checksum.
- Connection-oriented (**3-way handshake**), reliable, ordered, flow & congestion control.
- 🚗 Used for **DoIP** and large/critical SOME/IP transfers.

**Choosing UDP vs TCP (automotive rule of thumb):**
```
Periodic / event / streaming / SD ....... UDP
Diagnostics / bulk / must-not-lose ...... TCP
```

---

## 4.8 SOME/IP — Service-Oriented Communication 🟢🚗

**SOME/IP** = *Scalable service-Oriented MiddlewarE over IP*. It is the automotive middleware that lets ECUs **offer and consume services** over Ethernet (typically over UDP, sometimes TCP).

**Communication patterns:**
| Pattern | Meaning | Transport |
|---|---|---|
| **Method** (Request/Response) | Client calls a function on a server ECU | usually TCP or UDP |
| **Event** (Notification) | Server pushes data to subscribers | UDP (often multicast) |
| **Field** | A value with getter/setter + notifier | mixed |

**SOME/IP header (key fields):**
```
[ Message ID (Service ID + Method/Event ID) ]
[ Length ]
[ Request ID (Client ID + Session ID) ]
[ Protocol Ver ][ Interface Ver ][ Message Type ][ Return Code ]
[ Payload (serialized parameters) ]
```
- **Service ID** = which service; **Method/Event ID** = which operation.
- **Message Type** = REQUEST, RESPONSE, NOTIFICATION, ERROR, etc.

**SOME/IP-SD (Service Discovery):** how clients *find* services at runtime.
```
Server ── OfferService (multicast) ──► network   "I provide Service 0x1234"
Client ── FindService  (multicast) ──► network   "Who offers 0x1234?"
Client ── Subscribe    ──────────────► Server    "Send me its events"
Server ── SubscribeAck ──────────────► Client
Server ── Event/Notification ────────► Client(s)  (ongoing)
```

🧩 **CAN comparison:** In CAN you broadcast every signal periodically regardless of who needs it (defined in the DBC). SOME/IP is **on-demand**: a service is offered, discovered, and only subscribed clients receive events. This slashes unnecessary traffic and enables flexible, reusable ECUs. Think of it as "DBC signals → structured service methods/events, negotiated at runtime."

---

## 4.9 DoIP — Diagnostics over IP 🟢🚗

**DoIP (ISO 13400)** carries **UDS diagnostics** over Ethernet — the Ethernet replacement for **UDS-on-CAN (ISO-TP/DoCAN)**.

- Uses **TCP/UDP port 13400**.
- UDP for **vehicle announcement/discovery**, TCP for the **diagnostic session** (reliable).

**DoIP flow:**
```
Tester                                     Vehicle (DoIP entity/gateway)
  │── Vehicle Identification (UDP bcast) ──►│
  │◄── Vehicle Announcement (UDP) ──────────│  (VIN, logical address, EID/GID)
  │── TCP connect (port 13400) ────────────►│
  │── Routing Activation Request ──────────►│
  │◄── Routing Activation Response ─────────│
  │── Diagnostic Message (UDS request) ────►│  e.g. 0x22 ReadDataByIdentifier
  │◄── Diagnostic Message (UDS response) ───│  e.g. 0x62 + data
```

**DoIP wraps UDS:**
```
[ Ethernet ][ IP ][ TCP ][ DoIP header ][ UDS message (SID + data) ]
```
- **Logical address** in DoIP identifies the target ECU (like the diagnostic address on CAN).
- The **DoIP gateway** routes UDS to the right internal ECU (over internal Ethernet or onto CAN).

🧩 **CAN comparison:** UDS-on-CAN uses **ISO-TP (ISO 15765)** to segment messages across 8-byte frames. DoIP uses **TCP** (which already handles segmentation/reliability), so there's no ISO-TP — TCP does that work. Same UDS services (`0x10`, `0x22`, `0x2E`, `0x27`, `0x34/36/37`, etc.), different transport.

---

## 4.10 UDP/TCP-Based Automotive Communication Summary

| Protocol | Transport | Purpose |
|---|---|---|
| **SOME/IP** | UDP (events/SD), TCP (large/methods) | Service-oriented app communication |
| **SOME/IP-SD** | UDP multicast | Service discovery, subscribe |
| **DoIP** | TCP (session) + UDP (discovery) | UDS diagnostics over IP |
| **AVB/TSN streams** | dedicated (AVTP) | Audio/video, time-critical (Ch 5) |
| **DHCP / ARP / ICMP** | infra | Config & housekeeping |

---

## 4.11 Step-by-Step: One ECU Sends Data to Another ECU 🟢

**Scenario:** ECU-A (`192.168.0.10`) sends a **SOME/IP event** to ECU-B (`192.168.0.20`) over UDP through an Ethernet switch. Same subnet.

```
┌ ECU-A ─────────────────────────────────────────────────────────────┐
│ 1. App/SWC produces data → hands to SOME/IP middleware              │
│ 2. SOME/IP: serialize params, add SOME/IP header (Service/Method ID)│
│ 3. UDP: add UDP header (src port, dst port 30501)                   │
│ 4. IP: add IP header (src 192.168.0.10, dst 192.168.0.20, proto 17) │
│ 5. Need dst MAC? → check ARP cache → (if missing) ARP request/reply │
│ 6. Ethernet MAC: build frame (dst MAC B, src MAC A, EtherType 0x0800│
│    [+ VLAN tag if configured]), compute FCS                         │
│ 7. PHY: encode bits, transmit on the wire                          │
└─────────────────────────────────────────────────────────────────────┘
                              │
                     ┌────────▼────────┐
                     │  Ethernet Switch │  reads dst MAC → looks up MAC table
                     │  forwards only   │  → sends frame out ECU-B's port only
                     └────────┬────────┘  (applies VLAN/TSN priority if set)
                              │
┌ ECU-B ─────────────────────▼───────────────────────────────────────┐
│ 8. PHY: receive bits → frame                                        │
│ 9. MAC: check FCS (drop if bad), check dst MAC = mine, strip L2     │
│10. IP: check dst IP = mine, proto=17 → hand to UDP                  │
│11. UDP: check dst port → hand to SOME/IP socket                     │
│12. SOME/IP: parse header (Service/Method), deserialize payload      │
│13. Deliver to subscribed App/SWC                                    │
└─────────────────────────────────────────────────────────────────────┘
```

**If it were TCP (e.g. DoIP)**, add before step 2: a **3-way handshake** to establish the connection, and after: **ACKs** and possible **retransmissions** for reliability.

🧩 **Contrast with the CAN flow you know:**
```
CAN:  SWC → RTE → Com → PduR → CanIf → CAN Drv → transceiver → BUS
      (every node hears it, filters by CAN ID)

Eth:  SWC → RTE → SOME/IP → PduR → SoAd → TcpIp → EthIf → Eth Drv → PHY → SWITCH
      (switch delivers only to the addressed ECU; app identified by port)
```
Chapter 6 maps this AUTOSAR stack module-by-module.

---

## 4.12 Chapter Summary

- **Encapsulation**: each layer adds a header — App(SOME/IP) → Transport(UDP/TCP) → IP → Ethernet → PHY.
- **EtherType** (L2) and **IP Protocol field** (L3) and **port** (L4) each say "what's next / which app."
- **ARP** resolves IP→MAC before L2 delivery.
- **UDP** = fast/connectionless (SOME/IP events, SD); **TCP** = reliable/ordered (DoIP, bulk).
- **SOME/IP** = service-oriented middleware (methods/events/fields + Service Discovery), replacing CAN's static broadcast model.
- **DoIP** = UDS over IP (TCP 13400), replacing UDS-on-CAN (no ISO-TP needed — TCP handles segmentation).
- You can now narrate the **full send→switch→receive** path for both UDP and TCP.

**Next:** [Chapter 5 — Automotive Ethernet](Chapter_05_Automotive_Ethernet.md) — 100BASE-T1, zonal architecture, TSN, AVB, PTP, and security.
