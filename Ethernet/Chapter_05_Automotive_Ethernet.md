# Chapter 5 — Automotive Ethernet

> Goal: The heart of your transition. Why the industry moved to Ethernet, the physical layers (10/100/1000BASE-T1), switches and gateways, zonal & central-compute architectures, DoIP, SOME/IP, VLAN, and the time-critical trio **TSN / AVB / PTP**, plus security. 🚗

---

## 5.1 Why the Automotive Industry Is Moving from CAN to Ethernet 🟢

The drivers (great interview answer):

1. **Bandwidth explosion** — cameras, radar, lidar, ADAS, HD maps, infotainment need **100 Mbit/s–Gbit/s**. CAN maxes at ~1 Mbit/s (8 in CAN FD data phase). CAN physically cannot carry a camera stream.
2. **Wiring cost & weight** — dozens of separate CAN/LIN/FlexRay buses = heavy, complex harness. Ethernet + a switched backbone consolidates wiring (**single twisted pair** with T1).
3. **Software-defined vehicles & OTA** — large **over-the-air updates** need high throughput; Ethernet delivers.
4. **Service-oriented architecture** — **SOME/IP** enables flexible, reusable, on-demand communication vs CAN's rigid static signals.
5. **Scalability & standard IT ecosystem** — reuse mature TCP/IP tooling, security, and switching.
6. **Determinism now possible** — **TSN** brings CAN-like real-time guarantees to Ethernet, so it can even carry control traffic.

🧩 **The trade-off:** CAN is simpler, cheaper per node, and deterministic by nature. Ethernet is more complex (IP stack, config, switches) but vastly more capable. Cars use **both** — Ethernet backbone + CAN/LIN for simple edge sensors/actuators, bridged by gateways.

---

## 5.2 Automotive Ethernet Architecture Basics

```
        ┌──────────────────── Central / Backbone ────────────────────┐
        │                                                             │
 [ADAS ECU]──┐                                              ┌──[Infotainment]
             │                                              │
 [Camera]────┼───[ Automotive Ethernet SWITCH ]────────────┼──[Telematics/Cloud]
             │            (VLAN + TSN)                      │
 [Gateway ECU]┘                                             └──[Body/Chassis ECUs]
        │
        ├── CAN bus (legacy) ── [CAN ECUs]
        ├── LIN ── [LIN slaves]
        └── FlexRay (older) ── [FlexRay ECUs]
```

Key elements:
- **Switch** — forwards frames, applies VLANs and TSN priorities.
- **Gateway ECU** — bridges Ethernet ↔ CAN/LIN/FlexRay and routes between domains/zones; translates CAN signals ↔ SOME/IP, and UDS ↔ DoIP.
- **Host ECUs** — run IP stack + SOME/IP/DoIP.

---

## 5.3 ECU → Ethernet Switch → ECU Communication

Because Ethernet is **point-to-point full-duplex**, ECUs don't share a wire — each connects to a **switch port**. The switch:
1. Learns MAC↔port mappings.
2. Forwards each frame only to the destination port (unicast) or group ports (multicast).
3. Prioritizes/schedules frames (VLAN PCP + TSN) so critical traffic isn't delayed by bulk traffic.

🧩 This is the fundamental difference from CAN's shared bus: **no arbitration, no everyone-hears-everything** — the switch is the traffic cop.

---

## 5.4 Automotive Ethernet Physical Layers — the "T1" Family 🟢

**T1 = single twisted pair, full-duplex** (echo cancellation lets both directions share one pair). This is the automotive innovation: minimal wiring weight/cost, EMC-robust.

| Standard | Speed | Cable | Typical use |
|---|---|---|---|
| **10BASE-T1S** | 10 Mbit/s | single pair, **can be multidrop** (shared, short) | Edge/zonal sensors & actuators, low-cost — a CAN-like replacement |
| **10BASE-T1L** | 10 Mbit/s | single pair, **long reach** | Long-distance (more industrial) |
| **100BASE-T1** | 100 Mbit/s | single pair (STP/UTP) | The workhorse: ADAS, cameras, backbone links |
| **1000BASE-T1** | 1 Gbit/s | single pair | High-bandwidth: high-res cameras, central compute links |
| **2.5/5/10GBASE-T1** | up to 10 Gbit/s | single pair | Next-gen central compute / high-end sensors |

**Naming recap:** `100BASE-T1` = 100 Mbit/s, baseband, **T**wisted pair, **1** pair.

**IT vs Automotive PHY differences (must-know):**
| | IT Ethernet (e.g. 100BASE-TX) | Automotive (100BASE-T1) |
|---|---|---|
| Pairs | 2–4 twisted pairs | **1** twisted pair |
| Connector | RJ45 | rugged automotive connectors |
| Duplex on one pair | No (separate TX/RX pairs) | **Yes** (echo cancellation) |
| Link setup | Auto-negotiation, both equal | **Master/Slave** roles configured |
| Environment | office | automotive EMC, temperature, vibration |

### 10BASE-T1S special note
**10BASE-T1S** supports a **multidrop** topology (multiple nodes on one segment) using **PLCA (Physical Layer Collision Avoidance)** — a token-like scheme giving each node a turn.
🧩 This is the **closest thing to a CAN bus** in the Ethernet world: shared medium, deterministic access, low cost — ideal for replacing CAN at the sensor/actuator edge in zonal designs.

---

## 5.5 Automotive Ethernet PHY (Details)

- Converts MAC data ↔ single-pair line signaling (**PAM3** for 100BASE-T1).
- Handles **master/slave** clock roles (one side drives timing).
- Provides **link status**, wake/sleep (partial networking for power saving), and diagnostics (cable fault detection).
- Connects to the MAC via **MII/RMII/RGMII/SGMII**.

🧩 Same role as a **CAN transceiver** but far more sophisticated (link negotiation, echo cancellation, higher speeds).

---

## 5.6 Ethernet Switch (Automotive) 🟢

An automotive switch adds features beyond a plain IT switch:
- **VLAN (802.1Q)** — isolate domains, carry **PCP priority**.
- **TSN** — time-aware scheduling, frame preemption, guaranteed latency.
- **Rate limiting / policing** — protect against traffic floods (also security).
- **Port mirroring** — for debugging (mirror a port to a diagnostics port for Wireshark).
- **Managed configuration** — static MAC entries, per-port VLAN/QoS.

Switches may be **standalone** or **integrated into an ECU/SoC** (common in zonal controllers).

---

## 5.7 Gateway ECU 🟢

The **Gateway ECU** is the linchpin for a CAN-to-Ethernet engineer:
- **Protocol translation:** CAN/LIN/FlexRay signals ↔ **SOME/IP** service data.
- **Diagnostic routing:** UDS-on-CAN ↔ **DoIP** (routes tester requests to the right ECU/bus).
- **Routing between domains/zones** at L2/L3.
- **Security enforcement** (firewall, IDS, gating traffic between domains).

🧩 It's the evolution of the **CAN gateway** you already know — now bridging legacy buses to the Ethernet backbone and translating signal-based ↔ service-based communication.

---

## 5.8 Zonal Architecture vs Central Compute 🟢🚗

### Traditional (domain/distributed) architecture
Many function-specific ECUs, each on domain buses (powertrain CAN, body CAN, chassis CAN...), bridged by a central gateway. Lots of wiring.

### Zonal architecture
The car is divided by **physical zone** (front-left, front-right, rear...). Each zone has a **Zonal Controller** that:
- Connects nearby sensors/actuators via short CAN/LIN/10BASE-T1S links.
- Aggregates and forwards data over the **Ethernet backbone** to central computers.
- Drastically **reduces harness length/weight** (wires run to the nearest zone, not across the car).

```
        [ Central Compute (HPC) ]
              ║ Gbit Ethernet backbone ║
   ┌──────────╬───────────────╬──────────┐
[Zone FL]  [Zone FR]      [Zone RL]   [Zone RR]
   │           │              │           │
 local     local          local       local
 CAN/LIN/  sensors        actuators   sensors
 T1S
```

### Central compute architecture
A few powerful **High-Performance Computers (HPCs)** run most software (ADAS, infotainment, vehicle control) as software components; zonal controllers act as smart I/O. Enables the **software-defined vehicle** and OTA updates.

🧩 This is the biggest architectural shift: from **many small ECUs + many CAN buses** to **few powerful computers + Ethernet backbone + zonal I/O**. CAN doesn't disappear — it survives as cheap edge connectivity inside zones.

---

## 5.9 Diagnostics over Ethernet — DoIP 🟢

Covered in Ch 4.9. Automotive-focused recap:
- **DoIP (ISO 13400)** carries **UDS** over TCP/IP (port 13400).
- Replaces **UDS-on-CAN (ISO-TP/DoCAN)** — much faster (flashing an ECU over CAN takes long; over DoIP it's dramatically faster → key for OTA).
- **Routing activation** authorizes the tester; **logical addresses** identify target ECUs.
- The **DoIP gateway** routes diagnostics to internal ECUs (over Ethernet or onto CAN).

🧩 Same **UDS services** you know (0x10, 0x22, 0x27, 0x2E, 0x31, 0x34/36/37...), new transport. No ISO-TP because TCP handles segmentation.

---

## 5.10 SOME/IP & Service-Oriented Communication 🟢

Covered in Ch 4.8. Automotive emphasis:
- **Service-oriented** = ECUs offer/consume *services* (methods, events, fields) instead of blindly broadcasting signals.
- **SOME/IP-SD** discovers services at runtime → flexible, reusable ECUs, easier variant handling.
- Uses **multicast** for events (efficient one-to-many, like CAN's many listeners but on-demand).
- Enables **decoupling**: a client doesn't care *which* ECU provides a service, only that it's offered.

🧩 **Mindset shift from DBC:** CAN/DBC = "these signals are always on the bus at this rate." SOME/IP = "this service is available; subscribe if you need it, call methods when needed." More dynamic, more scalable.

---

## 5.11 Ethernet VLAN in Automotive 🟢

(See Ch 3.12.) Automotive uses **VLANs** to:
- **Segment domains** (ADAS / infotainment / diagnostics / chassis) on one physical backbone.
- **Isolate** for safety & security (infotainment can't touch chassis traffic).
- Carry **PCP priority** that feeds TSN scheduling.

---

## 5.12 TSN — Time-Sensitive Networking 🟢🚗

**The problem:** plain Ethernet is fast but **not deterministic** — a big infotainment burst could delay a brake-control frame. CAN never had this problem (priority arbitration). **TSN** fixes it.

TSN is a **set of IEEE 802.1 standards** adding real-time guarantees to standard Ethernet:

| TSN feature | Standard | What it does |
|---|---|---|
| **Time synchronization** | 802.1AS (gPTP) | Common clock across all nodes (basis for scheduling) |
| **Time-Aware Shaper (TAS)** | 802.1Qbv | Time-gated queues → scheduled transmission windows for critical traffic |
| **Frame Preemption** | 802.1Qbu / 802.3br | High-priority frame interrupts a low-priority one mid-transmission |
| **Credit-Based Shaper (CBS)** | 802.1Qav | Smooths/reserves bandwidth for A/V streams (AVB) |
| **Stream Reservation** | 802.1Qat/Qcc | Reserve bandwidth end-to-end for a stream |

**Result:** guaranteed **bounded latency and low jitter** — Ethernet can now carry **control and safety-critical** traffic, not just data.

🧩 TSN gives Ethernet the **determinism CAN had natively**. This is *the* enabler for using Ethernet as a vehicle backbone for time-critical functions.

---

## 5.13 AVB — Audio Video Bridging 🔵🚗

**AVB** is the predecessor/subset of TSN, focused on **streaming audio/video** with bounded latency:
- **802.1AS** — time sync (gPTP).
- **802.1Qav** — Credit-Based Shaper (reserves bandwidth, prevents A/V starvation).
- **802.1Qat** — Stream Reservation Protocol.
- **AVTP (1722)** — the transport for audio/video streams.

🚗 Used for **infotainment audio, camera/microphone streams, surround-view**. TSN is the broader evolution that also covers control traffic.

---

## 5.14 PTP / Time Synchronization 🟢

**PTP (Precision Time Protocol, IEEE 1588)** — and its automotive profile **gPTP (802.1AS)** — synchronize clocks across ECUs to **sub-microsecond** accuracy.

**Why it matters:**
- **Sensor fusion** — timestamp camera/radar/lidar data on a common time base.
- **TSN scheduling** — time-aware shapers need a shared clock.
- **Data logging & replay** — correlate events across ECUs.

**How it works (simplified):**
```
Grandmaster clock (best clock) ── distributes time ──► all nodes
Each link measures propagation delay (delay request/response)
Nodes correct their local clock to match the grandmaster.
```
- One **Grandmaster** is elected (BMCA — Best Master Clock Algorithm).
- **Boundary/transparent clocks** in switches account for switch delay.

🧩 CAN has no built-in time sync (AUTOSAR added a **Time Sync** module over CAN, but Ethernet/gPTP is far more precise). Precise time is *essential* for ADAS sensor fusion.

---

## 5.15 Automotive Ethernet Security Basics 🟢🚗

More capability = bigger attack surface. Key concepts:

| Concept | Purpose |
|---|---|
| **MACsec (802.1AE)** | Layer-2 encryption/authentication between switches/ECUs |
| **IPsec / TLS** | Secure IP/transport (encryption, authentication) for sensitive/external comm |
| **Firewall (Gateway ECU)** | Filter/allow traffic between domains/VLANs |
| **IDS/IPS** | Intrusion Detection/Prevention — detect anomalies |
| **SecOC (Secure Onboard Communication)** | AUTOSAR message authentication (MAC/freshness) — used on CAN *and* Ethernet PDUs |
| **VLAN segmentation** | Isolate critical domains from exposed ones (infotainment/telematics) |
| **Secure boot / secure diagnostics** | Authenticated flashing, UDS Security Access (0x27) over DoIP |

🧩 On CAN you likely used **SecOC** for authenticated messages and **UDS Security Access (0x27)** for diagnostics. Both concepts carry over to Ethernet, plus new network-level protections (MACsec, TLS, firewalls) because Ethernet is more exposed (especially telematics/OTA).

---

## 5.16 Chapter Summary

- Move to Ethernet is driven by **bandwidth, wiring cost, OTA/SDV, service-oriented comm, and (via TSN) determinism**.
- **T1 = single-pair full-duplex**: 10BASE-T1S (multidrop, CAN-like edge), 100BASE-T1 (workhorse), 1000BASE-T1 (high bandwidth).
- Automotive PHYs use **master/slave** roles and single pair with echo cancellation (vs IT's auto-neg + multi-pair).
- **Switch** (VLAN+TSN) replaces the shared bus; **Gateway ECU** bridges CAN/LIN ↔ Ethernet and UDS ↔ DoIP.
- **Zonal + central-compute** architecture = few HPCs + Ethernet backbone + zonal I/O → the software-defined vehicle.
- **DoIP** = UDS over IP; **SOME/IP** = service-oriented comm (methods/events/fields + SD).
- **TSN** brings determinism (time-aware shaping, preemption); **AVB** streams A/V; **gPTP/PTP** synchronizes clocks (essential for sensor fusion + TSN).
- Security: **MACsec, TLS/IPsec, firewalls, IDS, SecOC, secure diagnostics**.

**Next:** [Chapter 6 — AUTOSAR Ethernet Stack](Chapter_06_AUTOSAR_Ethernet.md) — module by module, compared to the AUTOSAR CAN stack you know.
