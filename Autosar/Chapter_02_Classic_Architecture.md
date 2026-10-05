# Chapter 2 — AUTOSAR Classic Architecture (Detailed)

> [◀ Chapter 1](Chapter_01_AUTOSAR_Introduction.md) | [Back to README](README.md) | [Next ▶ Chapter 3 — RTE](Chapter_03_RTE.md)

---

## 2.1 The full layered picture

```
╔══════════════════════════════════════════════════════════════════╗
║  APPLICATION LAYER                                                 ║
║  ┌────────┐  ┌────────┐  ┌────────┐   ← Software Components (SWC)  ║
║  │ SWC A  │  │ SWC B  │  │Sensor/ │      + Runnables               ║
║  │        │  │        │  │Actuator│                                ║
║  └───┬────┘  └───┬────┘  └───┬────┘                                ║
╠══════╪═══════════╪═══════════╪═════════════════════════════════════╣
║      ▼           ▼           ▼                                      ║
║  ██████████  RTE (Runtime Environment)  █████████████████████████  ║
╠══════╪══════════════════════════════════════════════════╪═════════╣
║      ▼                                                    ▼         ║
║  ┌──────────────── BASIC SOFTWARE (BSW) ─────────────────────────┐ ║
║  │  SERVICES LAYER                                               │ ║
║  │  System(OS,EcuM,BswM,SchM,Det) | Comm(Com,PduR,CanSM,CanNm)   │ ║
║  │  Memory(NvM,MemIf,Fee/Ea)      | Diag(Dcm,Dem,FiM)  Wdg(WdgM) │ ║
║  ├──────────────────────────────────────────────────────────────┤ ║
║  │  ECU ABSTRACTION LAYER                                        │ ║
║  │  CanIf, LinIf, FrIf, Eep/Fls wrappers, Port drivers wrappers  │ ║
║  ├──────────────────────────────────────────────────────────────┤ ║
║  │  MCAL (Microcontroller Abstraction Layer)                    │ ║  ┌─────┐
║  │  Can, Lin, Fr, Adc, Pwm, Spi, Dio, Port, Gpt, Wdg, Fls, Mcu  │ ║  │ CDD │ (spans
║  └──────────────────────────────────────────────────────────────┘ ║  └─────┘  layers)
╠════════════════════════════════════════════════════════════════════╣
║  MICROCONTROLLER (CPU, RAM, Flash, Peripherals)                    ║
╚════════════════════════════════════════════════════════════════════╝
```

The BSW is horizontally divided into **layers** and vertically into **stacks** (Communication stack, Memory stack, Diagnostic stack, System stack, I/O stack).

---

## 2.2 Application Layer

- Contains the **actual functional software** as **Software Components (SWCs)**.
- Each SWC contains **Runnables** (the executable functions), triggered by **RTE events** (timing, data reception, operation invocation, mode change).
- SWCs are **hardware-independent** — they only talk through **ports** to the RTE.

**Types of SWCs:**
| Type | Description |
|------|-------------|
| **Application SWC** | Pure application logic (hardware-independent) |
| **Sensor/Actuator SWC** | Interfaces to physical sensors/actuators; closest to hardware but still via RTE |
| **Service Proxy SWC** | Provides access to services on other ECUs |
| **Composition SWC** | A container grouping other SWCs (design-time only, no code) |
| **Parameter SWC** | Provides calibration/constant data |
| **ECU Abstraction / Complex Device Driver SWC** | Bridges application to specific hardware/CDD |

---

## 2.3 RTE — Runtime Environment

- The **middleware** between Application and BSW. It is the **concrete implementation of the VFB** on a specific ECU.
- Provides **standardized interfaces** so SWCs communicate without knowing peer location.
- Handles: SWC↔SWC communication, SWC↔BSW access, runnable activation, data consistency.
- Largely **generated** by the RTE generator from ARXML.

> Full detail in [Chapter 3 — RTE](Chapter_03_RTE.md).

---

## 2.4 BSW — Basic Software (overview of the 3 sub-layers)

BSW = everything below the RTE. It is divided into three main layers plus CDDs:

### (a) Services Layer (top of BSW)
Provides **basic services** to applications and other BSW:
- **Communication services:** Com, PduR, CanSM, CanNm, IpduM
- **Memory services:** NvM (Memory management)
- **Diagnostic services:** Dcm, Dem, FiM
- **System services:** OS, EcuM, BswM, SchM, Det, Time service, Watchdog Manager (WdgM)

Hardware-independent (mostly). This is where the "logic" of BSW lives.

### (b) ECU Abstraction Layer (middle of BSW)
- Abstracts the **ECU-specific layout** (which peripheral is wired to what).
- Offers an **API to access peripherals regardless of their location** (on-chip or external).
- Interfaces the MCAL upward. Examples: **CanIf**, **LinIf**, **FrIf**, memory abstraction interfaces.
- Makes higher layers independent of the **ECU hardware layout** (not just the µC).

### (c) Microcontroller Abstraction Layer (MCAL) — see 2.5

### (d) Complex Device Drivers (CDD) — see 2.7

---

## 2.5 MCAL — Microcontroller Abstraction Layer

- **Lowest software layer** — the only layer that **directly accesses µC registers & on-chip peripherals**.
- Makes upper layers **independent of the microcontroller**.
- Usually delivered by the **semiconductor/µC vendor** (Infineon, NXP, Renesas, ST).

**Typical MCAL drivers grouped:**

| Group | Drivers |
|-------|---------|
| **Communication drivers** | Can, Lin, Fr (FlexRay), Spi, Eth |
| **I/O drivers** | Dio, Port, Adc, Pwm, Icu, Ocu |
| **Memory drivers** | Fls (internal flash), Eep (internal EEPROM), Ram Test |
| **µC drivers** | Mcu (clock/reset), Gpt (timers), Wdg (watchdog), Core Test |

> **Key point for interviews:** *"MCAL is the only layer allowed to touch hardware registers directly. It makes the rest of the stack µC-independent."*

---

## 2.6 ECU Abstraction vs MCAL — the difference (common trap question)

| | **MCAL** | **ECU Abstraction Layer** |
|--|----------|----------------------------|
| Abstracts from | The **microcontroller** (registers/peripherals) | The **ECU hardware layout** (wiring, external devices) |
| Depends on | The specific µC | The ECU board design |
| Example | `Can` driver controlling on-chip CAN controller | `CanIf` giving a uniform CAN interface regardless of controller/transceiver |
| Who provides | µC vendor | Stack vendor / integrator |
| Direct HW access | **Yes** | No (goes through MCAL) |

Together they make the upper layers **fully hardware-independent** (both µC-independent and ECU-layout-independent).

---

## 2.7 Complex Device Drivers (CDD)

- Special drivers for functionality that:
  - Standard AUTOSAR BSW **does not cover**, or
  - Needs **very tight timing / direct hardware access** (e.g., injection control, PWM-based motor control, custom sensor protocols).
- A CDD can **span all layers** — it may talk to the application via RTE **and** access hardware directly, bypassing normal BSW.
- Used as an **escape hatch** to integrate non-standard or performance-critical code while staying within the AUTOSAR framework.

```
   Application (via RTE ports)
        │
        ▼
   ┌─────────┐   direct HW access
   │   CDD   │ ───────────────────► Microcontroller peripheral
   └─────────┘
```

**Interview example:** *"On our project, the crankshaft/camshaft angle-clock acquisition was implemented as a CDD because the standard ICU/GPT drivers couldn't meet the sub-microsecond timing needed for injection timing."*

---

## 2.8 Communication Stack, Memory Stack, Diagnostic Stack (vertical view)

AUTOSAR BSW is also viewed as **vertical stacks** that cut through the layers:

```
   COMMUNICATION STACK        MEMORY STACK           DIAGNOSTIC STACK
   ┌──────────────┐           ┌──────────────┐       ┌──────────────┐
   │ Com          │(Svc)      │ NvM          │(Svc)  │ Dcm   Dem    │(Svc)
   │ PduR         │           │ MemIf        │       │ FiM          │
   │ CanIf/CanTp  │(ECU Abs)  │ Fee / Ea     │(ECU)  │ (uses PduR   │
   │ Can          │(MCAL)     │ Fls / Eep    │(MCAL) │  + Com/CanTp)│
   └──────────────┘           └──────────────┘       └──────────────┘
```

| Stack | Purpose | Key modules |
|-------|---------|-------------|
| **Communication** | Send/receive signals & frames on bus | Com, PduR, IpduM, CanIf, CanTp, CanSM, CanNm, Can |
| **Memory** | Persist data in NV memory | NvM, MemIf, Fee, Ea, Fls, Eep |
| **Diagnostic** | UDS services, fault memory | Dcm, Dem, FiM, DET (dev), PduR, CanTp |
| **System** | OS, mode & ECU state management | OS, EcuM, BswM, SchM, WdgM, Det |
| **I/O** | Access sensors/actuators | IoHwAb, Adc, Pwm, Dio, Icu, Port |

---

## 2.9 Data flow through the layers (big picture)

**Example — a wheel-speed value read and transmitted on CAN:**
```
[Sensor SWC reads wheel speed]
        │ RTE_Write(...)
        ▼
[RTE] ── updates signal ──► [Com]  (signal → I-PDU packing)
        ▼
[PduR] routes the I-PDU to the CAN interface
        ▼
[CanIf] selects hardware TX object (HTH)
        ▼
[Can (MCAL)] writes to CAN controller mailbox
        ▼
[CAN Controller] transmits frame on the bus
```

> Full transmit/receive detail with API names is in [Chapter 5 — CAN Communication](Chapter_05_CAN_Communication.md).

---

## 2.10 Interview questions from this chapter
1. Explain the three sub-layers of the BSW.
2. Difference between MCAL and ECU Abstraction Layer?
3. What is a Complex Device Driver and when would you use one?
4. What are the different types of SWCs?
5. Explain the communication stack modules from top to bottom.
6. Who typically provides the MCAL, and why?

*(Answers in [Chapter 9](Chapter_09_Answers.md).)*

---

> [◀ Chapter 1](Chapter_01_AUTOSAR_Introduction.md) | [Back to README](README.md) | [Next ▶ Chapter 3 — RTE](Chapter_03_RTE.md)
