# Chapter 10 — Mock Interview (5-Year AUTOSAR)

> [◀ Chapter 9](Chapter_09_Answers.md) | [Back to README](README.md)

A realistic, full-length simulation of a **5-year experience AUTOSAR interview**. Read the **interviewer (Q)** line, answer out loud, then compare with the **ideal candidate (A)** response. Notes in *italics* explain *why* the answer works.

**Format:** three rounds — (1) Warm-up & fundamentals, (2) Deep technical (CAN + BSW + RTE), (3) Scenario/debugging + project + wrap-up.

---

## Round 1 — Warm-up & Fundamentals (10–15 min)

**Q1. Tell me briefly about yourself and your AUTOSAR experience.**
**A:** *"I'm an embedded software engineer with about 5 years in automotive, mostly on **Classic AUTOSAR**. I've worked on a Body Control Module using **Vector MICROSAR** on an **Infineon Aurix** microcontroller. My core strength is the **communication and diagnostic stacks** — I configure Com, PduR, CanIf, Can, CanTp, and Dcm/Dem — and I've also developed and integrated SWCs through the RTE, plus written a CDD for a timing-critical feature. Day to day I use **DaVinci Configurator and Developer, CANoe, and TRACE32**."*
> *Concise, specific, signals both breadth and depth, names real tools.*

**Q2. In one sentence, what is AUTOSAR and why do we use it?**
**A:** *"AUTOSAR is a standardized, layered automotive software architecture that decouples application software from hardware, so software becomes reusable, portable, and supplier-independent."*

**Q3. Draw or describe the Classic AUTOSAR layers.**
**A:** *"Top to bottom: **Application layer** with SWCs, then the **RTE** which realizes the VFB, then the **BSW** split into **Services**, **ECU Abstraction**, and **MCAL**, and finally the **microcontroller**. Complex Device Drivers span vertically and can bypass the standard layers for direct hardware access."*
> *If in person, actually draw it. The interviewer wants to see the mental model.*

**Q4. What's the difference between Classic and Adaptive AUTOSAR?**
**A:** *"Classic is for deeply-embedded, hard real-time control ECUs — C, OSEK OS, static configuration, signal-based CAN/LIN/FlexRay communication via the RTE. Adaptive is for high-performance compute like ADAS — C++, POSIX OS, service-oriented communication with SOME/IP through `ara::com`, dynamic memory, and OTA updates."*

**Q5. What is the RTE and how is it related to the VFB?**
**A:** *"The VFB is the abstract communication model at design time — SWCs talk over it without knowing where the peer is. The RTE is the concrete implementation of the VFB on a specific ECU. If the peer is local, it's a buffer copy or direct call; if remote, the RTE routes through Com and the bus. That's what makes SWCs relocatable — the code doesn't change if a component moves to another ECU."*
> *Connecting VFB → RTE → relocatability is exactly what a 5-year candidate should articulate.*

**Q6. What are the three sub-layers of the BSW?**
**A:** *"Services layer — Com, Dcm, Dem, NvM, OS, EcuM, BswM. ECU Abstraction layer — CanIf, and memory abstraction. And MCAL — the drivers that touch the microcontroller directly like Can, Adc, Pwm, Fls. MCAL abstracts the µC; ECU Abstraction abstracts the board/ECU layout."*

---

## Round 2 — Deep Technical (20–30 min)

**Q7. Walk me through the complete CAN transmission flow, with module names.**
**A:** *"The SWC calls `Rte_Write` for a signal. The RTE forwards it to **Com** via `Com_SendSignal`, which packs the signal into an **I-PDU** — applying bit position, endianness, and scaling. On the Tx trigger — cyclic or on-change — Com hands the I-PDU to **PduR**, which routes it to **CanIf** with `CanIf_Transmit`. CanIf maps the PDU to a hardware transmit handle, the **HTH**, and calls `Can_Write`. The **Can** driver writes the CAN ID, DLC, and data into a hardware mailbox and requests transmission; the **CAN controller** arbitrates and puts the **frame** on the bus. On completion, `CanIf_TxConfirmation` propagates back up through PduR to Com, which can notify the SWC."*
> *Fluent end-to-end with API names = strong signal. See [Chapter 5.3](Chapter_05_CAN_Communication.md).*

**Q8. Now the reception flow.**
**A:** *"A frame arrives at the controller; the **Can** driver's Rx interrupt reads the mailbox, identifies the **HRH**, and calls `CanIf_RxIndication`. **CanIf** maps HRH plus CAN ID to a RxPduId, does optional filtering and DLC checks, and calls `PduR_CanIfRxIndication`. **PduR** routes to **Com** via `Com_RxIndication`, which unpacks the signals from the I-PDU and updates its buffers, and starts Rx deadline monitoring. Then the SWC gets the value either via a DataReceivedEvent-triggered runnable or a cyclic `Rte_Read`."*

**Q9. What's the relationship between a signal, an I-PDU, a PDU, and a frame?**
**A:** *"A **signal** is the smallest application data item with a bit position, length, endianness, and scaling. Com packs several signals into an **I-PDU**, which is the payload. **PDU** is the generic term for a unit passed between layers — it's an I-PDU at Com, an N-PDU at CanTp, and an L-PDU at CanIf/Can, where it also carries the CAN ID and DLC. The **frame** is the actual physical CAN message on the wire with SOF, ID, control, data, CRC, ACK, and EOF."*

**Q10. Where does CanTp come in and why?**
**A:** *"CanTp does ISO 15765-2 segmentation and reassembly for messages larger than a single frame — mainly diagnostics. It defines Single Frame for up to 7 bytes, First Frame that announces total length, Consecutive Frames with sequence numbers, and Flow Control from the receiver that carries Clear-To-Send, BlockSize, and STmin. So a diagnostic path is CanIf ↔ CanTp ↔ PduR ↔ Dcm, unlike normal signals which go CanIf ↔ PduR ↔ Com."*

**Q11. Explain the difference between Sender-Receiver and Client-Server.**
**A:** *"Sender-Receiver is one-way data exchange, like publishing a signal — 1-to-N, no return value, using `Rte_Write` and `Rte_Read`, and the receiver is triggered by a DataReceivedEvent. Client-Server is a request/response operation invocation, like an RPC — N-to-1, using `Rte_Call`, which can be synchronous or asynchronous, and the server runnable is triggered by an OperationInvokedEvent."*

**Q12. How does the RTE ensure data consistency?**
**A:** *"Three main mechanisms: **exclusive areas** for critical sections — implemented via interrupt disabling or OS resources with priority ceiling; **implicit access**, where the RTE gives each runnable a coherent copy at its boundaries so the value can't change mid-runnable; and **inter-runnable variables** for data shared within one SWC. For example, two runnables incrementing a shared counter would race, so I'd wrap the access in `Rte_Enter`/`Rte_Exit` of an exclusive area."*

**Q13. Difference between EcuM and BswM?**
**A:** *"EcuM manages the **fixed ECU states** — startup, run, sleep, wakeup, shutdown — and the driver initialization sequence. BswM does **flexible, rule-based mode arbitration**: it evaluates mode requests from EcuM, ComM, Dcm, and the application against configured rules, and runs action lists — like enabling communication, switching a partial network, or triggering a coordinated shutdown. They work together."*

**Q14. Explain the memory stack.**
**A:** *"The application or BSW requests **NvM**, which is block-based and asynchronous. NvM goes through **MemIf**, which selects the right device by index — **Fee** over internal flash via **Fls**, or **Ea** over external EEPROM via **Eep**. Fee exists because flash has no native EEPROM behavior, so it handles block management, wear-leveling, and power-fail consistency. At startup EcuM triggers `NvM_ReadAll`, and at shutdown BswM/EcuM trigger `NvM_WriteAll`. The key gotcha is that `NvM_WriteBlock` only queues — the write happens later in `NvM_MainFunction`, so you must wait for the job result before assuming persistence."*
> *That "gotcha" shows real experience.*

**Q15. What are the three WdgM supervision mechanisms?**
**A:** *"Alive supervision checks whether a supervised entity runs at the expected frequency. Deadline supervision checks whether a start-to-end checkpoint pair completes within a time window. And logical or program-flow supervision checks that checkpoints occur in the correct order. If any fails, WdgM withholds the hardware watchdog trigger, which forces a reset."*

---

## Round 3 — Scenario, Debugging & Project (20–30 min)

**Q16. A signal you configured isn't appearing on the bus. How do you debug it?**
**A:** *"I work top-down through the Tx path. First, is the I-PDU Tx mode set — not NONE — with a valid cycle time, and is the signal actually mapped to that I-PDU? Then does PduR have the Com-to-CanIf route? Is the CanIf Tx PDU configured with the right CAN ID and HTH mapping? Is the CAN controller started — is ComM/CanSM in full communication? Is `Com_MainFunctionTx` scheduled in a task? I'd watch the bus on CANoe to confirm nothing goes out, and enable Det to catch any API-level error. Usually it's a missing PduR route, a wrong Tx mode, or an unscheduled main function."*
> *Structured, hypothesis-driven debugging is exactly what they want at 5 years.*

**Q17. The ECU keeps resetting every few seconds. Where do you look?**
**A:** *"My first suspicion is a **watchdog reset**. I'd check WdgM for a supervised entity missing its checkpoints, then look for a **stack overflow** or an unhandled trap. I'd read the reset-cause register at startup with the debugger, check error hooks and Det logs, and verify task timing. Often it's a runnable overrunning and starving the entity that services the watchdog, or a stack sized too small for a deep call path."*

**Q18. The tester gets no response to a multi-frame diagnostic request. What do you check?**
**A:** *"Because it's multi-frame, I focus on **CanTp** and the diagnostic routing. I check the CanTp N-PDU configuration, BlockSize, STmin, and the N_Bs/N_Cr timers; the PduR routes between Dcm, CanTp, and CanIf; the correct Rx/Tx PduIds; and Dcm session and security preconditions and P2 timing. I'd trace on CANoe to see whether the First Frame goes out and whether the tester's Flow Control is received and honored. A common cause is a missing or wrong Flow Control handling or a PduR route gap."*

**Q19. You need to persist a personalization counter across power cycles. Implement it end-to-end.**
**A:** *"I define an NvM block — I'd make it redundant with a CRC for a personalization value — with a RAM mirror and a ROM default. At startup `NvM_ReadAll` restores the RAM block; the SWC reads and increments it via its NvM service port during run. At shutdown, BswM/EcuM trigger `NvM_WriteAll`, and I make sure the write job completes before the ECU powers down. The ROM default covers the first-ever start when the NV block is still invalid."*

**Q20. A safety-relevant signal must be protected on the bus. What do you use?**
**A:** *"The **E2E library**. The sender calls `E2E_Protect` to add a CRC and an alive counter, and the receiver calls `E2E_Check`. That lets the receiver detect corruption, loss, repetition, insertion, and incorrect sequence — the failure modes ISO 26262 requires for inter-ECU communication — independent of the bus itself."*

**Q21. Tell me about the hardest bug you solved.**
**A:** *"On the BCM, we had intermittent **bus-off** under high bus load. Using CANoe I saw bursts of frames clustering, and with TRACE32 I found that `Com_MainFunctionTx` was mapped to a task whose timing occasionally coincided with other high-load activity, producing transmission bursts that pushed the controller into error-passive and then bus-off. I corrected the task scheduling so Tx was spread evenly, and tuned the **CanSM** bus-off recovery parameters. After that the resets disappeared and we passed the OEM's bus-load test. The lesson was that a communication problem was really a **scheduling** problem."*
> *STAR structure, real tools, measurable result, and a reflective takeaway.*

**Q22. Which modules did you configure versus develop?**
**A:** *"I **configured** the communication stack — Com, PduR, CanIf, Can, CanTp — and the diagnostic stack Dcm/Dem, plus NvM, EcuM, and BswM. I **developed** a couple of application SWCs for wiper and lighting logic, integrated Simulink-generated SWCs through the RTE, and wrote a **CDD** for PWM-based interior dimming that the standard drivers couldn't time precisely. I don't claim to have written the MCAL — that came from the silicon vendor — but I configured and integrated it."*
> *Honesty about scope + clear ownership boundaries builds trust.*

**Q23. Why did you use a CDD instead of a standard driver there?**
**A:** *"The dimming needed sub-millisecond, jitter-sensitive PWM updates synchronized to a specific event, and the standard Pwm/Gpt driver path through the RTE and BSW added latency and jitter we couldn't tolerate. A CDD let us access the timer hardware directly for the critical part while still exposing a clean interface to the application via the RTE. It was a deliberate trade-off: we gave up some portability for the timing we needed, and documented it."*

**Q24. Any questions for me?**
**A:** *"Yes — which AUTOSAR version and stack vendor does this project use, how is the work split between application and BSW teams, and what does the diagnostic and safety scope look like for this ECU?"*
> *Asking targeted questions signals genuine engagement and seniority.*

---

## Interviewer's evaluation rubric (what they're scoring)

| Area | Weak (1–2) | Strong (4–5) |
|------|-----------|--------------|
| **Architecture** | Vague layers | Draws layers + VFB/RTE + relocatability |
| **CAN flow** | Confuses modules | Fluent Tx/Rx with API names |
| **BSW depth** | Only names modules | Explains interactions + gotchas |
| **Debugging** | "I'd check the code" | Structured, hypothesis-driven, tool-based |
| **Diagnostics** | Only lists UDS SIDs | Explains Dcm/Dem/CanTp interplay |
| **Project ownership** | Vague "worked on AUTOSAR" | Specific scope, honest boundaries, measurable results |
| **Communication** | Rambling | Concise, structured, draws diagrams |

---

## Final tips for the real interview
1. **Draw.** Whenever you can, sketch the layer diagram or CAN flow — it demonstrates a real mental model.
2. **Structure answers.** Short answer first, then detail, then an example. (The Chapter 9 format.)
3. **Tell stories.** Have 2–3 STAR debugging stories ready (bus-off, diagnostics, NvM).
4. **Be honest about scope.** Say what you configured vs developed vs integrated.
5. **Quantify.** "Resolved intermittent resets", "passed the OEM test suite", "reduced bus load".
6. **Know your stack.** AUTOSAR version, vendor (MICROSAR/AutoCore), µC, tools.
7. **Connect concepts.** VFB→RTE→relocatability; Com→PduR→CanIf→Can; Dcm↔CanTp↔PduR.
8. **Stay calm on "I don't know."** Reason out loud from the architecture rather than bluffing an API name.

---

> **You've reached the end.** Re-read [Chapter 4 (BSW)](Chapter_04_BSW_Modules.md) and [Chapter 5 (CAN flow)](Chapter_05_CAN_Communication.md) until you can reproduce them from memory, build your project story from [Chapter 7](Chapter_07_Practical_Project.md), and drill [Chapter 8](Chapter_08_Interview_Questions.md) against [Chapter 9](Chapter_09_Answers.md). Good luck!
>
> [◀ Chapter 9](Chapter_09_Answers.md) | [Back to README](README.md)
