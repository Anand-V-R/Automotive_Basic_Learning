# Chapter 4 — BSW Modules (Deep Dive)

> [◀ Chapter 3](Chapter_03_RTE.md) | [Back to README](README.md) | [Next ▶ Chapter 5 — CAN Communication](Chapter_05_CAN_Communication.md)

This is one of the **most important interview chapters**. For each module: **purpose → where it sits → key interactions → interview notes.**

---

## 4.0 Module map (where each module lives)

```
 SERVICES LAYER   : OS  EcuM  BswM  SchM  Det  WdgM  Com  PduR  Dcm  Dem  NvM
 ECU ABSTRACTION  : CanIf  CanTp  MemIf  Fee/Ea
 MCAL             : Can  Fls  Eep  Wdg  Mcu  ...
```

| Stack | Modules covered here |
|-------|----------------------|
| **System** | OS, EcuM, BswM, SchM, Det, WdgM |
| **Communication** | Com, PduR, CanIf, Can, CanTp |
| **Diagnostics** | Dcm, Dem |
| **Memory** | NvM, MemIf, Fee |

---

## 4.1 OS (AUTOSAR Operating System)

**Purpose:** Real-time, **OSEK/VDX-based** operating system providing scheduling, task management, and resource control. It is the foundation everything runs on.

**Key concepts & features:**
| Feature | Description |
|---------|-------------|
| **Tasks** | Basic (run-to-completion) & Extended (can WAIT on events) |
| **Scheduling** | Priority-based, preemptive / non-preemptive; static config |
| **Events** | Synchronization for extended tasks |
| **Alarms** | Triggered by counters; activate tasks / set events / call callbacks |
| **Counters** | Driven by hardware timer (e.g., system tick) |
| **Resources** | Mutual exclusion via **Priority Ceiling Protocol** (no deadlock/priority inversion) |
| **Schedule Tables** | Time-triggered, deterministic activation (better than chained alarms) |
| **Hooks** | ErrorHook, StartupHook, ShutdownHook, PreTaskHook, PostTaskHook |
| **Counters/ISRs** | Category 1 (no OS API) & Category 2 (may use OS API) interrupts |
| **Protection** | Memory protection, timing protection (Scalability Classes SC1–SC4) |

**Interactions:** SchM & RTE map runnables to OS tasks; alarms/schedule tables drive periodic BSW main functions; Watchdog and timing supervision rely on OS.

> **Interview note:** AUTOSAR OS is **statically configured** (tasks, priorities, alarms defined at build time — no dynamic task creation). It uses the **Priority Ceiling Protocol** to avoid priority inversion and deadlocks.

---

## 4.2 EcuM (ECU State Manager)

**Purpose:** Manages the **ECU's basic states** — startup, run, sleep, wakeup, shutdown. It initializes the BSW during startup and coordinates shutdown/sleep.

**ECU state machine (simplified):**
```
   STARTUP ──► RUN ──► SHUTDOWN ──► OFF
      ▲          │
      │          ▼
   WAKEUP ◄──── SLEEP  (low-power; wait for wakeup source)
```

| Responsibility | Detail |
|----------------|--------|
| **Startup** | Init MCU, drivers, BSW (before OS: EcuM_Init → StartOS; after OS: EcuM_StartupTwo) |
| **Wakeup handling** | Validate wakeup sources (CAN, timer, pin) |
| **Sleep** | Put ECU in low-power mode |
| **Shutdown** | Coordinated shutdown of BSW |
| **Mode arbitration** | In modern AUTOSAR, EcuM handles *fixed* states; **BswM** does the flexible mode arbitration |

> **EcuM Flex vs EcuM Fixed:** Modern designs use **EcuM (Flex)** for wakeup/sleep + **BswM** for mode arbitration. Older ones used **EcuM Fixed** (self-contained state machine).

**Interactions:** Calls driver init, notifies **BswM** of wakeup events, works with **ComM/CanSM** for communication state, uses **Mcu** for low-power modes.

---

## 4.3 BswM (BSW Mode Manager)

**Purpose:** Implements **mode arbitration and control** using configurable **rules** (Arbitration → Action). It is the "**decision engine**" of the BSW.

**Concept:** BswM evaluates **mode requests** (from EcuM, ComM, Dcm, application, etc.) against configured **rules**; when conditions match, it executes **action lists** (e.g., start/stop communication, switch partial networking, enter a diagnostic session).

```
  Mode Requests ──►  ┌───────────────┐  ──► Action Lists
  (EcuM, ComM,       │  BswM rules   │      (Init module, Enable Com,
   Dcm, App, NvM)    │ (arbitration) │       Switch PNC, Trigger shutdown...)
                     └───────────────┘
```

**Typical use cases:** control ComM channels, coordinate NvM write-all at shutdown, manage Partial Networking (PNC), diagnostic session-dependent behavior, post-run/shutdown sequences.

> **Interview note:** *"EcuM handles fixed ECU states; BswM handles flexible, rule-based mode arbitration. Together they manage ECU behavior across states."*

---

## 4.4 SchM (BSW Scheduler / Schedule Manager)

**Purpose:** The **RTE's counterpart for the BSW**. It schedules the **main functions** of BSW modules and provides **data consistency** (exclusive areas) for BSW.

| Responsibility | Detail |
|----------------|--------|
| **Schedule BSW main functions** | e.g., `Com_MainFunctionTx`, `CanTp_MainFunction`, `Dem_MainFunction` — called cyclically |
| **Exclusive areas for BSW** | `SchM_Enter/Exit` for critical sections inside BSW |
| **Mode management for BSW** | Provides BSW mode switch mechanisms |
| **Generated** | Part of the RTE module (SchM is generated together with RTE) |

```c
SchM_Enter_Com_EA0();   /* protect Com internal data */
/* ... */
SchM_Exit_Com_EA0();
```

> **Interview note:** *"RTE schedules SWC runnables; SchM schedules BSW main functions. Both are generated and mapped onto OS tasks."*

---

## 4.5 Com (COM / AUTOSAR Communication)

**Purpose:** The **signal-based communication** module. It handles **signal ↔ I-PDU packing/unpacking**, transmission modes, timeouts, and notifications. It's the bridge between the **RTE (signals)** and the **PduR (PDUs)**.

**Key responsibilities:**
| Function | Detail |
|----------|--------|
| **Signal packing** | Packs application **signals** into **I-PDUs** (byte layout, endianness, bit position) |
| **Signal unpacking** | Extracts signals from received I-PDUs |
| **Transmission modes** | **Periodic (cyclic)**, **Direct/Event (on change)**, **Mixed** |
| **Deadline monitoring** | RX timeout → substitute value + notification; TX confirmation timeout |
| **Signal gateway** | Route signals between PDUs (signal-based routing) |
| **Notifications** | Tx confirmation, Rx indication, invalidation, filtering |
| **Update bit / init value** | Manage signal freshness and defaults |

```
   RTE ──Com_SendSignal(id, &val)──► [Com packs signal into I-PDU] ──► Com_TriggerIPDUSend ──► PduR
   PduR ──Com_RxIndication(PduId,..)──► [Com unpacks I-PDU into signals] ──► Rte_Read
```

> **Interview note:** *"Com works on **signals and I-PDUs**. It does NOT know about CAN/LIN specifics — it just packs signals into I-PDUs and hands them to PduR."*

---

## 4.6 PduR (PDU Router)

**Purpose:** The **central router of PDUs**. It routes I-PDUs between the **communication interface modules** (CanIf, LinIf, FrIf), the **transport protocols** (CanTp), and the **upper layers** (Com, Dcm, IpduM).

**Routing paths:**
| Source → Destination | Example |
|----------------------|---------|
| Com → CanIf | Normal CAN signal transmission |
| CanIf → Com | Normal CAN signal reception |
| Dcm → CanTp | Diagnostic response (segmented) |
| CanTp → Dcm | Diagnostic request (segmented) |
| CanIf → CanIf | **Gateway routing** (route a frame from one bus to another) |

```
                 ┌──────────────── PduR ───────────────┐
   Com  ◄────────┤   routing table (config-driven)     ├────────► CanIf
   Dcm  ◄────────┤   - upper layer ↔ lower layer       ├────────► CanTp
   IpduM◄────────┤   - gateway (if ↔ if)               ├────────► LinIf/FrIf
                 └──────────────────────────────────────┘
```

> **Interview note:** *"PduR is a **pure router** — it has no buffering logic of its own (except gateway FIFO). Its routing table is generated from config. It enables gateway ECUs to forward frames between buses."*

---

## 4.7 CanIf (CAN Interface)

**Purpose:** The **hardware-independent interface to CAN**. It abstracts the **Can driver (MCAL)** and the number/type of CAN controllers from the upper layers (PduR, CanTp, CanNm, CanSM).

**Key responsibilities:**
| Function | Detail |
|----------|--------|
| **Tx request** | Map L-PDU → **HTH** (Hardware Transmit Handle); call `Can_Write` |
| **Rx indication** | Receive from Can driver, identify **HRH** (Hardware Receive Handle), dispatch to upper layer |
| **Tx confirmation** | Notify upper layer that frame was sent (`CanIf_TxConfirmation`) |
| **PDU ↔ HW object mapping** | Maps software PDU IDs to hardware mailboxes |
| **Controller mode control** | Start/Stop/Sleep/Wakeup of CAN controllers (with CanSM) |
| **DLC & filtering** | Data length check, software reception filtering |
| **Tx buffering** | Buffer Tx L-PDUs if hardware is busy |

```
   PduR/CanTp/CanNm ─► CanIf_Transmit() ─► Can_Write(HTH,...) ─► Can driver
   Can driver ─► CanIf_RxIndication() ─► (lookup HRH) ─► PduR/CanTp/CanNm upper layer
```

> **Interview note:** *"CanIf makes upper layers independent of **how many** CAN controllers exist and **which** hardware they are. It maps PDUs to hardware objects (HTH/HRH)."*

---

## 4.8 Can (CAN Driver — MCAL)

**Purpose:** The **MCAL driver** that directly controls the **on-chip CAN controller** hardware (registers, mailboxes, interrupts). Makes CanIf independent of the µC.

**Key responsibilities:**
| Function | Detail |
|----------|--------|
| **Init** | Configure CAN controller (baud rate, mailboxes, filters) |
| **Transmit** | `Can_Write` — write L-PDU into a hardware transmit mailbox |
| **Receive** | On Rx interrupt/poll, notify CanIf via `CanIf_RxIndication` |
| **Confirm** | On Tx interrupt, call `CanIf_TxConfirmation` |
| **Mode control** | Set controller to Start/Stop/Sleep/Wakeup |
| **Bus-off handling** | Detect bus-off, notify CanIf/CanSM |

> **Interview note:** *"`Can` is the only module that touches CAN controller registers. It works with **hardware objects (mailboxes)** and communicates with CanIf only — never directly with PduR/Com."*

---

## 4.9 CanTp (CAN Transport Protocol — ISO 15765-2)

**Purpose:** **Segmentation and reassembly** of large PDUs that don't fit in a single CAN frame (8 bytes for CAN, up to 64 for CAN-FD). Essential for **diagnostics (UDS)**.

**Frame types (ISO-TP):**
| Frame | Purpose |
|-------|---------|
| **SF** (Single Frame) | Data ≤ 7 bytes (CAN) — fits in one frame |
| **FF** (First Frame) | First segment of a multi-frame message (announces total length) |
| **CF** (Consecutive Frame) | Subsequent data segments (with sequence number) |
| **FC** (Flow Control) | Receiver → sender: Clear-To-Send / Wait / Overflow + BlockSize + STmin |

```
  Sender                         Receiver
   FF (len=100) ───────────────►
             ◄─────────────────── FC (CTS, BS=8, STmin=5ms)
   CF#1 ───────────────────────►
   CF#2 ───────────────────────►
   ...  (up to BS frames)  ────►
             ◄─────────────────── FC (next block)  ...
```

**Interactions:** Sits between **PduR** and **CanIf**. Used mainly for **Dcm (diagnostics)** and any large I-PDU. Parameters: **BlockSize (BS)**, **STmin** (separation time), **N_Ar/N_Bs/N_Cr** timeouts.

> **Interview note:** *"CanTp handles messages > single frame. Diagnostic requests/responses (UDS over CAN) always go through CanTp: Dcm ↔ PduR ↔ CanTp ↔ CanIf ↔ Can."*

---

## 4.10 Dcm (Diagnostic Communication Manager)

**Purpose:** Implements the **diagnostic server (UDS / ISO 14229 & ISO 15765)** on the ECU. Handles diagnostic **requests from a tester**, manages **sessions**, **security access**, and dispatches service requests.

**Three sub-modules:**
| Sub-module | Role |
|------------|------|
| **DSL** (Diagnostic Session Layer) | Manages sessions, connections, timing (P2), tester present |
| **DSD** (Diagnostic Service Dispatcher) | Parses incoming request, checks conditions, dispatches to handler |
| **DSP** (Diagnostic Service Processing) | Executes the actual service (read DID, routine control, etc.) |

**Common UDS services (SIDs):**
| SID | Service | Purpose |
|-----|---------|---------|
| 0x10 | DiagnosticSessionControl | Switch session (default/programming/extended) |
| 0x11 | ECUReset | Reset ECU |
| 0x14 | ClearDiagnosticInformation | Clear DTCs |
| 0x19 | ReadDTCInformation | Read fault memory |
| 0x22 | ReadDataByIdentifier | Read a DID |
| 0x27 | SecurityAccess | Seed/Key authentication |
| 0x2E | WriteDataByIdentifier | Write a DID |
| 0x31 | RoutineControl | Start/stop/request routine |
| 0x34–0x37 | Request/Transfer/Exit Download | Flashing (bootloader) |
| 0x3E | TesterPresent | Keep session alive |

**Interactions:** Receives requests via **PduR ↔ CanTp**; reads/writes data via **RTE ↔ SWCs**; queries fault memory via **Dem**; stores data via **NvM**.

```
  Tester ─► CAN ─► Can ─► CanIf ─► CanTp ─► PduR ─► Dcm ─► (RTE/Dem/NvM) ─► response back
```

> **Interview note:** *"Dcm = the UDS server. It never talks to CAN directly — diagnostics flow through CanTp because responses are often multi-frame."*

---

## 4.11 Dem (Diagnostic Event Manager)

**Purpose:** Manages the **fault memory** — processes **diagnostic events**, stores **DTCs (Diagnostic Trouble Codes)**, **freeze frames**, and **extended data**. It is the "**fault memory database**."

**Key concepts:**
| Concept | Meaning |
|---------|---------|
| **Event** | A monitored fault condition (reported by SWC/BSW monitors) |
| **DTC** | Diagnostic Trouble Code mapped to an event |
| **Debouncing** | Counter/time based confirmation before a fault is "confirmed" |
| **Freeze Frame** | Snapshot of data at fault occurrence |
| **Extended Data** | Aging/occurrence counters, additional info |
| **Operation cycle** | e.g., driving cycle; controls aging/healing |
| **Status byte** | UDS DTC status (testFailed, confirmed, pending, etc. — 8 bits) |

```
  Monitor (SWC/BSW) ─► Dem_SetEventStatus(EventId, PASSED/FAILED)
                         │ (debouncing)
                         ▼
                   Dem stores DTC + freeze frame + extended data ─► NvM (persist)
                         ▲
                   Dcm reads DTCs (0x19) ◄── Tester
```

**Interactions:** Events come from **SWCs (via RTE)** and **BSW monitors**; **Dcm** reads DTC info for the tester; **NvM** persists the fault memory.

> **Interview note:** *"Dem stores faults; Dcm reports them to the tester. A monitor calls `Dem_SetEventStatus`; after debouncing, Dem confirms the DTC and stores freeze frames, then persists via NvM."*

---

## 4.12 NvM (NVRAM Manager)

**Purpose:** Manages **non-volatile data** (EEPROM/Flash). Provides a **block-based abstraction** so applications and BSW can persist/restore data reliably.

**Key concepts:**
| Concept | Meaning |
|---------|---------|
| **NVRAM block** | A unit of persistent data (has RAM block + ROM default + NV block) |
| **Block types** | **Native** (1 NV), **Redundant** (2 copies), **Dataset** (N copies/array) |
| **CRC** | Optional per-block CRC for integrity |
| **Default (ROM) data** | Used if NV block invalid/first start |
| **Read/Write All** | `NvM_ReadAll` at startup, `NvM_WriteAll` at shutdown (via BswM/EcuM) |
| **Async operation** | Requests are queued; processed in `NvM_MainFunction` |

```
  SWC ─Rte_Call_NvM_WriteBlock()─► NvM (queue) ─► MemIf ─► Fee/Ea ─► Fls/Eep ─► HW
  Startup: EcuM/BswM ─► NvM_ReadAll ─► restore all configured blocks into RAM
```

**Interactions:** Sits on top of **MemIf**; requested by SWCs (via RTE), Dem, Dcm; driven at startup/shutdown by **EcuM/BswM**.

> **Interview note:** *"NvM is **block-based and asynchronous**. `NvM_WriteBlock` only queues the request — the actual write happens later in `NvM_MainFunction`. Always wait for completion (job result) before assuming data is stored."*

---

## 4.13 MemIf (Memory Abstraction Interface)

**Purpose:** A **thin abstraction** that lets **NvM** work with **multiple underlying memory drivers** (Fee over Flash, Ea over EEPROM) **transparently**. It selects the right device based on the block's configured device index.

```
              ┌──────── NvM ────────┐
              ▼                     ▼
          ┌────────── MemIf (device selector) ──────────┐
              ▼                     ▼
          ┌───────┐             ┌───────┐
          │  Fee  │             │  Ea   │
          └───┬───┘             └───┬───┘
              ▼                     ▼
          ┌───────┐             ┌───────┐
          │  Fls  │             │  Eep  │
          └───────┘             └───────┘
          (Flash)               (EEPROM)
```

> **Interview note:** *"MemIf's whole job is to let NvM be agnostic about whether data ends up in Flash (via Fee) or EEPROM (via Ea). It routes by device index — no logic of its own."*

---

## 4.14 Fee (Flash EEPROM Emulation)

**Purpose:** **Emulates EEPROM behavior on Flash** memory. Flash can't be byte-erased and has limited erase cycles, so Fee provides **block-based storage with wear-leveling and consistency** on top of the **Fls** driver.

**Key responsibilities:**
| Function | Detail |
|----------|--------|
| **EEPROM emulation** | Provide read/write/erase of logical blocks on flash |
| **Wear leveling** | Distribute writes across flash to extend life |
| **Consistency** | Handle power-loss during write (valid/invalid markers) |
| **Garbage collection** | Reclaim space from obsolete block copies |
| **Sector management** | Manage flash sectors, switch when full |

```
   NvM ─► MemIf ─► Fee (emulation, wear-leveling) ─► Fls (MCAL) ─► internal Flash
```

> **Interview note:** *"Fee exists because flash has no true EEPROM behavior. It handles wear-leveling, block management, and power-fail consistency so NvM sees a simple EEPROM-like interface. Ea is the equivalent for real external EEPROM (over Eep)."*

---

## 4.15 WdgM (Watchdog Manager)

**Purpose:** **Supervises the correct execution** of the software (not just a hardware watchdog kick). It monitors **program flow, timing, and aliveness** of supervised entities and triggers recovery (reset) on failure.

**Three supervision mechanisms:**
| Mechanism | Checks |
|-----------|--------|
| **Alive Supervision** | Is a supervised entity running at the expected frequency? (too few/many checkpoints) |
| **Deadline Supervision** | Does a sequence of checkpoints complete within a time window? |
| **Logical (Program Flow) Supervision** | Do checkpoints occur in the correct **order**? |

```
   Supervised Entity ─► WdgM_CheckpointReached(SE, CP)
                         │ (aggregate results)
                         ▼
   WdgM ─► if OK: WdgIf_SetTriggerCondition ─► Wdg (MCAL) ─► kick HW watchdog
        └─► if FAIL: withhold trigger ─► HW watchdog expires ─► ECU reset
```

**Interactions:** Uses **WdgIf → Wdg (MCAL)** to control hardware watchdog; reports status to **BswM/EcuM/Dem**.

> **Interview note:** *"WdgM is **software supervision**, not just a watchdog kick. It checks alive/deadline/logical flow. If a supervised entity misbehaves, WdgM stops servicing the hardware watchdog, forcing a reset."*

---

## 4.16 Det (Development Error Tracer)

**Purpose:** A **development-time error reporting** mechanism. BSW modules call **`Det_ReportError`** when API contracts are violated (null pointer, wrong parameter, uninitialized module, wrong state). Usually **enabled in development, disabled in production**.

```c
if (PduInfoPtr == NULL_PTR) {
    Det_ReportError(CANIF_MODULE_ID, instanceId, apiId, CANIF_E_PARAM_POINTER);
    return E_NOT_OK;
}
```

| Aspect | Detail |
|--------|--------|
| **When** | Development phase (integration/debug) |
| **What** | Parameter checks, state checks, contract violations |
| **Config** | Controlled by `<Module>DevErrorDetect` switches; compiled out in production for performance |
| **Runtime errors** | Modern AUTOSAR also has **Det runtime errors** and transient fault reporting |

> **Interview note:** *"Det is your first debugging aid in AUTOSAR. Enable `DevErrorDetect`, hook `Det_ReportError`, and you immediately see which module/API/error code was violated. It's turned off in production to save runtime."*

---

## 4.17 Quick revision table — all modules

| Module | Layer | One-line purpose | Talks to |
|--------|-------|------------------|----------|
| **OS** | System | Real-time scheduling (OSEK) | Everything |
| **EcuM** | System | ECU states: startup/sleep/wakeup/shutdown | BswM, Mcu, ComM |
| **BswM** | System | Rule-based mode arbitration | EcuM, ComM, Dcm, Com |
| **SchM** | System | Schedule BSW main functions + BSW exclusive areas | All BSW, OS |
| **Det** | System | Development error reporting | All BSW |
| **WdgM** | System | Software supervision (alive/deadline/logical) | Wdg, BswM, Dem |
| **Com** | Comm | Signal ↔ I-PDU packing, Tx modes, timeouts | RTE, PduR |
| **PduR** | Comm | Route PDUs between layers/buses | Com, Dcm, CanIf, CanTp |
| **CanIf** | Comm (ECU Abs) | HW-independent CAN interface (PDU↔HTH/HRH) | PduR, CanTp, Can |
| **Can** | Comm (MCAL) | Control CAN controller registers | CanIf |
| **CanTp** | Comm (ECU Abs) | Segment/reassemble (ISO-TP) for diagnostics | PduR, CanIf |
| **Dcm** | Diag | UDS diagnostic server | PduR/CanTp, RTE, Dem, NvM |
| **Dem** | Diag | Fault memory (DTCs, freeze frames) | Monitors (RTE), Dcm, NvM |
| **NvM** | Memory | Block-based NV data management | MemIf, RTE, EcuM/BswM |
| **MemIf** | Memory (ECU Abs) | Select Fee/Ea for NvM | NvM, Fee, Ea |
| **Fee** | Memory (ECU Abs) | EEPROM emulation on Flash + wear-leveling | MemIf, Fls |

---

## 4.18 Interview questions from this chapter
1. Difference between EcuM and BswM?
2. What is the role of SchM vs RTE?
3. Explain the CAN communication stack modules top-to-bottom.
4. Why is CanTp needed? Explain SF/FF/CF/FC.
5. How do Dcm and Dem work together in a diagnostic session?
6. Explain the memory stack: NvM → MemIf → Fee → Fls.
7. Why do we need Fee? Why not write directly to flash?
8. What are the three supervision mechanisms of WdgM?
9. What is Det and when is it used?
10. What does PduR do and what is gateway routing?

*(Answers in [Chapter 9](Chapter_09_Answers.md).)*

---

> [◀ Chapter 3](Chapter_03_RTE.md) | [Back to README](README.md) | [Next ▶ Chapter 5 — CAN Communication](Chapter_05_CAN_Communication.md)
