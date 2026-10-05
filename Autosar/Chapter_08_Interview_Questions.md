# Chapter 8 — Interview Questions

> [◀ Chapter 7](Chapter_07_Practical_Project.md) | [Back to README](README.md) | [Next ▶ Chapter 9 — Answers](Chapter_09_Answers.md)

A curated question bank for a **5-year automotive embedded (AUTOSAR)** engineer. Model answers are in [Chapter 9 — Answers](Chapter_09_Answers.md). Try to answer **out loud** before reading the answer.

Legend: 🟢 Basic · 🟡 Intermediate · 🔴 Advanced

---

## 8.1 Basic Questions 🟢
1. What is AUTOSAR and why was it introduced?
2. What are the layers of the Classic AUTOSAR architecture?
3. What is the difference between Classic and Adaptive AUTOSAR?
4. What is the RTE and what does it do?
5. What is a Software Component (SWC)? What are its types?
6. What is the VFB (Virtual Function Bus)?
7. What is BSW? Name its three sub-layers.
8. What is MCAL and what does it abstract?
9. What is a runnable? What triggers it?
10. What is a port and a port interface?
11. What are the advantages and limitations of AUTOSAR?
12. What is ARXML?
13. What is the difference between MCAL and ECU Abstraction Layer?
14. What is a Complex Device Driver (CDD)?
15. What is a PDU?

## 8.2 Intermediate Questions 🟡
1. Explain Sender-Receiver vs Client-Server communication.
2. Explain explicit vs implicit S/R access.
3. What are the two RTE generation phases (contract vs generation)?
4. Explain the CAN transmission flow from SWC to bus.
5. Explain the CAN reception flow from bus to SWC.
6. What is the relationship between signal, I-PDU, PDU, and frame?
7. What are the Com transmission modes?
8. What is the role of PduR? What is gateway routing?
9. What does CanIf do? What are HTH and HRH?
10. Why is CanTp needed? Explain SF/FF/CF/FC.
11. What is the difference between EcuM and BswM?
12. What is SchM and how does it relate to the RTE?
13. Explain the memory stack: NvM → MemIf → Fee → Fls.
14. What is Det and when is it used?
15. What are pre-compile, link-time, and post-build configuration?
16. What is ECUC? Explain parameter definition vs value description.
17. Difference between DaVinci Developer and DaVinci Configurator?
18. What are RTE events? List the main ones.

## 8.3 Advanced Questions 🔴
1. How does the RTE guarantee data consistency? Explain exclusive areas, implicit access, and IRVs.
2. How does the AUTOSAR OS avoid priority inversion? Explain the Priority Ceiling Protocol.
3. Explain the diagnostic stack: how a UDS request flows to Dcm and back.
4. Explain the DTC lifecycle in Dem (debouncing, aging, healing, status byte).
5. How does NvM guarantee data integrity across power loss?
6. Why does Fee exist? How does wear-leveling and garbage collection work?
7. Explain WdgM's three supervision mechanisms in depth.
8. How does partial networking (PNC) work in the CAN stack?
9. Explain post-build loadable configuration and its use for variant handling.
10. How is functional safety (ISO 26262) reflected in AUTOSAR (E2E, memory partitioning, timing protection)?
11. Explain End-to-End (E2E) protection. What does the E2E library protect against?
12. How does the RTE map runnables to OS tasks, and how do you decide task allocation?
13. What are OS scalability classes (SC1–SC4)?
14. Explain measurement & calibration (XCP, MCD, calibration parameters).

## 8.4 Scenario-Based Questions 🧩
1. A new CAN signal must be added to an existing ECU. Walk through every step.
2. The application must persist a counter across power cycles. How do you implement it end-to-end?
3. A safety-relevant signal must be protected against corruption on the bus. What do you use?
4. You must add a new UDS DID (Read/Write DataByIdentifier). What config/code is needed?
5. Two ECUs must exchange a signal, but one is redesigned onto another ECU. What changes in the SWC code?
6. The ECU must go to sleep and wake up on CAN activity. Which modules coordinate this?
7. You need to route a message from CAN-A to CAN-B unchanged. How?
8. A runnable is too slow and overruns its task. How do you diagnose and resolve it?

## 8.5 Debugging Questions 🐞
1. A signal you configured is not being transmitted on the bus. How do you debug it?
2. A received signal always shows its initial/substitute value. What could be wrong?
3. The ECU keeps resetting periodically. How do you find the cause?
4. Diagnostics: the tester gets no response to a multi-frame request. What do you check?
5. The RTE generation fails after an OEM ARXML update. How do you approach it?
6. An NvM block is not being written. What are the likely causes?
7. The CAN controller goes bus-off repeatedly. How do you investigate?
8. A DTC is never getting confirmed/stored. What do you check in Dem?
9. How do you use Det to find integration bugs?
10. A value read via `Rte_Read` is stale/inconsistent. What mechanisms address this?

## 8.6 CAN + AUTOSAR Questions 🚗
1. Draw and explain the full CAN communication stack.
2. What happens on the bus when `Rte_Write` is called for a cyclic signal?
3. Difference between ComM, CanSM, and CanNm?
4. How is bus-off detected and recovered in AUTOSAR?
5. What is the difference between an I-PDU and an L-PDU?
6. How does CanIf select which hardware mailbox to use?
7. How does Com handle a missing periodic frame (Rx timeout)?
8. What is IpduM and when do you need I-PDU multiplexing?
9. How does CAN-FD affect the stack (DLC, payload)?
10. How does diagnostic communication differ from normal signal communication in the stack path?

## 8.7 RTE / BSW Questions ⚙️
1. What is the difference between RTE and SchM?
2. How does an SWC access a BSW service like NvM or Dem?
3. What are service ports?
4. Explain the role of BswM rules and action lists.
5. How does EcuM sequence the startup of the BSW?
6. What is the difference between a basic task and an extended task in the OS?
7. What are OS alarms and schedule tables? When prefer schedule tables?
8. What is an inter-runnable variable (IRV)?
9. How are BSW main functions scheduled?
10. What is the difference between Ea/Eep and Fee/Fls?

## 8.8 DaVinci / Configuration Questions 🛠️
1. What is the difference between DaVinci Developer and Configurator?
2. How do you import a communication matrix, and what formats are supported?
3. Walk through configuring a Tx PDU in Com → PduR → CanIf.
4. What is an ECU extract and how do you generate it?
5. What does the RTE generator output?
6. How do you configure a diagnostic service (e.g., a new DID) in Dcm?
7. How do you handle project variants in configuration?
8. What validation does the tool provide before generation?

## 8.9 Project-Based Questions 📋
1. Describe your AUTOSAR project, your role, and the toolchain.
2. Which modules did you configure vs develop?
3. Describe the hardest bug you solved on an AUTOSAR project.
4. How did your team split work (application vs BSW vs diagnostics)?
5. How did you test your ECU software?
6. How did you interact with the OEM and other Tier-1 teams?
7. What AUTOSAR version and stack vendor did you use, and why does it matter?
8. Give an example of a design trade-off you made (e.g., CDD vs standard driver).

---

> Study strategy: cover Chapter 9's answer, answer aloud, then check. Focus extra time on **8.4/8.5/8.6** — scenario, debugging, and CAN — because 5-year candidates are judged on *practical* depth.

> [◀ Chapter 7](Chapter_07_Practical_Project.md) | [Back to README](README.md) | [Next ▶ Chapter 9 — Answers](Chapter_09_Answers.md)
