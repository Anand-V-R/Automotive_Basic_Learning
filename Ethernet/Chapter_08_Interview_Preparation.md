# Chapter 8 — Interview Preparation

> Goal: Interview-ready Q&A for an automotive communication engineer (4–5 yrs) transitioning from CAN to Ethernet. This chapter has two parts:
>
> - **Part A — Deep-dive questions** (the most important topics) in full format: **Question → Expected Answer → Detailed Explanation → Practical Example → Common Mistakes**.
> - **Part B — Question banks**: 50 basic + 50 intermediate + 50 advanced/automotive + 25 scenario-based, each with a concise model answer.

🟢 = very frequently asked.

---

# PART A — Deep-Dive Questions (Full Format)

## A1. What is the difference between MAC address and IP address? 🟢

**Expected Answer:** MAC is a 48-bit hardware (Layer 2) address for delivery **within a local network**; IP is a logical (Layer 3) address for delivery **across networks**. MAC is fixed to the NIC; IP is assigned/configurable.

**Detailed Explanation:** A switch forwards frames using the **destination MAC** within one link/subnet. A router forwards packets using the **destination IP** between networks. IP stays constant end-to-end; the MAC is rewritten at every router hop. **ARP** maps IP→MAC so a frame can actually be delivered locally.

**Practical Example:** Sending to `192.168.0.20`: the host ARPs for that IP, gets MAC `AA:..:20`, and the switch delivers by that MAC. Crossing subnets, it instead sends to the gateway's MAC while keeping the final destination IP.

**Common Mistakes:** Saying MAC changes across the network (it's the IP that stays; MAC is per-hop). Confusing "IP identifies the app" — that's the **port**, not IP.

---

## A2. TCP vs UDP — when do you use each in automotive? 🟢

**Expected Answer:** TCP is connection-oriented, reliable, ordered, with flow/congestion control — used for **DoIP** and large/critical transfers. UDP is connectionless, fast, no guarantees — used for **SOME/IP events, notifications, and Service Discovery**.

**Detailed Explanation:** TCP uses a 3-way handshake (SYN, SYN-ACK, ACK), sequence/ACK numbers, retransmission, and windowing. UDP has just an 8-byte header (ports, length, checksum). Automotive periodic/event traffic favors UDP for low latency and multicast; diagnostics favor TCP for reliability during flashing.

**Practical Example:** A speed event streamed via SOME/IP over UDP multicast to subscribers; an ECU firmware flash via DoIP over TCP so no bytes are lost.

**Common Mistakes:** Claiming UDP has no error checking (it has an optional checksum). Saying TCP is "always better" — its overhead/latency is wrong for high-rate events; that's why UDP dominates SOME/IP events.

---

## A3. Explain the Ethernet II frame structure. 🟢

**Expected Answer:** Preamble (7B), SFD (1B), Destination MAC (6B), Source MAC (6B), EtherType (2B), Payload (46–1500B), FCS/CRC-32 (4B). VLAN adds a 4-byte 802.1Q tag after Source MAC.

**Detailed Explanation:** Preamble+SFD synchronize the receiver (Layer 1). MAC addresses handle L2 delivery. **EtherType** identifies the payload (0x0800 IPv4, 0x0806 ARP, 0x86DD IPv6, 0x8100 VLAN). Payload max = **MTU 1500**; min frame 64 bytes (padded). FCS is CRC-32 for error detection; bad frames are dropped.

**Practical Example:** In Wireshark, expand "Ethernet II" to see dst/src MAC and the Type field pointing to IPv4.

**Common Mistakes:** Forgetting FCS is CRC-**32** (CAN is 15-bit). Saying Ethernet retransmits bad frames — it doesn't; TCP does that at L4.

---

## A4. How does an Ethernet switch work vs CAN arbitration? 🟢🧩

**Expected Answer:** A switch learns MAC→port mappings from source MACs and forwards frames only to the destination port; each device has a dedicated full-duplex link, so there is **no collision and no arbitration**. CAN uses **bitwise priority arbitration** on a shared bus.

**Detailed Explanation:** Switch operations: learn (source MAC), forward (known dst MAC → one port), flood (unknown dst / broadcast → all ports), filter, and buffer/queue. CAN, being a shared bus, resolves simultaneous transmissions by ID priority (dominant bits win) with the loser backing off.

**Practical Example:** ECU-A→ECU-B traffic never reaches ECU-C on a switch, whereas on CAN every node sees every frame and filters by ID.

**Common Mistakes:** Saying Ethernet uses CSMA/CD today (obsolete with full-duplex switching). Confusing switch (L2/MAC) with router (L3/IP).

---

## A5. Why is the automotive industry moving from CAN to Ethernet? 🟢🚗

**Expected Answer:** Bandwidth (cameras/radar/lidar/ADAS need 100 Mbit/s–Gbit/s), reduced wiring cost/weight (single-pair T1 + switched backbone), OTA/software-defined vehicles, service-oriented communication (SOME/IP), scalability, and — via TSN — determinism for control traffic.

**Detailed Explanation:** CAN maxes near 1 Mbit/s (8 in CAN FD data phase) and 8/64-byte payloads — it can't carry high-bandwidth sensor data or fast OTA. Ethernet's switched full-duplex architecture plus VLAN/TSN enables consolidation into zonal/central-compute architectures.

**Practical Example:** A 360° surround-view system streams multiple camera feeds over 100/1000BASE-T1; OTA updates push gigabytes via DoIP/TCP.

**Common Mistakes:** Saying Ethernet fully replaces CAN — CAN survives at the low-cost sensor/actuator edge (often 10BASE-T1S) inside zones.

---

## A6. What is SOME/IP and how does it differ from CAN/DBC signaling? 🟢🚗

**Expected Answer:** SOME/IP is automotive service-oriented middleware over IP supporting **methods (request/response), events (notifications), and fields**, with **Service Discovery (SD)** to find services at runtime. Unlike CAN's static, always-broadcast DBC signals, SOME/IP communicates **on demand** to **subscribers**.

**Detailed Explanation:** A server ECU offers a service (Service ID); clients discover it via SOME/IP-SD (OfferService/FindService), subscribe to eventgroups, and receive events (often UDP multicast) or call methods. This decouples clients from providers and reduces unnecessary traffic.

**Practical Example:** Instead of a periodic CAN "EngineSpeed" message everyone hears, an EngineData service offers a Speed event only to subscribers.

**Common Mistakes:** Saying SOME/IP is only UDP (methods/large data can use TCP). Confusing SOME/IP (data) with SD (discovery).

---

## A7. What is DoIP and how does it compare to UDS-on-CAN? 🟢🚗

**Expected Answer:** DoIP (ISO 13400) carries **UDS diagnostics over TCP/IP** (port 13400): UDP for vehicle discovery/announcement, TCP for the diagnostic session with routing activation and logical addressing. It replaces UDS-on-CAN (ISO-TP/DoCAN).

**Detailed Explanation:** Same UDS services (0x10, 0x22, 0x27, 0x2E, 0x31, 0x34/36/37...), but on CAN they're segmented by **ISO-TP** (FF/CF/FC), while DoIP relies on **TCP** for segmentation and reliability — far faster, essential for flashing/OTA.

**Practical Example:** Reading VIN: tester sends `22 F1 90` inside a DoIP diagnostic message over TCP; ECU responds `62 F1 90 <VIN>`.

**Common Mistakes:** Saying DoIP uses ISO-TP (it doesn't). Forgetting routing activation must succeed before diagnostics.

---

## A8. Map the AUTOSAR Ethernet stack to the AUTOSAR CAN stack. 🟢🧩

**Expected Answer:**
```
CAN:  SWC→RTE→Com→PduR→CanIf→Can→CanTrcv→BUS
Eth:  SWC→RTE→Com/SOME-IP→PduR→SoAd→TcpIp→EthIf→Eth→EthTrcv/PHY→WIRE
```
Reused: RTE, Com, PduR, Dcm/Dem. New: SoAd, TcpIp, EthIf, Eth, EthTrcv, SOME/IP, Sd, DoIP.

**Detailed Explanation:** Can↔Eth (driver), CanTrcv↔EthTrcv, CanIf↔EthIf (HW abstraction) + **SoAd** (PDU↔socket), CanTp↔**DoIP** (diag transport), and **TcpIp** is entirely new (IP/ARP/UDP/TCP/DHCP).

**Practical Example:** A gateway ECU uses **PduR** to route PDUs between CanIf (CAN) and SoAd (Ethernet), translating CAN signals to SOME/IP.

**Common Mistakes:** Forgetting TcpIp has no CAN equivalent; mixing up EthIf (HW abstraction) with SoAd (socket routing).

---

## A9. What is TSN and why is it needed? 🟢🚗

**Expected Answer:** TSN (Time-Sensitive Networking) is a set of IEEE 802.1 standards adding **deterministic, bounded-latency** delivery to standard Ethernet: gPTP time sync (802.1AS), Time-Aware Shaper (802.1Qbv), Frame Preemption (802.1Qbu/802.3br), Credit-Based Shaper (802.1Qav). It's needed because plain Ethernet is fast but not deterministic — bursts can delay critical frames.

**Detailed Explanation:** With a shared time base, switches open time-gated windows so high-priority (control/safety) traffic transmits on schedule; preemption lets a critical frame interrupt a low-priority one. This gives Ethernet the real-time behavior CAN had natively.

**Practical Example:** Brake-control frames get guaranteed windows while a large infotainment transfer runs concurrently.

**Common Mistakes:** Saying TSN is one standard (it's a family). Confusing AVB (A/V subset) with full TSN.

---

## A10. What is gPTP/PTP and why does automotive need precise time sync? 🟢🚗

**Expected Answer:** PTP (IEEE 1588) / gPTP (802.1AS) synchronizes ECU clocks to sub-microsecond accuracy via a grandmaster clock and delay measurement. Needed for **sensor fusion timestamping**, **TSN scheduling**, and **coordinated logging**.

**Detailed Explanation:** A grandmaster is elected (BMCA); switches act as boundary/transparent clocks correcting for their delay; each link measures propagation delay so nodes align to the grandmaster.

**Practical Example:** Camera, radar, and lidar frames share a common timestamp so ADAS fusion aligns them correctly.

**Common Mistakes:** Confusing PTP with NTP (PTP is far more precise). Forgetting TSN scheduling depends on gPTP.

---

# PART B — Question Banks

Format: **Q → concise model answer.** Study these until the answers are automatic.

## B1. 50 Basic Ethernet & Networking Questions

1. **What is Ethernet?** IEEE 802.3 standard for wired LAN framing + physical signaling.
2. **What is a LAN?** A local network of devices (home, office, one car).
3. **LAN vs WAN?** LAN = local/small; WAN = wide/large (Internet is the biggest WAN).
4. **What is a node?** Any device on a network.
5. **What is a host?** A node that runs apps and has an IP.
6. **What is a MAC address?** 48-bit hardware L2 address, e.g. AA:BB:CC:DD:EE:FF.
7. **What is an IP address?** Logical L3 address for cross-network delivery.
8. **What is a port number?** 16-bit L4 identifier for an application/service.
9. **IPv4 size?** 32 bits, four octets.
10. **IPv6 size?** 128 bits.
11. **What is a switch?** L2 device forwarding frames by MAC.
12. **What is a router?** L3 device forwarding packets by IP between networks.
13. **Switch vs router?** Switch = MAC/within LAN; router = IP/between networks.
14. **What is a PHY?** Physical-layer chip converting bits ↔ line signals.
15. **What is a MAC (controller)?** L2 logic building/parsing frames + addressing.
16. **PHY vs MAC?** PHY = analog/wire; MAC = digital/framing.
17. **What is full duplex?** Simultaneous send + receive.
18. **What is half duplex?** One direction at a time.
19. **What is bandwidth?** Data capacity per second.
20. **What is latency?** End-to-end delay.
21. **What is jitter?** Variation in latency.
22. **What is throughput?** Actual useful data delivered/sec.
23. **What is the MTU?** Max payload = 1500 bytes.
24. **Minimum Ethernet frame size?** 64 bytes.
25. **What is EtherType?** 2-byte field naming the payload (0x0800 IPv4).
26. **What is FCS?** Frame Check Sequence, CRC-32 for error detection.
27. **What is a preamble?** 7-byte sync pattern before a frame.
28. **What is SFD?** Start Frame Delimiter (1 byte) marking frame start.
29. **Unicast?** One-to-one delivery.
30. **Multicast?** One-to-group delivery.
31. **Broadcast?** One-to-all (dst MAC FF:FF:FF:FF:FF:FF).
32. **What is ARP?** Resolves IP → MAC on the local subnet.
33. **What is DHCP?** Auto-assigns IP config (DORA).
34. **What is DNS?** Resolves names → IPs.
35. **What is a subnet mask?** Splits IP into network/host parts.
36. **What is a default gateway?** Router used for off-subnet destinations.
37. **What is CIDR /24?** 24 network bits = 255.255.255.0.
38. **What is loopback?** 127.0.0.1 (localhost).
39. **What is a private IP?** 10.x, 172.16–31.x, 192.168.x (non-routable publicly).
40. **What is TCP?** Reliable, connection-oriented L4 protocol.
41. **What is UDP?** Fast, connectionless L4 protocol.
42. **TCP handshake?** SYN, SYN-ACK, ACK.
43. **What is a socket?** Endpoint = IP + port + protocol.
44. **What is ICMP?** Diagnostic/error messaging (ping uses it).
45. **What does ping test?** L3 reachability + round-trip time.
46. **OSI layers count?** 7.
47. **TCP/IP model layers?** 4 (App, Transport, Internet, Link).
48. **Which layer is IP?** Layer 3 (Network).
49. **Which layer is Ethernet?** Layers 1–2 (Physical + Data Link).
50. **Which layer are ports?** Layer 4 (Transport).

## B2. 50 Intermediate Questions (TCP/IP, Frames, Switching, CAN-vs-Ethernet)

1. **How does a switch learn MACs?** From source MAC of incoming frames → MAC table.
2. **What happens on unknown dst MAC?** Flood to all ports except source.
3. **Collision domain in switched Ethernet?** Each link is its own; no collisions in full duplex.
4. **What is CSMA/CD?** Old half-duplex collision handling — obsolete now.
5. **What is a VLAN?** Logical network segmentation on shared switches (802.1Q).
6. **Size of 802.1Q tag?** 4 bytes.
7. **What is PCP?** 3-bit priority in the VLAN tag (QoS).
8. **What is VID?** 12-bit VLAN ID (0–4095).
9. **What is NAT?** Rewrites private↔public IP/port at a router.
10. **What is a routing table?** Maps IP networks → next hops.
11. **What is TTL?** Hop limit in IP header; prevents loops.
12. **IP protocol number for TCP/UDP/ICMP?** 6 / 17 / 1.
13. **What is fragmentation?** Splitting packets larger than MTU.
14. **UDP header size?** 8 bytes.
15. **TCP header size?** 20+ bytes.
16. **What is the 5-tuple?** src IP, src port, dst IP, dst port, protocol.
17. **What is flow control (TCP)?** Window-based sender rate limiting.
18. **What is congestion control?** TCP reduces rate on loss.
19. **TCP teardown?** FIN, ACK, FIN, ACK.
20. **What is a gratuitous ARP?** Self-announcement of IP/MAC.
21. **IPv6 replacement for ARP?** NDP (Neighbor Discovery).
22. **What is link-local IPv4?** 169.254.x.x (auto-IP).
23. **CAN ID vs MAC/IP?** CAN ID = message/priority; MAC = device, IP = host.
24. **CAN frame vs Ethernet frame?** 8/64B + ID vs up to 1500B + MAC/EtherType.
25. **CAN arbitration vs switching?** Priority on shared bus vs forwarding on dedicated links.
26. **CAN controller vs Ethernet MAC?** Both L2 framing brains.
27. **CAN transceiver vs PHY?** Both drive the physical wire.
28. **CAN gateway vs router?** Bus translation vs IP routing.
29. **CAN error handling vs Ethernet?** CAN built-in counters/bus-off; Ethernet FCS-drop + TCP recovery.
30. **Does Ethernet ACK at L2?** No; TCP acks at L4.
31. **Which is like CAN: TCP or UDP?** UDP (fire-and-forget).
32. **What is jumbo frame?** Payload >1500 bytes.
33. **What is a broadcast domain?** Set of devices reached by a broadcast (one VLAN/subnet).
34. **What is IGMP snooping?** Switch limits multicast to interested ports.
35. **Why multicast for SOME/IP events?** Efficient one-to-many to subscribers.
36. **What is auto-negotiation?** IT PHYs agree speed/duplex; automotive T1 uses master/slave config.
37. **What is MDI/MDI-X?** Auto crossover in IT Ethernet.
38. **What is a boundary clock?** PTP switch correcting for its delay.
39. **What decides same subnet?** Matching network part under the mask.
40. **Ping works, app fails — layer?** L4/L7 (port/app), not L3.
41. **What is a MAC table aging?** Old entries expire to adapt to topology changes.
42. **What is port mirroring?** Copy a port's traffic to a monitor port (debug).
43. **What is a broadcast storm?** Flooding loop; prevented by STP.
44. **What is STP?** Spanning Tree Protocol; prevents L2 loops.
45. **What is QoS?** Prioritizing traffic classes (PCP/DSCP).
46. **What is DSCP?** L3 QoS marking in the IP header.
47. **What is a default route?** 0.0.0.0/0 → gateway for unknown destinations.
48. **What is a socket bind?** Associating a socket with a local IP:port.
49. **UDP checksum optional?** Yes in IPv4, mandatory in IPv6.
50. **Why is TCP unsuitable for high-rate events?** Overhead/latency from acks/ordering.

## B3. 50 Advanced / Automotive Questions

1. **What is 100BASE-T1?** 100 Mbit/s, single twisted pair, full duplex.
2. **What is 1000BASE-T1?** 1 Gbit/s over single pair.
3. **What is 10BASE-T1S?** 10 Mbit/s, supports multidrop (PLCA) — CAN-like edge.
4. **What is 10BASE-T1L?** 10 Mbit/s long reach single pair.
5. **What does T1 mean?** Twisted pair, 1 pair.
6. **Why single pair?** Reduce wiring weight/cost.
7. **How is full duplex on one pair achieved?** Echo cancellation.
8. **What is PLCA?** Physical Layer Collision Avoidance for 10BASE-T1S multidrop.
9. **Master/slave PHY role?** One side drives timing; both ends must be opposite.
10. **What modulation does 100BASE-T1 use?** PAM3.
11. **IT vs automotive connector?** RJ45 vs rugged automotive connectors.
12. **What is a zonal architecture?** Zone controllers aggregate local I/O to an Ethernet backbone.
13. **What is central compute?** Few HPCs run most software; zones are smart I/O.
14. **What is a gateway ECU?** Bridges CAN/LIN ↔ Ethernet, routes domains, translates UDS↔DoIP.
15. **What is SOME/IP?** Service-oriented middleware over IP (methods/events/fields).
16. **What is SOME/IP-SD?** Service discovery (Offer/Find/Subscribe).
17. **SOME/IP transport?** UDP (events/SD), TCP (large/methods).
18. **What is an eventgroup?** A set of events a client subscribes to.
19. **What is a field (SOME/IP)?** Value with getter/setter/notifier.
20. **SOME/IP message types?** REQUEST, RESPONSE, NOTIFICATION, ERROR, etc.
21. **What is DoIP?** UDS over IP (ISO 13400, port 13400).
22. **DoIP discovery transport?** UDP (announcement); TCP for session.
23. **What is routing activation?** DoIP authorization step before diagnostics.
24. **What is a logical address (DoIP)?** Identifies target ECU for diagnostics.
25. **Why DoIP has no ISO-TP?** TCP handles segmentation/reliability.
26. **What is TSN?** IEEE 802.1 family for deterministic Ethernet.
27. **802.1AS?** gPTP time synchronization.
28. **802.1Qbv?** Time-Aware Shaper (scheduled gates).
29. **802.1Qbu/802.3br?** Frame preemption.
30. **802.1Qav?** Credit-Based Shaper (AVB bandwidth reservation).
31. **What is AVB?** A/V bridging subset (audio/video streaming, low latency).
32. **What is AVTP?** Transport for audio/video streams (IEEE 1722).
33. **What is gPTP?** Automotive PTP profile (802.1AS).
34. **Why precise time sync?** Sensor fusion, TSN scheduling, logging.
35. **What is BMCA?** Best Master Clock Algorithm (grandmaster election).
36. **What is MACsec?** L2 encryption/authentication (802.1AE).
37. **What is SecOC?** AUTOSAR message authentication (MAC + freshness).
38. **UDS Security Access?** Service 0x27 (seed/key) — used over DoIP too.
39. **AUTOSAR Eth driver module?** Eth.
40. **AUTOSAR Ethernet interface module?** EthIf.
41. **AUTOSAR TCP/IP module?** TcpIp.
42. **AUTOSAR socket adaptor?** SoAd.
43. **AUTOSAR service discovery module?** Sd.
44. **AUTOSAR PDU router role in gateway?** Routes PDUs between CanIf and SoAd.
45. **What replaces CanTp on Ethernet diagnostics?** DoIP.
46. **What config artifact for Ethernet?** ARXML/Fibex (vs DBC/LDF).
47. **How is CAN signal ↔ SOME/IP done?** Signal-to-service mapping (transformer + Com) in gateway.
48. **Why VLANs in cars?** Domain isolation for safety/security + PCP priority.
49. **What is partial networking (Ethernet)?** PHY wake/sleep to save power.
50. **What enables OTA on Ethernet?** High bandwidth + DoIP/TCP for fast, reliable flashing.

## B4. 25 Scenario-Based Questions

1. **Two ECUs can't communicate; link is up.** Check IP/subnet match, VLAN, ARP resolution, then ports/app config; capture in Wireshark.
2. **Ping works but SOME/IP event not received.** SD offer/subscribe missing, multicast/VLAN/eventgroup mismatch — trace `someip`.
3. **DoIP session fails after TCP connect.** Routing activation rejected or wrong logical address — check activation response code.
4. **Intermittent latency spikes on control traffic.** Bulk traffic starving critical frames — apply VLAN PCP / TSN scheduling.
5. **Sensor fusion timestamps drift.** gPTP not synced / grandmaster lost — verify PTP state.
6. **New ECU added, gets no IP.** Static misconfig or DHCP unavailable; check config/lease.
7. **Duplicate IP detected.** Two ECUs share a static IP — gratuitous ARP conflict; fix IP plan.
8. **Frames dropped with FCS errors.** EMC/cable/connector issue — inspect physical link.
9. **Switch floods all ports constantly.** Broadcast storm/loop — check STP / topology.
10. **Camera stream stutters.** Insufficient bandwidth reservation — use CBS/TSN, check MTU/jumbo.
11. **Tester can reach Ethernet ECU but not a CAN ECU.** Gateway DoIP↔CanTp routing broken — check PduR routes.
12. **SOME/IP method call times out.** Server not offering / TCP not established / wrong port — trace request/response.
13. **VLAN traffic invisible to receiver.** Port on wrong VLAN or missing tag — verify PVID/tagging.
14. **High retransmissions on TCP flash.** Packet loss/link quality — check `tcp.analysis.retransmission`.
15. **ECU reachable then drops off.** Link flap / master-slave mismatch / power mode — check PHY.
16. **Two grandmasters appear.** BMCA misconfig / domain split — enforce single gPTP domain.
17. **Multicast events reach wrong ECUs.** IGMP snooping off / static multicast wrong — fix switch multicast.
18. **Diagnostics slow over DoIP.** Nagle/segmentation/window issues or app delay — analyze TCP.
19. **Security: infotainment can access chassis.** VLAN/firewall segmentation missing — enforce isolation.
20. **Gateway drops some CAN→Eth signals.** PduR/Com mapping incomplete — verify signal-to-service config.
21. **OTA update fails midway.** TCP reset / power / memory — check DoIP transfer (0x34/36/37) and link stability.
22. **Wireshark shows SYN but no SYN-ACK.** Server not listening / firewall — check SoAd socket & port.
23. **PHY won't link at 100BASE-T1.** Both set master or both slave — set opposite roles.
24. **Latency ok on bench, bad in car.** Real traffic load — validate TSN schedule under full load.
25. **ARP table shows wrong MAC for an IP.** ARP spoofing or duplicate/stale entry — clear cache, investigate security.

---

## 8.x How to Structure Any Interview Answer (Template)

For each question, aim to hit:
1. **One-line definition** (simple language).
2. **Technical detail** (the precise mechanism).
3. **CAN comparison** (leverage your background — interviewers love this).
4. **Practical/automotive example**.
5. **A common pitfall** you know to avoid.

This structure signals both fundamentals *and* real experience — exactly what a 4–5 year engineer transitioning from CAN should project.

**Next:** [Chapter 9 — Final Revision](Chapter_09_Final_Revision.md) — cheat sheets, comparison tables, diagrams, glossary, and a 30-day roadmap.
