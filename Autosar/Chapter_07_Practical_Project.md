# Chapter 7 — Practical Project Understanding

> [◀ Chapter 6](Chapter_06_Configuration_and_Tools.md) | [Back to README](README.md) | [Next ▶ Chapter 8 — Interview Questions](Chapter_08_Interview_Questions.md)

This chapter shows **how AUTOSAR is actually used** in real projects and **how a 5-year engineer should present their project** in an interview. This is often what separates a strong candidate from a memorizer.

---

## 7.1 What a real AUTOSAR project looks like

### Typical org / role split
```
   OEM (e.g., Daimler, VW, Ford)
      │  provides: System ARXML, communication matrix (DBC/Fibex), diagnostic spec (CDD/ODX),
      │            requirements, test specs
      ▼
   Tier-1 supplier (e.g., Bosch, Continental, ZF, Aptiv)
      │  builds the ECU: hardware + AUTOSAR software
      ▼
   Teams inside Tier-1:
      • Application (SWC) team   → functional logic (control algorithms)
      • BSW/Integration team     → configure & integrate BSW stack, RTE
      • Diagnostics team         → Dcm/Dem, UDS services, DTCs
      • MCAL/HW team             → µC drivers, board bring-up
      • Test/Validation team     → HIL, CANoe, requirement-based testing
```

### Typical toolchain in a project
- **Stack:** Vector MICROSAR **or** EB tresos AutoCore
- **Config:** DaVinci Configurator + DaVinci Developer (or EB tresos Studio)
- **MCAL:** µC vendor (Infineon Aurix, NXP S32, Renesas RH850)
- **Test:** CANoe/CANalyzer, HIL (dSPACE, NI), Lauterbach TRACE32 debugger
- **Diagnostics:** CANdelaStudio (CDD/ODX), Vector Indigo / diagnostic tester
- **SCM/CI:** Git, Jenkins, requirement tools (DOORS/Polarion)

---

## 7.2 A concrete example project (use this as a template)

> **Project: Body Control Module (BCM)** on Infineon Aurix, Classic AUTOSAR R4.x, Vector MICROSAR.
>
> **Responsibilities of the ECU:** exterior lighting, central locking, wiper control, window control, and gateway routing of some CAN signals between body-CAN and comfort-CAN.
>
> **My role (BSW/Integration + partial Application, 5 yrs):**
> - Configured the **communication stack** (Com, PduR, CanIf, Can, CanTp) from the OEM DBC.
> - Configured **diagnostics** (Dcm/Dem): ~40 DTCs, DIDs (0x22/0x2E), routine controls (0x31), security access (0x27), and flashing support (0x34–0x37).
> - Configured **NvM** blocks for storing personalization and DTC data.
> - Configured **EcuM/BswM** for startup, partial networking (PNC), and shutdown sequences.
> - Developed a few **SWCs** (wiper logic, lighting state machine) and integrated Simulink-generated SWCs.
> - Wrote a **CDD** for a special PWM-based interior lighting dimming function.
> - Integrated **RTE**, resolved generation conflicts, and did on-target debugging with TRACE32.
> - Supported **HIL/CANoe** testing and fixed bus-off and timeout issues.

You should replace the details with **your** actual project, but keep this **structure**.

---

## 7.3 How the pieces fit in a real ECU (data journey)

```
   Physical switch (door) ─► Dio/Icu (MCAL) ─► IoHwAb ─► Sensor SWC ─► RTE
        │
        ▼
   Application SWC (locking logic) decides "lock all doors"
        │ Rte_Write(LockCmd)
        ▼
   Com packs LockCmd into I-PDU ─► PduR ─► CanIf ─► Can ─► BUS ─► other ECUs
        │
        ▼
   Actuator SWC drives lock motor via IoHwAb ─► Pwm/Dio (MCAL) ─► hardware
   NvM stores "last lock state" for next power cycle
   Dem logs a DTC if the motor feedback is implausible
   Dcm answers the tester when it reads that DTC (0x19) or a DID (0x22)
```

---

## 7.4 The 2-minute project pitch (interview gold)

Structure your answer as: **Context → Your role → Tech → Challenges → Impact.**

> *"I worked on a **Body Control Module** for a passenger vehicle, built on Classic AUTOSAR R4.x with **Vector MICROSAR** and an **Infineon Aurix** microcontroller. My primary responsibility was the **BSW integration and communication stack**: I configured **Com, PduR, CanIf, Can and CanTp** from the OEM's DBC, set up the **diagnostic stack (Dcm/Dem)** with around 40 DTCs and the standard UDS services including security access and flashing, and configured **NvM** for persistent data.*
>
> *On the application side, I developed a couple of **SWCs** for wiper and lighting logic and integrated **Simulink-generated components** through the RTE. I also wrote a **CDD** for a timing-critical PWM dimming feature that standard drivers couldn't handle.*
>
> *A key challenge was a **recurring bus-off** during high bus load; I traced it with **CANoe and TRACE32**, found a mis-scheduled `Com_MainFunctionTx` causing burst transmissions, corrected the task timing and CanSM recovery config, and the issue was resolved. I used **DaVinci Configurator** daily and worked closely with the OEM on the communication matrix and diagnostic spec."*

**Why this works:** it shows breadth (comm + diag + NvM + RTE), depth (a real debugging story), tools, and collaboration.

---

## 7.5 How to talk about challenges you solved

Prepare **2–3 STAR stories** (Situation, Task, Action, Result). Good AUTOSAR-flavored themes:

| Theme | Example story skeleton |
|-------|------------------------|
| **Bus-off / timing** | High load → bursts → bus-off; fixed Tx scheduling + CanSM recovery |
| **Diagnostics** | Tester couldn't read multi-frame response → CanTp BS/STmin/P2 timing misconfig; fixed FC params |
| **NvM data loss** | Reset during write → data lost; added redundant block + waited for job completion in shutdown |
| **RTE generation** | Port/interface mismatch after OEM ARXML update → resolved data type + connection issues |
| **Memory/stack** | Stack overflow in a task → analyzed with TRACE32, resized stacks, moved buffers |
| **Integration** | Multi-vendor BSW mismatch → aligned versions, adjusted ECUC, fixed callback wiring |

> **Interview tip:** Always end with a **measurable result** ("resolved intermittent resets", "reduced bus load by X%", "passed OEM diagnostic test suite").

---

## 7.6 What interviewers probe on "your project"
- "Which **AUTOSAR version** and **stack vendor**?" → Know it (R4.x, MICROSAR/AutoCore).
- "Which **modules did you configure vs develop**?" → Be honest and specific.
- "Show me the **CAN flow** for one of your signals." → Draw Tx/Rx (Chapter 5).
- "How did you **debug** a real issue?" → Have a concrete story (7.5).
- "Which **tools** did you use daily?" → DaVinci, CANoe, TRACE32, CANdelaStudio.
- "How do you handle **variants**?" → Post-build config, PNC, calibration.
- "What was **hardest**?" → Pick something technical, explain the fix.

---

## 7.7 Do's and Don'ts when presenting your project

| ✅ Do | ❌ Don't |
|------|---------|
| Know your stack, µC, and AUTOSAR version | Claim you did everything alone |
| Be specific about *your* modules | Give vague "I worked on AUTOSAR" answers |
| Have 2–3 concrete debugging stories | Memorize without understanding the flow |
| Draw diagrams when explaining | Confuse Developer vs Configurator |
| Admit what you didn't own, but show you understand it | Bluff on APIs you never used |
| Quantify impact | Overstate seniority/scope |

---

## 7.8 A 5-year engineer's expected competency checklist
- [ ] Can draw the **layered architecture** and **CAN Tx/Rx flow** from memory.
- [ ] Configured **Com/PduR/CanIf/Can/CanTp** and can debug signal issues.
- [ ] Configured **Dcm/Dem** and understands UDS services + DTC lifecycle.
- [ ] Configured **NvM/MemIf/Fee** and understands async job handling.
- [ ] Understands **RTE generation** phases and can resolve generation errors.
- [ ] Understands **EcuM/BswM** startup/shutdown/mode arbitration.
- [ ] Used **DaVinci/EB tresos**, **CANoe**, and a **debugger (TRACE32)**.
- [ ] Wrote or integrated **SWCs** and possibly a **CDD**.
- [ ] Knows **MISRA**, static analysis, and basic functional safety awareness (ISO 26262 basics).
- [ ] Has real **debugging stories** (bus-off, timeout, NvM, RTE).

---

> [◀ Chapter 6](Chapter_06_Configuration_and_Tools.md) | [Back to README](README.md) | [Next ▶ Chapter 8 — Interview Questions](Chapter_08_Interview_Questions.md)
