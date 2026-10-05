# Chapter 7 — Practical Learning

> Goal: Hands-on skills — two ECUs through a switch, sending a SOME/IP message, DoIP diagnostics, and above all **reading packets in Wireshark** and troubleshooting real problems. This is what makes you effective on day one.

---

## 7.1 Example: Two ECUs Communicating Through an Ethernet Switch 🟢

**Setup:**
```
ECU-A                         Switch                        ECU-B
IP 192.168.0.10 ── port1 ──[  MAC table  ]── port2 ── IP 192.168.0.20
MAC AA:..:0A                                          MAC AA:..:0B
(same subnet /24, one VLAN)
```

**What must be configured for them to talk:**
1. **IP + subnet mask** on both (same subnet → no router needed).
2. **Link parameters** match: 100BASE-T1, one PHY **master**, one **slave**.
3. **VLAN** — if used, both ports on the same VLAN (or tagging configured).
4. **Application endpoints** — matching UDP/TCP ports (e.g. SOME/IP service port).

**First message sequence:**
```
ECU-A wants to send to 192.168.0.20
1. ARP: "Who has 192.168.0.20?" (broadcast) → ECU-B replies with its MAC
2. ECU-A builds frame (dst MAC B) → switch learns "MAC A on port1"
3. Switch forwards to port2 (learns/knows MAC B there)
4. ECU-B receives, replies (switch already knows both MACs → direct forward)
```

🧩 Compare: on CAN you'd just put a frame with a CAN ID on the bus and every node filters. Here, **ARP + switch learning** set up point-to-point delivery first.

---

## 7.2 Example: Sending a SOME/IP Message 🟢🚗

**Scenario:** ECU-B offers a service `EngineData` (Service ID `0x1234`) with an event `Speed` (Event ID `0x8001`). ECU-A subscribes.

```
1. ECU-B (server): OfferService 0x1234  (SOME/IP-SD, UDP multicast)
2. ECU-A (client): sees the offer → sends Subscribe (event group)
3. ECU-B: SubscribeAck
4. ECU-B: periodically sends Notification (Speed event) to ECU-A (UDP)
```

**The SOME/IP event frame on the wire:**
```
[ Eth ][ IP (src B, dst A or multicast) ][ UDP (dst port 30501) ]
[ SOME/IP: MsgID=0x1234_8001, Length, RequestID, MsgType=NOTIFICATION, RetCode=0 ]
[ Payload: serialized Speed value (e.g. 0x1F40 = 8000) ]
```

**A method call (request/response), e.g. `GetVin`:**
```
Client → Request  (MsgType=REQUEST, RequestID = ClientID+SessionID)
Server → Response (MsgType=RESPONSE, same RequestID, RetCode=0x00, payload=VIN)
```

🧩 **DBC vs SOME/IP:** In DBC you'd define a periodic CAN message with a Speed signal. In SOME/IP, Speed is an **event** of a **service** that clients **subscribe** to — sent only to subscribers.

---

## 7.3 Example: Diagnostic Communication Using DoIP 🟢🚗

**Scenario:** A tester reads the VIN from an ECU using UDS `0x22` (ReadDataByIdentifier) over DoIP.

```
Tester (192.168.0.100)                       Vehicle Gateway (192.168.0.1, logical addr 0x1000)
  │── UDP: Vehicle Identification Request ─────►│
  │◄── UDP: Vehicle Announcement (VIN, addr) ───│
  │── TCP connect :13400 ──────────────────────►│
  │── DoIP Routing Activation Request ─────────►│
  │◄── DoIP Routing Activation Response (OK) ───│
  │── DoIP Diag Msg: UDS 22 F1 90 ─────────────►│   (Read VIN, DID 0xF190)
  │◄── DoIP Diag Msg: UDS 62 F1 90 <VIN...> ────│   (positive response 0x62)
```

**On the wire:**
```
[ Eth ][ IP ][ TCP :13400 ][ DoIP header (payload type=Diag Msg, source/target logical addr) ]
[ UDS: 22 F1 90 ]
```

🧩 **UDS-on-CAN vs DoIP:** Same UDS service (`0x22 → 0x62`). On CAN it'd be segmented by **ISO-TP** (First Frame/Consecutive Frames/Flow Control). On DoIP, **TCP** handles segmentation — no ISO-TP frames, and it's much faster (crucial for flashing/OTA).

---

## 7.4 Ethernet Packet / Frame Analysis Fundamentals 🟢

When you open a capture, read a packet **top-down through the layers**:

```
Frame 42: 128 bytes
  Ethernet II
    Destination: AA:BB:CC:00:00:0B
    Source:      AA:BB:CC:00:00:0A
    Type: IPv4 (0x0800)          ← EtherType tells you what's next
  Internet Protocol Version 4
    Source: 192.168.0.10
    Destination: 192.168.0.20
    Protocol: UDP (17)           ← IP protocol field tells you what's next
  User Datagram Protocol
    Source Port: 51000
    Destination Port: 30501      ← port tells you which app (SOME/IP)
  SOME/IP
    Service ID: 0x1234
    Method/Event ID: 0x8001
    Message Type: NOTIFICATION
    Payload: ...
```

**The "what's next" chain to always follow:**
`EtherType (L2) → IP Protocol (L3) → Port (L4) → app dissector (SOME/IP / DoIP)`

---

## 7.5 Basic Wireshark Usage 🟢

**Core workflow:**
1. **Select the interface** (or open a `.pcap`/`.pcapng` file, e.g. exported from CANoe.Ethernet).
2. **Start capture**, reproduce the issue, **stop**.
3. **Filter** to find relevant packets.
4. **Click a packet** → expand the layer tree (as above).
5. **Follow stream** for TCP/UDP conversations.

**Display filters you must know:**

| Filter | Shows |
|---|---|
| `ip.addr == 192.168.0.20` | All traffic to/from that IP |
| `eth.addr == AA:BB:CC:00:00:0B` | Traffic to/from that MAC |
| `udp.port == 30501` | SOME/IP on that port |
| `tcp.port == 13400` | DoIP traffic |
| `arp` | ARP requests/replies |
| `icmp` | ping / errors |
| `someip` | SOME/IP messages (with dissector) |
| `doip` | DoIP messages |
| `vlan.id == 10` | Traffic on VLAN 10 |
| `tcp.flags.syn == 1` | TCP handshake starts |
| `tcp.analysis.retransmission` | Retransmissions (problem indicator) |

**Handy columns to add:** VLAN ID, PCP priority, Delta time (for latency/jitter).

**Coloring/expert info:** Wireshark's **Expert Information** flags retransmissions, duplicate ACKs, checksum errors — your first stop for "something's wrong."

🧩 Wireshark is your **CANoe/CANalyzer trace equivalent** for Ethernet. (CANoe.Ethernet also decodes SOME/IP/DoIP and can export to pcap for Wireshark.)

---

## 7.6 How to Identify MAC / IP / Port / Protocol in a Packet 🟢

Cheat map for reading any frame:

| You want | Look at | Layer |
|---|---|---|
| Which devices (local) | `eth.src` / `eth.dst` | L2 |
| Which hosts | `ip.src` / `ip.dst` | L3 |
| Which protocol above IP | `ip.proto` (6=TCP, 17=UDP, 1=ICMP) | L3 |
| Which application | `udp.dstport` / `tcp.dstport` | L4 |
| Which service (auto) | SOME/IP Service/Method ID, or DoIP payload type | L7 |
| VLAN / priority | `vlan.id`, `vlan.priority` (PCP) | L2 |

**Worked example — a subscribe not working:**
- You see `OfferService` from the server (good).
- You **don't** see a `Subscribe` from the client → client-side config/route issue.
- Or you see `Subscribe` but no `SubscribeAck` → server rejected (check eventgroup/config).

---

## 7.7 Example Packet Capture Analysis (Walkthrough) 🟢

**Symptom:** "ECU-A isn't receiving Speed events from ECU-B."

Step through the capture:
```
1. Filter: someip
   → Do you see OfferService (0x1234) from B?  NO → B isn't offering (config/startup).
                                               YES ↓
2. Do you see Subscribe from A?  NO → A can't see the offer.
   → Filter arp / check VLAN: is A on the same VLAN/subnet as the multicast?
   → Check multicast group membership (IGMP / static multicast on switch).
                                   YES ↓
3. Do you see SubscribeAck from B?  NO → B rejected (wrong eventgroup ID / port).
                                    YES ↓
4. Do you see Notification frames?  NO → app not producing / TSN gate/timing.
                                    YES → A receives but app ignores → app-side bug.
```

This **layer-by-layer elimination** is exactly the mindset interviewers want.

---

## 7.8 Basic Ethernet Configuration Concepts 🚗

What you typically configure on an automotive ECU/switch:
- **IP address, subnet mask, gateway** (usually static in ARXML).
- **PHY mode:** speed (100/1000BASE-T1), **master/slave role**.
- **VLAN:** port VLAN ID (PVID), tagging, allowed VLANs, **PCP priority**.
- **Multicast:** static multicast MAC entries / IGMP snooping for SOME/IP events.
- **Socket connections (SoAd):** local/remote IP:port, UDP/TCP, PDU mapping.
- **SOME/IP-SD:** service IDs, eventgroups, offer/subscribe timing, multicast SD address/port.
- **DoIP:** logical address, routing activation, VIN/EID/GID.
- **TSN:** traffic classes, time-aware schedule, gPTP domain.

🧩 This is the Ethernet analog of building your **CAN comm matrix** from a DBC — same discipline, richer parameters.

---

## 7.9 Common Debugging Problems and How to Troubleshoot 🟢

| Problem | Likely cause | How to find/fix |
|---|---|---|
| No link | PHY master/slave both same, cable, speed mismatch | Check PHY roles & link LED/register; both ends must agree |
| Can ping but app fails | Wrong port / firewall / app not listening | `netstat`, check SoAd socket config, Wireshark port |
| Can't ping peer | Wrong IP/subnet, VLAN mismatch, ARP fail | `ipconfig`, `arp -a`, check VLAN/subnet |
| SOME/IP event not received | SD offer/subscribe missing, multicast/VLAN, eventgroup ID | Filter `someip`, follow OfferService→Subscribe→Ack→Notify |
| DoIP session fails | Routing activation rejected, wrong logical address, no TCP | Filter `doip`, check routing activation response code |
| Intermittent high latency | Bursty bulk traffic starving critical frames | Check VLAN PCP / TSN scheduling; measure delta time |
| Sensor fusion timestamps off | gPTP not synced / grandmaster lost | Check PTP sync state, grandmaster election |
| Frames dropped | FCS errors (EMC/cable), switch buffer overflow | Wireshark checksum errors, switch port stats |
| Retransmissions / slow TCP | Packet loss, congestion | `tcp.analysis.retransmission`, check link quality |
| Duplicate IP | Two ECUs same static IP | Gratuitous ARP conflict, review IP plan |

**Golden debugging habit:** always capture with **Wireshark (or CANoe.Ethernet)** and reason **bottom-up** (link → IP → port → app), just like you'd read a CAN trace for bus-off/ACK errors.

---

## 7.10 Practice Exercises (Do These)

1. **Frame dissection:** Given a hex dump of an Ethernet frame, identify Dst MAC, Src MAC, EtherType, and whether it's IPv4/ARP.
2. **Subnet check:** Are `192.168.5.10/24` and `192.168.6.10/24` on the same subnet? What's needed to connect them?
3. **Wireshark filter writing:** Write filters to isolate (a) all DoIP, (b) SOME/IP on VLAN 20, (c) TCP retransmissions.
4. **SD flow:** Sketch the OfferService→Find→Subscribe→Ack→Notify sequence for a new client.
5. **DoIP vs ISO-TP:** Explain why DoIP has no ISO-TP frames but UDS-on-CAN does.
6. **Gateway routing:** Draw how a UDS request from a DoIP tester reaches a CAN-only ECU.

(Answers are embedded throughout Chapters 3–6; use them to self-check.)

---

## 7.11 Chapter Summary

- Two ECUs talk via **ARP + switch learning**, then unicast/multicast on matching IP/port.
- **SOME/IP**: OfferService → Subscribe → Ack → Notification; methods are Request/Response.
- **DoIP**: UDP discovery → TCP connect → routing activation → UDS diag messages (no ISO-TP; TCP segments).
- Read packets **top-down** and follow the **EtherType → IP proto → port → app** chain.
- Master Wireshark **display filters** and **Expert Info**; it's your Ethernet trace tool (like CANoe for CAN).
- Debug **bottom-up** (link → IP/VLAN → port → app); know the common failure table cold.

**Next:** [Chapter 8 — Interview Preparation](Chapter_08_Interview_Preparation.md) — 175+ Q&A across all topics.
