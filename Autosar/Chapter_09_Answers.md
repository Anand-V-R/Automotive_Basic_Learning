# Chapter 9 — Answers

> [◀ Chapter 8](Chapter_08_Interview_Questions.md) | [Back to README](README.md) | [Next ▶ Chapter 10 — Mock Interview](Chapter_10_Mock_Interview.md)

Model answers. For important questions the format is:
**🎤 Short answer** → **📖 Detailed explanation** → **🔧 Practical example** → **↪ Follow-ups an interviewer may ask.**

---

## 9.1 Basic Answers 🟢

### B1. What is AUTOSAR and why was it introduced?
**🎤 Short:** AUTOSAR is a standardized, layered automotive software architecture and methodology that decouples application software from hardware, enabling reusable, portable, supplier-independent ECU software.

**📖 Detailed:** Before AUTOSAR, ECU software was tightly coupled to specific microcontrollers, so it couldn't be reused or ported easily, and multi-vendor integration was painful. AUTOSAR (founded 2003 by OEMs and Tier-1s) introduced a layered architecture (Application → RTE → BSW → µC), standardized interfaces, and a common exchange format (ARXML). Its motto is *"cooperate on standards, compete on implementation."* It solves reusability, portability, standardization, scalability, and integration.

**🔧 Example:** The same "wiper control" SWC can be deployed on an Infineon Aurix BCM in one project and an NXP ECU in another — only the MCAL and configuration change, not the application logic.

**↪ Follow-ups:** *Advantages/limitations? What makes an SWC portable? What is the VFB?*

### B2. Layers of Classic AUTOSAR?
Application → **RTE** → BSW (Services → ECU Abstraction → MCAL) → Microcontroller. CDDs span vertically. RTE realizes the VFB. (See [Chapter 1](Chapter_01_AUTOSAR_Introduction.md) diagram.)

### B3. Classic vs Adaptive AUTOSAR?
**🎤 Short:** Classic = deeply-embedded, hard real-time, signal-based, C, OSEK OS, static config. Adaptive = high-performance compute, service-oriented (SOA), C++, POSIX OS, dynamic/OTA.

**📖 Detailed:** Classic targets control ECUs (engine, ABS, BCM) on microcontrollers with static scheduling and CAN/LIN/FlexRay signal communication via the RTE. Adaptive targets ADAS/HPC on microprocessors, uses `ara::com` with SOME/IP/DDS service-oriented communication, POSIX OS, dynamic memory, and supports over-the-air deployment. (Full table in [Chapter 1](Chapter_01_AUTOSAR_Introduction.md).)

**↪ Follow-ups:** *Which OS each uses? Communication paradigm? Can they coexist? (Yes, via Foundation + gateways.)*

### B4. What is the RTE?
The middleware realizing the VFB on an ECU. It provides standardized APIs for SWC↔SWC and SWC↔BSW communication, activates runnables, and ensures data consistency. Mostly generated. (See [Chapter 3](Chapter_03_RTE.md).)

### B5. What is a SWC and its types?
The unit of application software containing runnables. Types: Application, Sensor/Actuator, Service Proxy, Composition, Parameter, ECU Abstraction/CDD SWC. (See [Chapter 2](Chapter_02_Classic_Architecture.md).)

### B6. What is the VFB?
The abstract communication model at design time — SWCs communicate over the VFB without knowing peer location. The RTE implements it per ECU, enabling relocatability.

### B7. What is BSW and its sub-layers?
All standardized software below the RTE. Sub-layers: **Services**, **ECU Abstraction**, **MCAL** (plus CDDs). Organized into stacks (communication, memory, diagnostic, system, I/O).

### B8. What is MCAL?
The lowest software layer; the only layer that directly accesses µC registers/peripherals. It makes upper layers µC-independent. Usually provided by the µC vendor. Drivers: Can, Adc, Pwm, Spi, Dio, Port, Gpt, Fls, Wdg, Mcu.

### B9. What is a runnable?
An executable function inside an SWC, activated by the RTE when an RTE event occurs (timing, data received, operation invoked, mode switch, init).

### B10. Port and port interface?
A **port** is a SWC's interaction point (PPort provides, RPort requires). A **port interface** defines what flows through it (S/R data elements or C/S operations). Two ports connect only if their interfaces are compatible.

### B11. Advantages & limitations?
**Advantages:** hardware independence, reusability, standardization, scalability, reduced cost/time, supplier independence. **Limitations:** steep learning curve, expensive tools, runtime/memory overhead, configuration complexity, harder debugging, overkill for tiny ECUs. (See [Chapter 1](Chapter_01_AUTOSAR_Introduction.md).)

### B12. What is ARXML?
The standardized AUTOSAR XML exchange format for all AUTOSAR data — SWC descriptions, interfaces, communication matrix, system topology, and ECUC. It enables the multi-vendor workflow.

### B13. MCAL vs ECU Abstraction Layer?
MCAL abstracts the **microcontroller** (registers/peripherals). ECU Abstraction abstracts the **ECU layout** (wiring, external devices). Together they make upper layers fully hardware-independent. (Table in [Chapter 2](Chapter_02_Classic_Architecture.md).)

### B14. What is a CDD?
A Complex Device Driver — a custom driver for functionality not covered by standard BSW or needing tight timing/direct hardware access. It can span layers, talking to the application via RTE and hardware directly.

### B15. What is a PDU?
Protocol Data Unit — a unit of data exchanged between layers = SDU (payload) + PCI (control info). Named by layer: I-PDU (Com), N-PDU (CanTp), L-PDU (CanIf/Can).

---

## 9.2 Intermediate Answers 🟡

### I1. Sender-Receiver vs Client-Server ⭐
**🎤 Short:** S/R is one-way data exchange (publish/subscribe, 1:N). C/S is request/response operation invocation (RPC-like, N:1).

**📖 Detailed:** In S/R, a sender writes a data element and receivers read it; no return value; supports last-is-best or queued semantics. In C/S, a client invokes an operation implemented by a server and gets a return; can be synchronous (blocking) or asynchronous. S/R receiver is triggered by `DataReceivedEvent`; C/S server by `OperationInvokedEvent`.

**🔧 Example:** Vehicle speed broadcast to several SWCs → **S/R**. A SWC asking the diagnostic component to read a calibration value → **C/S** (`Rte_Call_...`).

**↪ Follow-ups:** *Explicit vs implicit S/R? Sync vs async C/S? Cardinality of each? What event triggers each?*

### I2. Explicit vs implicit S/R access ⭐
**🎤 Short:** Explicit = SWC controls exactly when it reads/writes (`Rte_Read`/`Rte_Write`). Implicit = the RTE copies data at runnable boundaries (`Rte_IRead`/`Rte_IWrite`) giving a coherent snapshot.

**📖 Detailed:** Implicit access buffers a consistent copy at runnable start and writes back at the end, avoiding race conditions and guaranteeing that a value doesn't change mid-runnable. Explicit access reads/writes the live buffer immediately, which is more up-to-date but requires care with consistency.

**🔧 Example:** A control loop that reads a signal multiple times and needs the value to be stable throughout the cycle → **implicit**. An event handler that must send immediately on a condition → **explicit**.

**↪ Follow-ups:** *When would you choose one over the other? How does implicit relate to data consistency?*

### I3. RTE generation phases ⭐
**🎤 Short:** Contract phase generates only the component API header (`Rte_<SWC>.h`) from the SWC interface, so SWCs can be coded independently. Generation phase produces the full RTE (`Rte.c/.h`, tasks, routing) after ECU configuration.

**📖 Detailed:** Contract phase needs only the SWC description → suppliers develop against a stable API before integration. Generation phase needs the complete ECU extract with connections → produces the concrete glue, buffers, and task bodies.

**↪ Follow-ups:** *Why two phases? What input does each need? What files are produced?*

### I4. CAN transmission flow ⭐ → see [Chapter 5.3](Chapter_05_CAN_Communication.md)
`SWC (Rte_Write) → Com (SendSignal, pack signal→I-PDU) → PduR (route) → CanIf (map to HTH, Can_Write) → Can (mailbox) → CAN controller → BUS`. Confirmation flows back Can→CanIf→PduR→Com→(RTE→SWC).

### I5. CAN reception flow ⭐ → see [Chapter 5.4](Chapter_05_CAN_Communication.md)
`BUS → CAN controller → Can (RxIndication) → CanIf (map HRH→PduId) → PduR → Com (unpack I-PDU→signals) → RTE → SWC (Rte_Read)`.

### I6. Signal ↔ I-PDU ↔ PDU ↔ Frame ⭐
Signal = smallest app data (bit position/length/endianness/scaling). I-PDU = several signals packed by Com. PDU = generic unit between layers (I-PDU/N-PDU/L-PDU). L-PDU = CAN ID + DLC + data. Frame = the physical CAN frame on the wire. (Diagram in [Chapter 5.2](Chapter_05_CAN_Communication.md).)

### I7. Com transmission modes?
Periodic/cyclic, Direct/Event (on change, n-times), Mixed (cyclic + on change), None. Configured per I-PDU in Com.

### I8. PduR role & gateway routing?
PduR is the central PDU router, routing I-PDUs between upper layers (Com, Dcm, IpduM) and lower layers (CanIf, CanTp) using a generated routing table. **Gateway routing** forwards a PDU from one interface to another (e.g., CAN-A → CAN-B) without involving the application.

### I9. CanIf, HTH, HRH?
CanIf is the hardware-independent CAN interface. It maps software PDU IDs to hardware objects: **HTH** (Hardware Transmit Handle → Tx mailbox) and **HRH** (Hardware Receive Handle → Rx object). It also does Tx buffering, DLC checks, controller mode control.

### I10. Why CanTp? SF/FF/CF/FC ⭐
**🎤 Short:** CanTp segments/reassembles messages larger than one CAN frame (ISO 15765-2 / ISO-TP), essential for diagnostics.

**📖 Detailed:** **SF** (Single Frame, ≤7 bytes), **FF** (First Frame, announces total length), **CF** (Consecutive Frames with sequence number), **FC** (Flow Control: CTS/Wait/Overflow + BlockSize + STmin). The receiver's FC controls how many frames and how fast the sender may send.

**🔧 Example:** A UDS ReadDataByIdentifier response of 40 bytes → FF then multiple CFs, gated by the tester's FC (BS/STmin).

**↪ Follow-ups:** *What are BS and STmin? What timers (N_As/N_Bs/N_Cr)? Where does CanTp sit in the stack?*

### I11. EcuM vs BswM ⭐
EcuM manages **fixed ECU states** (startup, sleep, wakeup, shutdown) and driver init. BswM does **flexible, rule-based mode arbitration** (arbitration → action lists) for things like communication control, PNC, and diagnostic-session behavior. They work together.

### I12. SchM vs RTE?
RTE schedules **SWC runnables**; SchM (BSW Scheduler) schedules **BSW main functions** and provides BSW exclusive areas. SchM is generated together with the RTE. Both map onto OS tasks.

### I13. Memory stack NvM→MemIf→Fee→Fls ⭐
NvM (block-based NV management) → MemIf (selects Fee/Ea by device index) → Fee (EEPROM emulation on flash, wear-leveling) → Fls (MCAL flash driver) → hardware flash. Ea/Eep is the parallel path for real EEPROM.

### I14. What is Det?
Development Error Tracer. BSW modules call `Det_ReportError` on API contract violations (null pointer, wrong param/state). Enabled in development (`DevErrorDetect`), usually disabled in production.

### I15. Pre-compile / link-time / post-build config ⭐
When a parameter becomes fixed: **pre-compile** (compile time, most efficient), **link-time** (link time), **post-build** (after build, loadable config blob flashable without recompiling — great for variants).

### I16. What is ECUC?
The complete BSW parameter set for an ECU. **Parameter Definition** = the template (what params exist, ranges) from AUTOSAR/vendor; **Value Description** = your actual values. Generators turn values into C config tables.

### I17. DaVinci Developer vs Configurator ⭐
Developer = design SWCs/architecture (VFB level, ports, interfaces, runnables). Configurator = configure BSW/ECUC and generate RTE + BSW code. Don't confuse them.

### I18. RTE events?
TimingEvent, DataReceivedEvent, DataSendCompletedEvent, OperationInvokedEvent, ModeSwitchEvent, DataReceiveErrorEvent, InitEvent, BackgroundEvent, ExternalTriggerOccurredEvent.

---

## 9.3 Advanced Answers 🔴

### A1. RTE data consistency ⭐
**🎤 Short:** Via exclusive areas (critical sections), implicit read/write (coherent copies at runnable boundaries), and inter-runnable variables (IRVs) for intra-SWC data.

**📖 Detailed:** Exclusive areas serialize access to shared data (implemented by disabling interrupts, OS resources via priority ceiling, or cooperative scheduling). Implicit access gives each runnable a coherent snapshot. IRVs are RTE-managed variables shared among runnables of the same SWC, with optional consistency guarantees.

**🔧 Example:** Two runnables incrementing a shared counter → wrap in `Rte_Enter_/Rte_Exit_ExclusiveArea` to prevent a read-modify-write race.

**↪ Follow-ups:** *How is an exclusive area implemented? Explicit vs implicit trade-offs? What is an IRV?*

### A2. OS priority inversion / Priority Ceiling ⭐
**🎤 Short:** AUTOSAR OS (OSEK) uses the **Priority Ceiling Protocol (PCP)**: each resource has a ceiling = highest priority of any task that uses it; a task holding the resource is temporarily raised to that ceiling, preventing priority inversion and deadlocks.

**📖 Detailed:** When a task locks a resource, its priority is boosted to the resource's ceiling, so no lower-priority task holding a needed resource can be preempted by a medium task and block a high task indefinitely. Because a task can't be blocked while holding a resource, deadlocks are avoided.

**↪ Follow-ups:** *Difference from priority inheritance? What are scalability classes? Category 1 vs 2 ISRs?*

### A3. Diagnostic stack / UDS flow ⭐
Tester → CAN → Can → CanIf → CanTp (reassemble) → PduR → Dcm. Dcm's DSL manages session/timing, DSD dispatches the service, DSP processes it (reading via RTE/SWCs, querying Dem, using NvM). Response goes back Dcm → PduR → CanTp (segment) → CanIf → Can → bus → tester. (See [Chapter 4.10](Chapter_04_BSW_Modules.md).)

### A4. DTC lifecycle in Dem ⭐
**📖 Detailed:** A monitor reports an event (`Dem_SetEventStatus` PASSED/FAILED). **Debouncing** (counter- or time-based) confirms the fault before it becomes "confirmed." Dem updates the **UDS status byte** (testFailed, pending, confirmed…), captures **freeze frames** and **extended data**, and persists via NvM. **Aging** removes a DTC after N fault-free operation cycles; **healing** clears the warning indicator. Dcm reads DTCs via service 0x19.

**↪ Follow-ups:** *What is debouncing? What is an operation cycle? Difference between aging and healing? What's in the status byte?*

### A5. NvM integrity across power loss ⭐
Per-block **CRC**, **redundant** blocks (two copies), valid/invalid markers, and Fee's power-fail-safe write (write new copy, then invalidate old). At startup NvM validates CRC and falls back to the redundant copy or ROM default. Always ensure the write **job completes** before reset (coordinate with BswM/EcuM `WriteAll` at shutdown).

### A6. Why Fee? Wear-leveling & GC ⭐
Flash can't be byte-erased and has limited erase cycles, so Fee emulates EEPROM on flash. **Wear-leveling** distributes writes across sectors; each write appends a new block copy rather than overwriting. **Garbage collection** reclaims space from obsolete copies by copying valid blocks to a fresh sector and erasing the old one. Handles power-fail with validity markers.

### A7. WdgM three mechanisms ⭐
**Alive supervision** (correct frequency — checkpoints hit expected number of times per window), **deadline supervision** (a start→end checkpoint pair completes within a time window), **logical/program-flow supervision** (checkpoints occur in the correct order). On failure WdgM withholds the hardware watchdog trigger → reset. (See [Chapter 4.15](Chapter_04_BSW_Modules.md).)

### A8. Partial Networking (PNC)?
PNC keeps parts of the network asleep to save power. CanNm carries PNC info; ComM/BswM arbitrate which PNCs are active; only ECUs belonging to an active PNC stay awake. Enables selective wake/sleep of function clusters on a shared bus.

### A9. Post-build loadable config?
Configuration parameters are stored in a separate flashable blob loaded at startup, so variants can be selected/updated without recompiling code — ideal for platform ECUs deployed across models with different communication matrices.

### A10. Functional safety in AUTOSAR (ISO 26262)?
Mechanisms: **E2E protection** (CRC + counter on safety signals), **memory partitioning** (OS-Applications, MPU) to isolate ASIL software, **timing protection** (execution/arrival budgets), **program flow supervision** (WdgM), and **scalability classes** (SC3/SC4). Combined with development-process rigor (MISRA, reviews, tool qualification).

### A11. End-to-End (E2E) protection ⭐
**🎤 Short:** The E2E library adds a **CRC and a sequence counter** (and data ID) to safety-relevant signals so the receiver can detect corruption, loss, repetition, insertion, incorrect sequence, and masquerading.

**📖 Detailed:** The sender calls `E2E_Protect` (adds CRC + alive counter) before transmission; the receiver calls `E2E_Check` to validate. It protects against the failure modes required by ISO 26262 for inter-ECU communication — independent of the bus.

**↪ Follow-ups:** *What failures does it detect? Where in the stack does E2E sit? What are E2E profiles?*

### A12. RTE → OS task mapping?
The integrator maps runnables to OS tasks based on period, priority, and dependencies; the RTE generates task bodies that call runnables in order. Considerations: group same-period runnables, avoid long runnables blocking high-priority tasks, respect data dependencies and exclusive-area contention.

### A13. OS scalability classes?
**SC1** (base OSEK features), **SC2** (+ timing protection), **SC3** (+ memory protection), **SC4** (+ both timing and memory protection). Higher classes support freedom-from-interference for safety.

### A14. Measurement & calibration?
Calibration parameters are tunable constants accessed via calibration ports/CDS. **XCP** (over CAN/Ethernet) with an A2L description enables online measurement and calibration from tools (INCA, CANape). Allows tuning without reflashing.

---

## 9.4 Scenario Answers 🧩

### S1. Add a new CAN signal ⭐ (see [Chapter 5.9](Chapter_05_CAN_Communication.md))
1. Extend the **communication matrix** (DBC/ARXML) with the signal/message. 2. Import into DaVinci Configurator. 3. Map signal ↔ SWC port. 4. Add **Com** signal + I-PDU (bit position, endianness, Tx mode). 5. Add **PduR** route. 6. Add **CanIf** Tx/Rx PDU + HTH/HRH. 7. Configure **Can** mailbox if needed. 8. Ensure `Com_MainFunctionTx` scheduled. 9. Generate RTE+BSW, build. 10. Test on CANoe.

### S2. Persist a counter across power cycles ⭐
Define an **NvM block** (native or redundant, with CRC) and a RAM mirror. SWC accesses via NvM service port: read at startup (`NvM_ReadAll` restores the RAM block), increment during run, and ensure `NvM_WriteBlock` / `NvM_WriteAll` at shutdown (coordinated by BswM/EcuM). Wait for the job result before assuming persistence. Provide a ROM default for first start.

### S3. Protect a safety signal on the bus?
Use the **E2E library**: sender adds CRC + counter (`E2E_Protect`), receiver validates (`E2E_Check`). Detects corruption, loss, repetition, and sequence errors as required by ISO 26262.

### S4. Add a new UDS DID (0x22/0x2E)?
In **Dcm**: configure the DID, its data length, read/write access, session/security preconditions, and link it to a data source — either an application callback (via RTE service port / `Rte_Call`) or a Dcm data element. Update the diagnostic description (CANdelaStudio/ODX). Generate and test with a diagnostic tester.

### S5. SWC moved to another ECU — code changes?
**None in the SWC code.** Because the RTE realizes the VFB, `Rte_Write/Read` stay the same; only the generated RTE and configuration change (local call becomes bus communication via Com). This is relocatability.

### S6. Sleep + wake on CAN activity?
Coordinated by **EcuM** (sleep/wakeup states), **BswM** (mode arbitration), **ComM** (communication request), **CanSM** (channel state), **CanNm** (network sleep/wakeup), **CanTrcv** (transceiver wakeup), and the **Can** driver (wakeup detection). CanNm keeps the network coordinated; a bus wakeup validated by EcuM brings the ECU back to RUN.

### S7. Route CAN-A → CAN-B unchanged?
Configure a **PduR gateway route** from CanIf(A) to CanIf(B). PduR forwards the L-PDU without application involvement (optionally with a gateway FIFO for rate adaptation).

### S8. Runnable overruns its task?
Diagnose with a **debugger/trace (TRACE32)** and OS timing/execution-time measurement. Options: move the runnable to a lower-priority/longer-period task, split it into smaller runnables, optimize the algorithm, or reassign task mapping. Check for blocking on exclusive areas/resources.

---

## 9.5 Debugging Answers 🐞

### D1. Signal not transmitted ⭐
Check in order: I-PDU **Tx mode** (not NONE) and cycle time; signal mapped to the I-PDU; **PduR** route Com→CanIf exists; **CanIf** Tx PDU + HTH mapping; **Can** controller started (CanSM/ComM in full comm); `Com_MainFunctionTx` scheduled; observe on **CANoe**; enable **Det** to catch API errors. (See [Chapter 6.9](Chapter_06_Configuration_and_Tools.md).)

### D2. Received signal shows init/substitute value ⭐
Frame not arriving (check CANoe), wrong **RxPduId/CAN ID** mapping, **DLC** mismatch, **Rx deadline** timeout applying substitute value, wrong **bit position/endianness/scaling** in Com, or update-bit logic. Verify PduR route CanIf→Com and `Com_MainFunctionRx` scheduling.

### D3. Periodic ECU reset ⭐
Likely **watchdog reset**. Check WdgM supervision failures (a supervised entity missing checkpoints), stack overflow, unhandled exception/trap, or a failing task. Use a debugger to catch the reset cause register, review `Det`/error hooks, and check WdgM/Wdg config and task timing.

### D4. No response to multi-frame diagnostic request ⭐
Check **CanTp** config (N-PDU IDs, BS, STmin, N_Bs/N_Cr timers), **PduR** routes Dcm↔CanTp and CanTp↔CanIf, correct **Rx/Tx PduIds**, **session/security** preconditions in Dcm, P2/P2* timing, and Flow Control handling. Trace on CANoe to see whether FC frames are exchanged.

### D5. RTE generation fails after ARXML update ⭐
Compare the new ARXML: unconnected required ports, **interface/data type mismatches**, changed signal↔port mapping, missing service ports. Read the generator error precisely (it names the element), fix the connection/type, re-validate, regenerate.

### D6. NvM block not written ⭐
Block not in the **WriteAll** set, job **not completed** before reset, **CRC**/config mismatch, wrong block ID, MemIf/Fee/Fls layer error, or the write request never queued. Check `NvM_MainFunction` scheduling and the job result callback.

### D7. Repeated bus-off ⭐
High error rate: check **baud rate/bit timing** mismatch, termination/wiring, **transceiver** faults, or software causing bursty Tx. In AUTOSAR, verify **CanSM** bus-off recovery config and that `Com_MainFunctionTx` isn't flooding the bus. Trace error frames on CANoe.

### D8. DTC never confirmed/stored ⭐
Event not configured in **Dem**, **debouncing** threshold never reached, **operation/driving cycle** not started, enable conditions not met, or monitor not calling `Dem_SetEventStatus(FAILED)` enough. Check Dem event config, debounce params, and NvM persistence of the fault memory.

### D9. Using Det to find integration bugs ⭐
Enable `DevErrorDetect` for the modules, implement/hook `Det_ReportError` to log Module ID, Instance, API ID, and Error ID. On a failure you immediately see *which module/API/contract* was violated (e.g., `CANIF_E_PARAM_POINTER`), pointing straight to the misconfiguration or misuse.

### D10. Stale/inconsistent `Rte_Read` value ⭐
Use **implicit access** for a coherent snapshot, wrap shared data in **exclusive areas**, or use IRVs — and check the sender's update timing and Rx deadline settings.

---

## 9.6 CAN + AUTOSAR Answers 🚗

### C1. Full CAN stack → [Chapter 5.1](Chapter_05_CAN_Communication.md). Com↔PduR↔CanIf↔Can with CanTp for diagnostics; ComM/CanSM/CanNm manage state.

### C2. What happens on `Rte_Write` for a cyclic signal?
The value is packed into the I-PDU buffer by Com; the actual bus frame is sent on the next cyclic `Com_MainFunctionTx` at the configured period — not necessarily immediately (unless the mode is direct/mixed).

### C3. ComM vs CanSM vs CanNm ⭐
**ComM** = user-facing communication manager (full/silent/no comm requests). **CanSM** = per-channel CAN state manager (start/stop/bus-off recovery). **CanNm** = network management (coordinated sleep/wakeup, keep-awake). ComM requests → CanSM controls CanIf/Can; CanNm coordinates network-wide sleep.

### C4. Bus-off detection & recovery → [Chapter 5.8](Chapter_05_CAN_Communication.md). Can detects → CanIf → CanSM runs recovery (delay/retries) → restart controller → notify ComM/Dem.

### C5. I-PDU vs L-PDU?
I-PDU = Com-level interaction PDU (packed signals, payload). L-PDU = CanIf/Can-level PDU (CAN ID + DLC + data) mapped to a hardware object.

### C6. How does CanIf pick a mailbox?
By configuration: each Tx PDU is linked to a specific **HTH** (hardware transmit handle → mailbox); each Rx object maps via **HRH** and CAN ID/filter to a RxPduId.

### C7. Missing periodic frame handling?
Com's **Rx deadline monitoring** detects the timeout, substitutes the configured replacement value, and notifies the application (via RTE) so it can react to a lost/stale signal.

### C8. IpduM / I-PDU multiplexing?
IpduM handles I-PDUs whose layout changes based on a **multiplexer selector**, allowing several signal groups to share the same CAN ID at different times. Needed when the comm matrix defines multiplexed messages.

### C9. CAN-FD effect on the stack?
Larger payload (up to 64 bytes) and separate data-phase baud rate; DLC encoding differs. Config: Can controller CAN-FD mode, larger PDUs, CanIf/Com sizing. Upper-layer flow is the same; more data per frame reduces segmentation needs.

### C10. Diagnostic vs signal path?
Normal signals: CanIf↔**PduR**↔**Com**↔RTE. Diagnostics: CanIf↔**CanTp**↔**PduR**↔**Dcm** (because responses are often multi-frame and need segmentation).

---

## 9.7 RTE / BSW Answers ⚙️

### R1. RTE vs SchM → RTE schedules SWC runnables; SchM schedules BSW main functions + BSW exclusive areas. Both generated, both map to OS tasks.

### R2. SWC access to BSW service?
Through the RTE via **service ports** connected to standardized AUTOSAR service interfaces, e.g., `Rte_Call_NvM_WriteBlock`, `Rte_Call_Dem_SetEventStatus`.

### R3. Service ports?
Ports on an SWC connected to BSW modules' service interfaces, letting the application use NvM, Dem, Dcm, EcuM modes, etc., through the RTE.

### R4. BswM rules & action lists?
BswM evaluates **arbitration rules** over mode requests; when a rule's condition is true, it executes an **action list** (init a module, enable/disable communication, switch PNC, trigger shutdown). This makes ECU behavior configurable and rule-driven.

### R5. EcuM startup sequence?
EcuM_Init runs before the OS (init MCU, basic drivers), then StartOS; EcuM's StartupTwo initializes the rest of the BSW (scheduler, communication, NvM ReadAll) and transitions to RUN, notifying BswM.

### R6. Basic vs extended task?
**Basic** tasks run to completion (no waiting), only synchronize at start/end. **Extended** tasks can **WAIT** on OS events (block and resume), so they can be suspended mid-execution.

### R7. Alarms vs schedule tables?
Alarms fire on counter values to activate tasks/set events. **Schedule tables** give deterministic, time-triggered activation of multiple expiry points in one synchronized table — preferred for complex periodic schedules and time synchronization (less drift than chained alarms).

### R8. Inter-runnable variable (IRV)?
An RTE-managed variable shared among runnables **of the same SWC**, with optional consistency (explicit/implicit) — used for internal state without external ports.

### R9. BSW main function scheduling?
`SchM`/OS calls each module's `*_MainFunction` cyclically (mapped to OS tasks with configured periods), e.g., `Com_MainFunctionTx/Rx`, `CanTp_MainFunction`, `Dem_MainFunction`, `NvM_MainFunction`.

### R10. Ea/Eep vs Fee/Fls?
**Fee** emulates EEPROM on internal **flash** (via **Fls**). **Ea** provides EEPROM abstraction over real external **EEPROM** (via **Eep**). Both sit under MemIf and serve NvM.

---

## 9.8 DaVinci / Configuration Answers 🛠️

### V1. Developer vs Configurator → see I17. Developer designs SWCs; Configurator configures BSW + generates RTE/BSW.

### V2. Import a communication matrix?
Import **DBC**, **Fibex**, or **ARXML (system extract)** into DaVinci Configurator; it creates/updates the ECUC skeleton for the communication stack (signals, PDUs, frames).

### V3. Configure a Tx PDU (Com→PduR→CanIf)?
Com: define signal + I-PDU (bit layout, Tx mode/period). PduR: add a routing path Com→CanIf. CanIf: add Tx PDU (CAN ID, DLC, HTH ref, Tx confirmation to PduR). Then generate. (See [Chapter 5.9](Chapter_05_CAN_Communication.md).)

### V4. ECU extract?
The subset of the system description relevant to one ECU (its SWCs, signals, connections), generated from the system configuration; it's the input to ECU-level BSW/RTE configuration.

### V5. RTE generator output?
`Rte.c`, `Rte.h`, `Rte_<SWC>.h`, `Rte_Type.h`, and `SchM_*` files — the runtime glue, per-component APIs, types, and BSW scheduler.

### V6. Configure a new DID in Dcm → see S4.

### V7. Handling variants?
**Post-build** configuration, **PNC**, calibration parameters, and feature flags let one code base support multiple vehicle variants by loading different configs.

### V8. Pre-generation validation?
The tool validates parameter ranges, references, mandatory settings, and consistency (e.g., unconnected ports, ID clashes), reporting errors/warnings before RTE/BSW generation.

---

## 9.9 Project-Based Answers 📋
Use [Chapter 7](Chapter_07_Practical_Project.md) as your template. Key points to always include: **AUTOSAR version + stack vendor + µC**, **which modules you configured vs developed**, a **concrete debugging story with a measurable result**, the **tools** (DaVinci, CANoe, TRACE32, CANdelaStudio), and **how the team/OEM collaboration** worked. Be specific and honest about your scope.

---

> [◀ Chapter 8](Chapter_08_Interview_Questions.md) | [Back to README](README.md) | [Next ▶ Chapter 10 — Mock Interview](Chapter_10_Mock_Interview.md)
