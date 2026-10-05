# Chapter 9 — Final Revision

> Goal: One place for last-minute revision — cheat sheet, CAN-vs-Ethernet table, architecture and AUTOSAR-stack diagrams, glossary, quick-fire questions, and a 30-day roadmap.

---

## 9.1 Ethernet One-Page Cheat Sheet 🟢

**Addressing (3 levels):** MAC = device (L2, 48-bit) · IP = host/network (L3, 32/128-bit) · Port = app (L4, 16-bit).

**Ethernet II frame:**
```
[Preamble 7][SFD 1][Dst MAC 6][Src MAC 6][EtherType 2][Payload 46-1500][FCS 4]
   (+4-byte 802.1Q VLAN tag after Src MAC if tagged)
```
**Key numbers:** MTU 1500 · min frame 64 · FCS = CRC-32.

**EtherType:** 0x0800 IPv4 · 0x0806 ARP · 0x86DD IPv6 · 0x8100 VLAN.
**IP protocol:** 6 TCP · 17 UDP · 1 ICMP.
**Ports:** DoIP 13400 · SOME/IP-SD ~30490 (project-defined).

**Models:** OSI 7 layers (App·Pres·Sess·Trans·Net·DataLink·Phys) → TCP/IP 4 (App·Transport·Internet·Link).

**TCP handshake:** SYN → SYN-ACK → ACK. **Teardown:** FIN→ACK→FIN→ACK.
**UDP:** connectionless, 8-byte header, fire-and-forget (≈ CAN).

**Switch (L2/MAC):** learn·forward·flood·filter, full-duplex, no arbitration.
**Router (L3/IP):** forwards between networks, uses routing table + TTL.
**ARP:** IP→MAC on local subnet (IPv6 uses NDP).

**Automotive PHYs:** 10BASE-T1S (multidrop/PLCA, CAN-like edge) · 100BASE-T1 (workhorse) · 1000BASE-T1 (high BW). T1 = single pair, full duplex (echo cancel), master/slave roles.

**Automotive protocols:** SOME/IP (services: methods/events/fields + SD) · DoIP (UDS over TCP/IP).
**Determinism:** TSN (gPTP 802.1AS, TAS 802.1Qbv, preemption 802.1Qbu, CBS 802.1Qav) · AVB (A/V) · PTP time sync.
**Security:** MACsec · TLS/IPsec · firewall/IDS · SecOC · UDS 0x27.

**AUTOSAR Eth flow:** `SWC→RTE→Com/SOME-IP→PduR→SoAd→TcpIp→EthIf→Eth→PHY`.

**Debug order:** link → IP/subnet → VLAN → route → port/app → Wireshark capture.

---

## 9.2 CAN vs Ethernet Comparison Table 🟢

| Aspect | CAN | Ethernet |
|---|---|---|
| Topology | Shared bus | Switched star, full duplex |
| Access method | Bitwise arbitration | Switching (no arbitration) |
| Addressing | CAN ID (message) | MAC / IP / port |
| Max payload | 8 B (64 CAN FD) | 46–1500 B (jumbo more) |
| Speed | ≤1 Mbit/s (8 FD) | 10 Mbit/s–10 Gbit/s |
| Duplex | Half | Full |
| L2 ACK | In-frame bit ACK | None (TCP acks at L4) |
| Error handling | Counters, bus-off, error frames | FCS detect+drop; TCP recovery |
| Retransmission | Automatic (controller) | TCP (L4) or app |
| Determinism | Native (priority) | Needs TSN |
| Controller | CAN controller | Ethernet MAC |
| Transceiver | CAN transceiver | Ethernet PHY |
| Multi-net bridge | CAN gateway | Router / gateway ECU |
| Comm model | Signal broadcast (DBC) | Service-oriented (SOME/IP) |
| Diagnostics | UDS + ISO-TP (DoCAN) | UDS + DoIP |
| Time sync | AUTOSAR TimeSync (coarse) | gPTP (sub-µs) |
| Config artifact | DBC / LDF | ARXML / Fibex |
| Tooling | CANoe / CANalyzer | Wireshark / CANoe.Ethernet |
| AUTOSAR lower stack | CanIf, Can, CanTrcv, CanTp | SoAd, TcpIp, EthIf, Eth, EthTrcv, DoIP |

---

## 9.3 Automotive Ethernet Architecture Diagram (Zonal + Central Compute)

```
                        ┌──────────────────────────────┐
                        │   Central Compute (HPC(s))    │
                        │  ADAS · Infotainment · VehCtl │
                        └───────────────┬──────────────┘
                                        │  Gbit Ethernet backbone (VLAN + TSN + gPTP)
              ┌─────────────────┬───────┴───────┬─────────────────┐
        ┌─────┴─────┐     ┌─────┴─────┐   ┌─────┴─────┐     ┌─────┴─────┐
        │ Zone FL   │     │ Zone FR   │   │ Zone RL   │     │ Zone RR   │
        │ Controller│     │ Controller│   │ Controller│     │ Controller│
        └─────┬─────┘     └─────┬─────┘   └─────┬─────┘     └─────┬─────┘
   100/1000BASE-T1        local links     10BASE-T1S/CAN     CAN/LIN
        │                     │                 │                 │
   [Cameras/Radar]     [Sensors/Actuators]  [Edge sensors]   [Body/LIN slaves]

   Telematics/OTA ── [Gateway/Firewall] ── (secured boundary to cloud)
```
Key ideas: few powerful computers, an Ethernet **backbone**, **zonal controllers** aggregating nearby I/O, legacy **CAN/LIN at the edge**, secured **gateway** to the outside.

---

## 9.4 AUTOSAR Ethernet Stack Diagram 🟢

```
┌──────────────────────────── Application Layer ─────────────────────────────┐
│                          SWC (Software Components)                          │
└───────────────────────────────────┬────────────────────────────────────────┘
                                    RTE
┌───────────────────────────────────┼────────────────────────────────────────┐
│ Services   Com │ SOME/IP │ Sd │ Dcm/Dem │ SecOC │        DoIP               │
│                └────┬────┘    └────┬────┘        └────────┬──────────────────┤
│                    PduR ◄──────────┴──────── routing ─────┘                  │
│                     │                                                        │
│                   SoAd  (Socket Adaptor: PDU ↔ socket)                       │
│                     │                                                        │
│                   TcpIp (IP · ARP · ICMP · UDP · TCP · DHCP)                 │
├─────────────────────┼────────────────────────────────────────── ECU Abstr. ┤
│                    EthIf (Ethernet Interface / switch mgmt)                  │
├─────────────────────┼───────────────────────────────────────────── MCAL ───┤
│              Eth (Driver) ── EthTrcv (PHY driver)                            │
└─────────────────────┼────────────────────────────────────────────────────── ┘
                   PHY / WIRE
```

**Compare to CAN:** `SWC→RTE→Com→PduR→CanIf→Can→CanTrcv→BUS` (reused: RTE, Com, PduR, Dcm/Dem).

---

## 9.5 Glossary (Important Terminology) 🟢

| Term | Meaning |
|---|---|
| **Ethernet** | IEEE 802.3 wired LAN standard (framing + PHY) |
| **MAC address** | 48-bit L2 hardware address |
| **IP address** | L3 logical address (IPv4 32-bit / IPv6 128-bit) |
| **Port** | 16-bit L4 application identifier |
| **PHY** | Physical-layer transceiver chip |
| **MAC (controller)** | L2 media access controller (framing/addressing) |
| **Switch** | L2 device forwarding by MAC |
| **Router** | L3 device forwarding by IP |
| **Frame / Packet / Segment** | L2 / L3 / L4 data units |
| **EtherType** | Field naming L2 payload type |
| **FCS** | Frame Check Sequence (CRC-32) |
| **MTU** | Max payload = 1500 bytes |
| **VLAN (802.1Q)** | Logical network segmentation; carries PCP priority |
| **PCP / VID** | Priority Code Point / VLAN ID (in VLAN tag) |
| **ARP / NDP** | IP→MAC resolution (IPv4 / IPv6) |
| **DHCP** | Auto IP configuration (DORA) |
| **DNS** | Name → IP resolution |
| **NAT** | Address translation at a router |
| **TCP / UDP** | Reliable-connection / fast-connectionless L4 |
| **Socket** | IP + port + protocol endpoint |
| **ICMP** | IP diagnostics (ping) |
| **SOME/IP** | Service-oriented middleware over IP |
| **SOME/IP-SD** | Service Discovery (Offer/Find/Subscribe) |
| **DoIP** | UDS diagnostics over IP (ISO 13400, port 13400) |
| **UDS** | Unified Diagnostic Services (ISO 14229) |
| **ISO-TP** | CAN transport for UDS (not used in DoIP) |
| **100BASE-T1** | 100 Mbit/s single-pair automotive Ethernet |
| **10BASE-T1S** | 10 Mbit/s multidrop (PLCA) automotive Ethernet |
| **PLCA** | Physical Layer Collision Avoidance (10BASE-T1S) |
| **TSN** | Time-Sensitive Networking (deterministic Ethernet) |
| **gPTP / PTP** | Generalized/Precision Time Protocol (802.1AS / 1588) |
| **AVB** | Audio Video Bridging |
| **TAS / CBS** | Time-Aware / Credit-Based Shaper |
| **MACsec / SecOC** | L2 encryption / AUTOSAR message authentication |
| **SoAd / TcpIp / EthIf / Eth** | AUTOSAR Ethernet BSW modules |
| **PduR / Com / RTE** | AUTOSAR modules reused from CAN stack |
| **Zonal architecture** | Zone controllers + Ethernet backbone |
| **Gateway ECU** | Bridges CAN/LIN ↔ Ethernet; routes/translates |

---

## 9.6 Interview Quick-Revision Questions (Rapid Fire) 🟢

1. Three addressing levels? → MAC, IP, port.
2. Ethernet frame fields? → Preamble, SFD, DstMAC, SrcMAC, EtherType, Payload, FCS.
3. MTU? → 1500. FCS? → CRC-32.
4. TCP handshake? → SYN, SYN-ACK, ACK.
5. UDP or TCP for SOME/IP events? → UDP (multicast).
6. DoIP port? → 13400 (TCP/UDP).
7. Switch vs router? → MAC/L2 vs IP/L3.
8. ARP does what? → IP→MAC on local subnet.
9. VLAN carries what priority field? → PCP.
10. T1 means? → single twisted pair, full duplex.
11. 10BASE-T1S special feature? → multidrop (PLCA), CAN-like.
12. TSN gives Ethernet? → determinism (bounded latency).
13. gPTP provides? → sub-µs time sync (sensor fusion, TSN).
14. SOME/IP patterns? → methods, events, fields.
15. DoIP replaces? → UDS-on-CAN (ISO-TP).
16. AUTOSAR Ethernet lower modules? → SoAd, TcpIp, EthIf, Eth.
17. Reused AUTOSAR modules? → RTE, Com, PduR, Dcm.
18. CanTp equivalent on Ethernet? → DoIP.
19. Which is like CAN, TCP or UDP? → UDP.
20. Ethernet L2 retransmits? → No; TCP at L4 does.
21. Config artifact vs DBC? → ARXML/Fibex.
22. Ethernet trace tool? → Wireshark.
23. Multicast MAC prefix (IPv4)? → 01:00:5E.
24. Broadcast MAC? → FF:FF:FF:FF:FF:FF.
25. Debug order? → link→IP→VLAN→route→port→capture.

---

## 9.7 30-Day Learning Roadmap 🟢

**Week 1 — Fundamentals (Ch 1–2)**
- Day 1: Ethernet basics, why Ethernet, Ethernet vs Internet vs TCP/IP (1.1–1.3).
- Day 2: MAC/IP/port, PHY/MAC/switch/router (1.4–1.5).
- Day 3: OSI & TCP/IP models, Ethernet frame (1.6–1.7).
- Day 4: Unicast/multicast/broadcast, duplex, bandwidth/latency (1.8–1.11).
- Day 5: CAN→Ethernet mapping part 1 (2.1–2.6).
- Day 6: CAN→Ethernet mapping part 2 + master table (2.7–2.13).
- Day 7: Review + rapid-fire Q's (9.6), redraw the frame from memory.

**Week 2 — Networking + Stack (Ch 3–4)**
- Day 8: IPv4/IPv6, subnet mask, gateway (3.1–3.3).
- Day 9: DHCP, ARP, DNS (3.4–3.6).
- Day 10: TCP vs UDP, ports, sockets, client/server (3.7–3.10).
- Day 11: Routing/switching, VLAN, NAT, troubleshooting (3.11–3.14).
- Day 12: Encapsulation, L1–L3, ARP, IP, ICMP (4.1–4.6).
- Day 13: UDP/TCP, SOME/IP, DoIP (4.7–4.10).
- Day 14: Step-by-step ECU→ECU flow (4.11) + review.

**Week 3 — Automotive + AUTOSAR (Ch 5–6)**
- Day 15: Why Ethernet, architecture, ECU↔switch (5.1–5.3).
- Day 16: T1 PHYs (10/100/1000BASE-T1), PHY details (5.4–5.5).
- Day 17: Switch, gateway ECU, zonal/central compute (5.6–5.8).
- Day 18: DoIP, SOME/IP, VLAN in automotive (5.9–5.11).
- Day 19: TSN, AVB, PTP, security (5.12–5.15).
- Day 20: AUTOSAR stack mapping + modules (6.1–6.6).
- Day 21: Diagnostics/comm paths, gateway, config (6.7–6.9) + review.

**Week 4 — Practical + Interview (Ch 7–9)**
- Day 22: Two-ECU + SOME/IP + DoIP examples (7.1–7.3).
- Day 23: Wireshark, packet analysis, filters (7.4–7.7).
- Day 24: Config concepts + debugging table + exercises (7.8–7.10).
- Day 25: Deep-dive interview Q's A1–A10 (Ch 8 Part A).
- Day 26: Basic + intermediate banks (8 B1–B2).
- Day 27: Advanced/automotive + scenarios (8 B3–B4).
- Day 28: Cheat sheet + CAN-vs-Ethernet table (9.1–9.2).
- Day 29: Diagrams + glossary (9.3–9.5).
- Day 30: Full mock: rapid-fire (9.6) + re-answer 20 random Q's using the 5-part template.

**Ongoing after Day 30:** capture real traffic in Wireshark, decode a SOME/IP + DoIP sample, and practice narrating the full send→switch→receive flow aloud.

---

## 9.8 Final Advice for Your Interviews

- **Lead with your CAN strength** — always bridge the answer back to CAN; it shows depth and eases learning-curve concerns.
- **Draw diagrams** — the frame, the stack, and the architecture. Visual answers stand out.
- **Know the flow cold** — `SWC→RTE→SOME-IP→PduR→SoAd→TcpIp→EthIf→Eth→PHY` and its CAN twin.
- **Be precise on the "new" trio** — SoAd, TcpIp, and the SOME/IP+SD+DoIP layer are where CAN engineers get tested.
- **Have a debugging story** — describe a bottom-up troubleshooting approach with Wireshark.

You now have a complete, chapter-wise Ethernet + Automotive Ethernet learning and interview package. Good luck!
