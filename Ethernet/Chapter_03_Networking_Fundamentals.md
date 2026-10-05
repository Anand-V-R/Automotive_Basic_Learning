# Chapter 3 — Networking Fundamentals

> Goal: Learn the core TCP/IP networking concepts every Ethernet engineer needs: IP addressing, subnets, gateways, DHCP, ARP, DNS, TCP vs UDP, ports, sockets, client/server, routing/switching, VLAN, NAT, and basic troubleshooting. 🔵

These are **standard/IT concepts** that also apply directly to Automotive Ethernet. Where relevant, 🚗 notes show the automotive angle and 🧩 notes show CAN comparisons.

---

## 3.1 IPv4 and IPv6 Basics 🟢

### IPv4
- **32-bit** address, written as 4 decimal **octets**: `192.168.1.10` (each 0–255).
- Total ~4.3 billion addresses (now exhausted globally → NAT & IPv6).
- Split into **network part** and **host part** (defined by the subnet mask).

**Address classes / private ranges (must-know):**

| Range | Use |
|---|---|
| `10.0.0.0/8` | Private |
| `172.16.0.0/12` | Private |
| `192.168.0.0/16` | Private |
| `127.0.0.0/8` | Loopback (`127.0.0.1` = localhost) |
| `169.254.0.0/16` | Link-local (auto-IP when DHCP fails) |
| `224.0.0.0/4` | Multicast |

🚗 Automotive ECUs typically use **static private IPs** (e.g. `192.168.x.x` or `169.254.x.x` link-local) configured via ARXML — not public addresses.

### IPv6
- **128-bit** address, written in hex groups: `2001:0db8:0000:0000:0000:0000:0000:0001`.
- Shortened: `2001:db8::1` (`::` = one run of zeros).
- Practically unlimited addresses; no NAT needed.
- **Link-local** addresses start with `fe80::` (used a lot for local automotive comm).
- Uses **NDP** (Neighbor Discovery) instead of ARP.

🧩 CAN has no address at all — going to IP means every ECU now needs an assigned/configured address.

---

## 3.2 Subnet Mask 🟢

The subnet mask splits an IP into **network** and **host** parts. Devices on the **same subnet** can talk directly (Layer 2); different subnets need a **router**.

**Example:** IP `192.168.1.10`, mask `255.255.255.0` (`/24`)
```
IP     : 192.168.1.10   = 11000000.10101000.00000001.00001010
Mask   : 255.255.255.0  = 11111111.11111111.11111111.00000000
                           └────── network ──────┘└─ host ─┘
Network: 192.168.1.0
Host   : .10
Range  : 192.168.1.1 – 192.168.1.254  (usable)
Broadcast: 192.168.1.255
```

**CIDR notation:** `/24` = 24 network bits = `255.255.255.0`.

| CIDR | Mask | Usable hosts |
|---|---|---|
| /24 | 255.255.255.0 | 254 |
| /25 | 255.255.255.128 | 126 |
| /30 | 255.255.255.252 | 2 (point-to-point) |

**Quick test:** Are `192.168.1.10/24` and `192.168.2.20/24` on the same subnet? → Network parts `192.168.1` vs `192.168.2` differ → **No**, they need a router.

---

## 3.3 Default Gateway 🟢

The **default gateway** is the router a host sends packets to when the destination is **not on its own subnet**.

```
Host wants to send to 8.8.8.8
  Is 8.8.8.8 on my subnet? ── No ──► send to Default Gateway (e.g. 192.168.1.1)
                            ── Yes ─► send directly (ARP for its MAC, deliver)
```

🚗 In cars, a **Gateway ECU** often acts as the default gateway/router between zones or between the Ethernet backbone and legacy CAN/LIN domains.

---

## 3.4 DHCP (Dynamic Host Configuration Protocol) 🔵

Automatically hands out IP config (IP, subnet mask, gateway, DNS) to devices.

**DORA handshake:**
```
Client                          DHCP Server
  │── Discover (broadcast) ────────►│   "Anyone give me an IP?"
  │◄──────── Offer ─────────────────│   "Here, take 192.168.1.50"
  │── Request (broadcast) ─────────►│   "I'll take that one"
  │◄──────── Ack ───────────────────│   "It's yours, lease X sec"
```

🚗 Most automotive ECUs use **static IPs** (deterministic, no boot-time delay), but **DHCP is common on the diagnostic/OBD interface** and for tester tools connecting to the vehicle. **DoIP** defines how a tester gets/uses an IP (DHCP or AutoIP/link-local).

---

## 3.5 ARP (Address Resolution Protocol) 🟢

ARP maps a known **IP address → MAC address** on the local subnet. You can't deliver an Ethernet frame without the destination MAC.

```
Host A knows it wants IP 192.168.1.20 but not its MAC.
A ── ARP Request  (broadcast FF:FF:FF:FF:FF:FF) ─► "Who has 192.168.1.20?"
B ── ARP Reply    (unicast to A)                ─► "192.168.1.20 is at AA:BB:...:20"
A caches it in its ARP table and sends the frame.
```

- **ARP table/cache** stores IP↔MAC mappings temporarily.
- **Gratuitous ARP** = a device announces its own IP/MAC (used to detect duplicates, update caches).
- IPv6 replaces ARP with **NDP (Neighbor Discovery)**.

🧩 There is nothing like ARP in CAN because CAN never needs a device address. In Ethernet, ARP is the glue between IP (L3) and MAC (L2).

---

## 3.6 DNS (Domain Name System) 🔵

Translates human names → IP addresses: `www.example.com` → `93.184.216.34`.

```
Browser ── "IP of example.com?" ──► DNS Server ── "93.184.216.34" ──► Browser
```

🚗 DNS is rarely used inside the vehicle network (ECUs use fixed IPs/service discovery). It matters mostly for **connected/cloud** features and telematics. **SOME/IP Service Discovery** plays the "find the service" role inside the car instead of DNS.

---

## 3.7 TCP vs UDP 🟢🧩

Both are **Layer 4 (Transport)** protocols that use **ports**. This is one of the most asked interview topics.

| Feature | TCP | UDP |
|---|---|---|
| Connection | Connection-oriented (handshake) | Connectionless |
| Reliability | Guaranteed, retransmits lost data | No guarantee (fire-and-forget) |
| Ordering | In-order delivery | No ordering |
| Speed/overhead | Slower, more overhead | Fast, minimal overhead |
| Flow/congestion control | Yes | No |
| Header size | 20+ bytes | 8 bytes |
| Use cases | File transfer, **DoIP**, large SOME/IP | Streaming, **SOME/IP events**, **SD**, control |

**TCP 3-way handshake (memorize):**
```
Client ── SYN ──────────► Server
Client ◄── SYN, ACK ───── Server
Client ── ACK ──────────► Server   (connection established)
```

**TCP connection teardown:** `FIN → ACK → FIN → ACK`.

🧩 **CAN analogy:**
- **UDP is like CAN** — you put a message out; no guaranteed delivery, low overhead, low latency. Perfect for periodic sensor data and SOME/IP events.
- **TCP is like a reliable transport layer** (closest CAN analogy is **ISO-TP/CAN-TP** for segmented, acknowledged UDS transfers) — used for **DoIP** and large/critical data.

🚗 Rule of thumb in automotive: **UDP for events/streaming/SD**, **TCP for diagnostics (DoIP) and large method calls**.

---

## 3.8 Ports 🟢

- A **16-bit** number (0–65535) that identifies an application/service on a host.
- **Well-known ports:** 0–1023 (HTTP 80, HTTPS 443, DNS 53).
- **Registered / dynamic:** 1024–65535.
- A connection is identified by the **5-tuple**: `(src IP, src port, dst IP, dst port, protocol)`.

🚗 Key automotive ports:
- **DoIP:** TCP & UDP **13400**.
- **SOME/IP:** configurable (commonly 30490 for SD, 3050x range for services — project-defined).

---

## 3.9 Sockets 🔵

A **socket** is a software endpoint = **IP + port + protocol**. Applications read/write data through sockets.

```
Server socket:  bind(IP, port) → listen() → accept()     (TCP)
Client socket:  connect(server IP, server port)
UDP socket:     bind() then sendto()/recvfrom()          (no connection)
```

🚗 In AUTOSAR, the **Socket Adaptor (SoAd)** manages sockets and maps them to PDUs — you configure "socket connections" in ARXML rather than writing raw socket code (Ch 6).

---

## 3.10 Client / Server Communication 🔵

```
Server: waits, offers a service on a known IP:port
Client: initiates the request

Client ── request ──►  Server
Client ◄─ response ──  Server
```

🚗 SOME/IP formalizes this as **service-oriented communication**: a **server ECU** *offers* a service (methods, events, fields); **client ECUs** *find* it (Service Discovery), then call methods or subscribe to events. This replaces CAN's rigid "always broadcast every signal" model with "communicate only what's needed, when needed."

---

## 3.11 Routing and Switching 🟢

| | Switching (L2) | Routing (L3) |
|---|---|---|
| Uses | MAC addresses | IP addresses |
| Scope | Within one subnet/LAN | Between subnets/networks |
| Device | Switch | Router |
| Table | MAC address table | Routing table |
| Decision | "Which port has this MAC?" | "Which next-hop for this IP network?" |

**Switch learning:** a switch inspects **source MAC** of incoming frames to learn "MAC X is on port N," building its MAC table. Unknown destination → **flood** to all ports (except source).

**Router forwarding:** a router looks up the **destination IP** in its routing table, decrements TTL, and forwards to the next hop; it rewrites L2 MAC addresses at each hop (IP stays the same end-to-end).

🚗 An **automotive Ethernet switch** does L2 switching plus often **VLAN tagging** and **TSN scheduling**; the **Gateway ECU** does L3 routing between zones.

---

## 3.12 VLAN (Virtual LAN) 🟢🚗

A VLAN logically **splits one physical switch/network into isolated virtual networks**. Devices in different VLANs cannot talk directly without a router.

**802.1Q tag** — 4 bytes inserted into the Ethernet frame after Source MAC:
```
[Dst MAC][Src MAC][ 802.1Q Tag (TPID 0x8100 + PCP + VID) ][EtherType][Payload][FCS]
                     │                                     │
                     PCP = 3-bit priority (0-7)  VID = 12-bit VLAN ID (0-4095)
```
- **VID** = which VLAN (up to 4094 usable).
- **PCP (Priority Code Point)** = QoS priority — **crucial in automotive** to prioritize control/safety traffic over infotainment.

🚗 Automotive uses VLANs to **separate traffic domains** (e.g. ADAS, infotainment, diagnostics, chassis) on one physical backbone — isolation for **safety and security**. PCP feeds into **TSN** traffic prioritization.

🧩 CAN separates domains by having *physically separate buses*; Ethernet does it *logically* with VLANs on shared wiring.

---

## 3.13 NAT (Network Address Translation) 🔵

NAT rewrites IP addresses (and ports) as packets cross a router — typically mapping many **private** IPs to one **public** IP (how your home router shares one internet address).

```
LAN 192.168.1.x  ──►  [ NAT Router ]  ──►  Internet (one public IP)
private addresses          rewrites src IP/port        public address
```

🚗 NAT is **mostly used at the telematics/gateway boundary** between the in-vehicle network and external/cloud connectivity — not inside the core in-vehicle network. Good to know for connected-car and OTA discussions.

---

## 3.14 Basic Network Troubleshooting 🟢

Core tools and what they prove:

| Tool | What it does | What it proves |
|---|---|---|
| **ping** | Sends ICMP echo request/reply | Host reachable at IP level (L3) + round-trip latency |
| **arp -a** | Shows ARP cache | IP↔MAC resolution working (L2/L3 link) |
| **ipconfig / ip addr** | Shows local IP config | Your IP, mask, gateway are set correctly |
| **traceroute / tracert** | Shows hops to destination | Where routing breaks between networks |
| **netstat** | Shows sockets/ports/connections | Which apps listen/connect on which ports |
| **Wireshark** | Captures & decodes packets | Everything — the ultimate truth (Ch 7) |

**A practical troubleshooting ladder (bottom-up, matches OSI):**
1. **Link up?** Is the PHY link established (LED/status)? — L1
2. **Correct IP/subnet/gateway?** `ipconfig` — L3 config
3. **Same subnet reachable?** `ping` the peer, check `arp -a` — L2/L3
4. **Different subnet?** `ping` the gateway, then `tracert` — routing
5. **Right port/app?** `netstat`, check firewall — L4
6. **Still failing?** **Wireshark** capture to see exactly what's sent/received.

🚗 Automotive additions:
- Check **VLAN tag / PCP** — traffic on the wrong VLAN is invisible to the receiver.
- Check **link speed/mode** (100BASE-T1 both ends must match; master/slave PHY roles).
- Check **SOME/IP Service Discovery** — if SD offers aren't seen, subscriptions never happen.
- Check **PTP time sync** — if clocks aren't synced, TSN/AVB streams fail.

🧩 CAN troubleshooting (bus load, ACK errors, bus-off, DBC decode in CANoe) → Ethernet troubleshooting (link state, IP/VLAN config, Wireshark decode, SD state).

---

## 3.15 Chapter Summary

- **IP** addresses hosts (IPv4 32-bit, IPv6 128-bit); **subnet mask** splits network/host; different subnets need a **router (default gateway)**.
- **DHCP** auto-assigns IPs (automotive mostly static, DHCP for testers/DoIP).
- **ARP** resolves IP→MAC locally (the glue between L3 and L2); IPv6 uses NDP.
- **TCP = reliable/ordered/connection** (DoIP, big data); **UDP = fast/connectionless** (SOME/IP events, SD). UDP ≈ CAN's fire-and-forget.
- **Ports** identify apps; **sockets** = IP+port+protocol endpoints (managed by SoAd in AUTOSAR).
- **Switching = L2/MAC**, **routing = L3/IP**; **VLAN** logically isolates domains and carries **PCP priority** (feeds TSN).
- Troubleshoot **bottom-up** (link → IP → subnet → route → port → capture); automotive adds VLAN/PHY-mode/SD/PTP checks.

**Next:** [Chapter 4 — Ethernet Protocol Stack](Chapter_04_Ethernet_Protocol_Stack.md) — the full layer-by-layer journey of a packet, including SOME/IP and DoIP.
