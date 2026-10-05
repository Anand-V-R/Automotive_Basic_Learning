# Chapter 6 — AUTOSAR Ethernet Stack

> Goal: Map the AUTOSAR **Ethernet** communication stack module-by-module against the AUTOSAR **CAN** stack you already know. By the end you can explain the full flow `App → RTE → COM/SOME-IP → PduR → SoAd/TcpIp → EthIf → Eth → PHY` and every module's responsibility.

---

## 6.1 The Two Stacks Side by Side 🟢

```
        AUTOSAR CAN stack                    AUTOSAR ETHERNET stack
────────────────────────────────────────────────────────────────────────
   Application (SWC)                        Application (SWC)
        │                                        │
       RTE                                      RTE
        │                                        │
       Com  ◄── signals/PDUs                Com  ──┐   SOME/IP (Ser. Oriented)
        │                                   │      │        │
      PduR  ◄── PDU routing                PduR ◄──┘   (Sd for discovery)
        │                                        │
     CanIf  ◄── CAN interface              SoAd (Socket Adaptor)
        │                                        │
   CAN Driver (Can)                         TcpIp (TCP/UDP/IP/ARP/ICMP/DHCP)
        │                                        │
   CAN Transceiver (CanTrcv)                EthIf (Ethernet Interface)
        │                                        │
     CAN BUS                                 Eth Driver (Eth)
                                                 │
                                            EthTrcv + PHY
                                                 │
                                            Ethernet WIRE
```

**What's reused from your CAN knowledge:** `RTE`, `Com`, `PduR`. **What's new:** `SoAd`, `TcpIp`, `EthIf`, `Eth`, `EthTrcv`, plus **SOME/IP**, **DoIP**, and **Sd** (Service Discovery). Diagnostics use `Dcm`/`Dem` above DoIP just like above CanTp/PduR on CAN.

---

## 6.2 Full Send Flow (memorize) 🟢

**Ethernet (SOME/IP or COM data):**
```
SWC → RTE → Com / SOME-IP → PduR → SoAd → TcpIp → EthIf → Eth → EthTrcv/PHY → WIRE
```
**CAN (for comparison):**
```
SWC → RTE → Com → PduR → CanIf → Can → CanTrcv → BUS
```

The **top half is identical in spirit** (RTE → Com → PduR). The **bottom half differs**: CanIf/Can/CanTrcv become SoAd/TcpIp/EthIf/Eth/EthTrcv, and you gain the SOME/IP + Sd service layer.

---

## 6.3 Module-by-Module Responsibilities

### Eth (Ethernet Driver) 🧩 = Can (CAN Driver)
- Lowest software layer; controls the **Ethernet Controller (MAC)** hardware.
- Manages **transmit/receive buffers (descriptors)**, interrupts, controller init.
- Hands frames to/from the hardware.
- **CAN equivalent:** the **Can** driver that controls the CAN controller.

### EthTrcv (Ethernet Transceiver Driver) 🧩 = CanTrcv
- Controls the **PHY** (transceiver): link up/down, speed, master/slave mode, wake/sleep.
- **CAN equivalent:** **CanTrcv** (controls the CAN transceiver, wake-up, modes).

### EthIf (Ethernet Interface) 🧩 = CanIf
- **Hardware abstraction** above Eth/EthTrcv. Lets upper layers work independently of which controller/PHY/switch is used.
- Manages **multiple Eth controllers and switch ports**, distributes frames, handles link state.
- **CAN equivalent:** **CanIf** — abstracts the CAN driver(s) from upper layers, routes L-PDUs.

### TcpIp (TCP/IP Stack) 🆕 (no direct CAN equivalent)
- Implements **IP, ARP, ICMP, UDP, TCP, DHCP** (and IPv6/NDP).
- Manages **IP addressing**, socket buffers, checksums, connection state (TCP handshakes/retransmits).
- Provides a **socket interface** upward to SoAd.
- **No CAN equivalent** — this is the entire "networking brain" CAN never needed.

### SoAd (Socket Adaptor) 🧩 ≈ CanIf's routing role, at socket level
- The **bridge between the PDU world (PduR) and the socket world (TcpIp)**.
- Maps **PDUs ↔ socket connections** (which PDU goes to which IP:port over UDP/TCP).
- Handles **PDU header** option, framing multiple PDUs into one socket, and routing received socket data back to PduR.
- **CAN analogy:** like CanIf mapping PDUs to CAN hardware objects — here it maps PDUs to sockets.

### PduR (PDU Router) 🧩 = PduR (same module, reused!)
- Routes PDUs between upper layers (Com, SOME/IP/Dcm) and lower interfaces (SoAd for Ethernet, CanIf for CAN).
- In a **gateway ECU**, PduR routes PDUs **between CAN and Ethernet** — the software heart of protocol bridging.
- **Same module you already know**, just with SoAd as an additional lower layer.

### Com (Communication) 🧩 = Com (reused)
- **Signal ↔ PDU packing/unpacking**, transmission modes, timeouts, gateway signal routing.
- On Ethernet, Com can still be used for **signal-based PDUs sent over IP** (e.g. via SoAd), *alongside* SOME/IP for service-based communication.
- **Same as CAN Com.**

### SOME/IP (Serialization + transformer) 🆕🚗
- Implements **service-oriented communication**: serializes method/event/field data into SOME/IP messages.
- In AUTOSAR it's realized via **transformers** (SOME/IP Transformer in the "COM-based transformer" chain) and the SOME/IP TP for segmentation of large messages.
- Works with **Sd** for runtime service discovery.
- **CAN equivalent:** none — CAN is signal-based (Com/DBC). This is the paradigm shift.

### Sd (Service Discovery) 🆕🚗
- Implements **SOME/IP-SD**: OfferService, FindService, Subscribe/Unsubscribe (see Ch 4.8/5.10).
- Manages service availability at runtime; tells SoAd/routing which connections to activate.
- **CAN equivalent:** none — CAN services are statically defined in the DBC.

### DoIP 🆕🚗 (diagnostics transport)
- Implements **ISO 13400** — carries **UDS** (from Dcm) over TCP/IP via SoAd/TcpIp.
- Handles **vehicle announcement, routing activation, logical addressing**.
- **CAN equivalent:** **CanTp (ISO-TP)** — the segmentation transport for UDS-on-CAN. DoIP replaces CanTp for Ethernet (TCP handles segmentation).

### Dcm / Dem (Diagnostics) 🧩 = same modules
- **Dcm** (Diagnostic Communication Manager) processes **UDS** services; **Dem** manages DTCs.
- Same modules on CAN and Ethernet — only the **transport below** changes (CanTp → DoIP).

---

## 6.4 Diagnostics Path Comparison 🟢

```
CAN diagnostics:      Dcm → PduR → CanTp → CanIf → Can → CanTrcv → BUS
Ethernet diagnostics: Dcm → PduR → DoIP → SoAd → TcpIp → EthIf → Eth → PHY → WIRE
```
- **Dcm** is unchanged (same UDS services 0x10/0x22/0x27/0x2E/0x31/0x34-37...).
- **CanTp → DoIP** is the key swap; **CanIf/Can → SoAd/TcpIp/EthIf/Eth**.

---

## 6.5 Communication Path Comparison (Signal vs Service) 🟢

**Signal-based over Ethernet (COM):**
```
SWC → RTE → Com → PduR → SoAd → TcpIp → EthIf → Eth → PHY
```
**Service-based over Ethernet (SOME/IP):**
```
SWC → RTE → SOME/IP (transformer) → PduR → SoAd → TcpIp → EthIf → Eth → PHY
                         │
                        Sd (service discovery, controls connections)
```
**Signal-based over CAN (your background):**
```
SWC → RTE → Com → PduR → CanIf → Can → CanTrcv → BUS
```

---

## 6.6 BSW vs RTE (recap in Ethernet context) 🧩

- **RTE (Runtime Environment):** the middleware between **application SWCs** and the **BSW**. SWCs call RTE APIs (send/receive, client-server) and never touch hardware directly. **Same concept on CAN and Ethernet.**
- **BSW (Basic Software):** everything below RTE — Com, PduR, SoAd, TcpIp, EthIf, Eth, DoIP, Sd, Dcm, etc. Layered into **Services / ECU Abstraction / Microcontroller Abstraction (MCAL)**.
  - **MCAL** here = **Eth driver, EthTrcv** (touch hardware).
  - **ECU Abstraction** = **EthIf**.
  - **Services** = **TcpIp, SoAd, Com, PduR, DoIP, Sd, Dcm, SecOC**.

🧩 Your CAN mental model (SWC ↔ RTE ↔ BSW ↔ MCAL ↔ hardware) is **identical** on Ethernet — only the module names in the lower layers change.

---

## 6.7 Configuration: DBC/LDF vs ARXML/Service Config 🟢🚗

| CAN world | Ethernet world |
|---|---|
| **DBC** defines CAN messages/signals | **ARXML** defines PDUs, SOME/IP services, socket connections, SD config |
| **LDF** for LIN | **Fibex/ARXML** for network & SOME/IP deployment |
| Signal → message → bus | Signal/service → PDU → socket → IP:port → VLAN |
| CanIf HRH/HTH config | SoAd socket connection config, TcpIp addressing |

🧩 The tooling changes (Vector DaVinci / EB tresos / etc. still used) but the **workflow** — configure comm matrix, generate BSW config, integrate — is the same discipline you know from CAN.

---

## 6.8 Gateway ECU in AUTOSAR (CAN ↔ Ethernet) 🟢

This is a top interview scenario for you:
```
CAN side:   CanTrcv → Can → CanIf ─┐
                                    ├──► PduR (routes/translates) ──┐
Eth side:   PHY → Eth → EthIf → TcpIp → SoAd ──────────────────────┘
```
- **PduR** performs **PDU gateway routing** between CanIf (CAN) and SoAd (Ethernet).
- **Com** may do **signal gateway** translation (signal-level mapping).
- For **service translation** (CAN signal ↔ SOME/IP event), a **signal-to-service mapping** (SOME/IP transformer + Com) is configured.
- For **diagnostics**, the gateway routes **DoIP ↔ CanTp** so a single tester can reach CAN ECUs over Ethernet.

---

## 6.9 Quick Module Mapping Table (Print This) 🟢

| Layer/Role | AUTOSAR CAN | AUTOSAR Ethernet |
|---|---|---|
| Application | SWC | SWC |
| Middleware | RTE | RTE |
| Signal packing | Com | Com (+ SOME/IP transformer) |
| Service comm | — | **SOME/IP** |
| Service discovery | — | **Sd** |
| PDU routing | PduR | PduR |
| Interface/adaptor | CanIf | **SoAd** |
| Networking stack | — | **TcpIp** (IP/ARP/ICMP/UDP/TCP/DHCP) |
| HW abstraction | CanIf | **EthIf** |
| Driver (MCAL) | Can | **Eth** |
| Transceiver | CanTrcv | **EthTrcv** (PHY) |
| Diagnostics app | Dcm/Dem | Dcm/Dem |
| Diagnostics transport | CanTp (ISO-TP) | **DoIP** |
| Security | SecOC | SecOC + MACsec/TLS |
| Config artifact | DBC/LDF | ARXML/Fibex |

---

## 6.10 Chapter Summary

- The AUTOSAR Ethernet stack **reuses RTE, Com, PduR, Dcm/Dem** — your CAN knowledge transfers directly.
- New lower layers: **Eth (driver) → EthIf (interface) → TcpIp (networking) → SoAd (socket adaptor)**; new upper layers: **SOME/IP + Sd** (service-oriented) and **DoIP** (diagnostics transport).
- Module mapping: **Can↔Eth, CanTrcv↔EthTrcv, CanIf↔EthIf (HW abstraction) + SoAd (socket routing), CanTp↔DoIP**.
- **TcpIp** is entirely new (no CAN equivalent) — it's the networking brain.
- **Gateway ECU** uses **PduR** to route/translate between CAN and Ethernet — the software core of your transition role.
- Config moves from **DBC/LDF → ARXML/Fibex** but the engineering workflow is the same.

**Next:** [Chapter 7 — Practical Learning](Chapter_07_Practical_Learning.md) — Wireshark, SOME/IP & DoIP captures, packet analysis, and debugging.
