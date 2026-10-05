# Chapter 6 — Configuration & Tools

> [◀ Chapter 5](Chapter_05_CAN_Communication.md) | [Back to README](README.md) | [Next ▶ Chapter 7 — Practical Project](Chapter_07_Practical_Project.md)

AUTOSAR is a **configuration-driven** methodology. You spend more time **configuring** than hand-coding. This chapter covers the exchange format, the config model, tools, and the full build flow.

---

## 6.1 The AUTOSAR Methodology (big picture)

```
 ┌──────────────┐   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
 │  System      │──►│  ECU         │──►│  BSW + RTE   │──►│  Build /     │
 │  Description │   │  Extract     │   │  Config      │   │  Integrate   │
 │  (VFB, SWCs, │   │  (per ECU)   │   │  (ECUC)      │   │  (compile,   │
 │  comm matrix)│   │              │   │              │   │   link,flash)│
 └──────────────┘   └──────────────┘   └──────────────┘   └──────────────┘
        │                                    │
      ARXML ◄──────── everything exchanged as ARXML ──────►
```

1. **System design** — define SWCs, VFB connections, and the **communication matrix** (which signal on which bus).
2. **System configuration** — map SWCs to ECUs → generate a **System Description** (ARXML).
3. **ECU extraction** — extract the part relevant to **one ECU** → **ECU Extract** (ARXML).
4. **ECU configuration** — configure BSW modules (**ECUC**), generate RTE + BSW.
5. **Build & integrate** — compile, link, flash, test.

---

## 6.2 ARXML — AUTOSAR XML

**ARXML** is the **standardized XML exchange format** for *all* AUTOSAR data. It is how OEMs, suppliers, and tools exchange information.

**What ARXML describes:**
| Content | Example |
|---------|---------|
| SWC descriptions | Ports, interfaces, internal behavior, runnables |
| Interfaces & data types | S/R interfaces, C/S operations, ImplementationDataTypes |
| Communication matrix | Signals, PDUs, frames, buses, timing |
| System & topology | ECUs, connectors, mapping |
| **ECU Configuration (ECUC)** | BSW module parameter values |
| Constraints, timing, calibration | Timing extensions, calibration params |

```xml
<!-- Simplified: a Sender-Receiver interface with one data element -->
<SENDER-RECEIVER-INTERFACE>
  <SHORT-NAME>IfVehicleSpeed</SHORT-NAME>
  <DATA-ELEMENTS>
    <VARIABLE-DATA-PROTOTYPE>
      <SHORT-NAME>Value</SHORT-NAME>
      <TYPE-TREF DEST="IMPLEMENTATION-DATA-TYPE">/DataTypes/uint16</TYPE-TREF>
    </VARIABLE-DATA-PROTOTYPE>
  </DATA-ELEMENTS>
</SENDER-RECEIVER-INTERFACE>
```

> **Interview note:** *"ARXML is the backbone of AUTOSAR exchange. Whether it's an SWC description, a communication matrix, or a BSW ECUC configuration — it's all ARXML. Tools import/export ARXML, which is what enables the multi-vendor workflow."*

---

## 6.3 ECUC — ECU Configuration

**ECUC** = **ECU Configuration**. It is the **complete parameter set** describing how every BSW module on a specific ECU is configured.

**Three key artifacts of the config methodology:**
| Artifact | Meaning |
|----------|---------|
| **ECUC Parameter Definition** | The *template* — what parameters a module has, their types/ranges (defined by AUTOSAR/vendor) |
| **ECUC Value Description** | The *actual values* you set for those parameters (your project's config) |
| **ECU Configuration Description** | The full ECUC = all module value descriptions for the ECU |

**Example (conceptual) — CanIf Tx PDU:**
```
CanIf
 └─ CanIfInitCfg
     └─ CanIfTxPduCfg [EngineData_Tx]
         ├─ CanIfTxPduId          = 5
         ├─ CanIfTxPduCanId       = 0x120
         ├─ CanIfTxPduDlc         = 8
         ├─ CanIfTxPduHthRef      → CAN0/HTH_0
         └─ CanIfTxPduUserTxConfirmationName = PduR
```

The **BSW generators** read the ECUC values and produce the `*_Cfg.c/.h`, `*_PBcfg.c` files for each module.

> **Interview note:** *"ECUC is where BSW behavior actually gets defined. Parameter Definitions come from AUTOSAR/vendor; you fill in the Value Description; the generator turns it into C config tables."*

---

## 6.4 Configuration classes: Pre-compile / Link-time / Post-build

BSW parameters have a **configuration variance class** — when the value becomes fixed:

| Class | Fixed at | Change requires | Use case |
|-------|----------|-----------------|----------|
| **Pre-compile time** | Compile time (`#define`, macros) | Recompile | Most efficient; fixed params |
| **Link-time** | Link time (const in separate unit) | Relink | Params fixed per variant, chosen at link |
| **Post-build (loadable)** | After build (separate config blob flashed) | Reflash config only (no recompile) | Variants selectable without rebuilding code |

> **Interview note:** *"Post-build configuration lets you change parameters (e.g., a communication matrix variant) by flashing a new config blob — no source recompilation. Pre-compile is the fastest/smallest but least flexible."*

---

## 6.5 The Tools

### DaVinci (Vector) — the most common toolchain
| Tool | Purpose |
|------|---------|
| **DaVinci Developer** | **Design SWCs** — define components, ports, interfaces, runnables, compositions; author the application architecture (VFB level). Exports SWC ARXML. |
| **DaVinci Configurator (Pro)** | **Configure the BSW/ECUC** — configure Com, PduR, CanIf, Dcm, NvM, OS, etc.; import comm matrix (DBC/Fibex/ARXML); **generate RTE + BSW** code. |
| **CANoe / CANalyzer** | **Test & simulate** the bus — send/receive frames, run test cases, restbus simulation. |
| **CANdela / CANdelaStudio** | Author **diagnostic descriptions** (CDD/ODX) used by Dcm/Dem config. |

### Other common tools
| Tool | Vendor | Purpose |
|------|--------|---------|
| **EB tresos Studio** | Elektrobit | BSW configuration & generation (alternative to DaVinci) |
| **Vector MICROSAR** | Vector | BSW stack (paired with DaVinci) |
| **EB tresos AutoCore** | Elektrobit | BSW stack |
| **Artop / Eclipse** | Open | ARXML tooling platform |
| **Matlab/Simulink + Embedded Coder** | MathWorks | Model-based SWC development (generate AUTOSAR SWC code) |
| **Vehicle Spy / Neo** | Intrepid | Bus tooling |

> **Interview note:** *"DaVinci Developer = design the software architecture (SWCs). DaVinci Configurator = configure BSW and generate RTE + BSW. Don't mix them up — that mistake is spotted instantly."*

---

## 6.6 RTE Generation (recap + tool view)

Two phases (see [Chapter 3](Chapter_03_RTE.md)):
```
 Contract Phase   :  SWC ARXML ──► Rte_<SWC>.h  (APIs only) → suppliers code SWCs
 Generation Phase :  ECU config + connections ──► Rte.c/Rte.h (full glue + tasks)
```
In **DaVinci Configurator**, the **RTE generator** produces:
- `Rte.c`, `Rte.h` — the runtime environment implementation
- `Rte_<SWC>.h` — per-component API headers
- `Rte_Type.h` — data types
- `SchM_*` — BSW scheduler code

---

## 6.7 Basic configuration workflow (step-by-step)

```
 1. Receive inputs      : System ARXML / ECU Extract + Communication matrix (DBC/Fibex/ARXML)
 2. Import              : Load into DaVinci Configurator (creates ECUC skeleton)
 3. Configure BSW       : OS, EcuM, BswM, Com, PduR, CanIf, Can, CanTp, Dcm, Dem, NvM...
 4. Map signals↔ports   : Connect SWC ports to system signals
 5. Configure RTE       : Task mapping (runnables → OS tasks), events
 6. Resolve conflicts   : Validate (tool shows errors/warnings), fix parameter issues
 7. Generate            : RTE + BSW code generation
 8. Add application code : Write/refresh SWC runnable bodies (against Rte_<SWC>.h)
 9. Build               : Compile + link (make/IDE)
10. Flash & Test        : Download to ECU; verify on CANoe / debugger
```

---

## 6.8 Build & integration flow

```
   ┌─────────────┐   ┌────────────────┐   ┌────────────┐   ┌──────────┐   ┌────────┐
   │ Config      │──►│ Generate        │──►│ Compile     │──►│ Link     │──►│ Flash  │
   │ (ECUC/ARXML)│   │ (RTE + BSW *.c) │   │ (.c → .o)   │   │ (.elf/   │   │ (to ECU│
   │             │   │ + app code      │   │             │   │  .hex/s19)│   │  flash)│
   └─────────────┘   └────────────────┘   └────────────┘   └──────────┘   └────────┘
                                                                             │
                                                                        Test/Debug
                                                                    (CANoe, debugger,
                                                                     trace, Det logs)
```

**Generated + handwritten sources that get compiled:**
| Source | Origin |
|--------|--------|
| `Rte.c/.h`, `Rte_<SWC>.h`, `SchM_*` | RTE generator |
| `Com_Cfg.c`, `PduR_PBcfg.c`, `CanIf_Cfg.c`, `Can_Cfg.c`, `Dcm_Cfg.c`, `NvM_Cfg.c` ... | BSW generators (from ECUC) |
| `Os_Cfg.c` (tasks, alarms) | OS generator |
| `<SWC>.c` (runnable bodies) | **Handwritten** application code |
| BSW module core `.c` | Vendor-supplied (MICROSAR/AutoCore) |

---

## 6.9 Common configuration pitfalls (great for interviews)
- **Signal not transmitted** → I-PDU Tx mode set to NONE, or PduR route missing, or CanIf HTH not mapped, or `Com_MainFunctionTx` not scheduled.
- **Signal wrong value** → endianness / bit position / factor-offset mismatch vs DBC.
- **RTE generation fails** → port interface mismatch, unconnected required port, data type inconsistency.
- **Diagnostics not responding** → CanTp N-PDU config missing, PduR route Dcm↔CanTp missing, wrong RxPduId, session/security misconfig.
- **NvM data not persisted** → block not in `WriteAll`, or job not completed before reset, or CRC mismatch.
- **DTC not stored** → Dem event not configured, debouncing not reaching threshold, operation cycle not started.

---

## 6.10 Interview questions from this chapter
1. What is ARXML and why is it central to AUTOSAR?
2. What is ECUC? Explain parameter definition vs value description.
3. Difference between pre-compile, link-time, and post-build configuration?
4. Difference between DaVinci Developer and DaVinci Configurator?
5. Explain the end-to-end build and integration flow.
6. What files does the RTE generator produce?
7. A signal is not being transmitted — how do you debug the configuration?

*(Answers in [Chapter 9](Chapter_09_Answers.md).)*

---

> [◀ Chapter 5](Chapter_05_CAN_Communication.md) | [Back to README](README.md) | [Next ▶ Chapter 7 — Practical Project](Chapter_07_Practical_Project.md)
