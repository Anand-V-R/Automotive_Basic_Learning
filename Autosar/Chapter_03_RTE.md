# Chapter 3 — RTE (Runtime Environment)

> [◀ Chapter 2](Chapter_02_Classic_Architecture.md) | [Back to README](README.md) | [Next ▶ Chapter 4 — BSW Modules](Chapter_04_BSW_Modules.md)

---

## 3.1 What is the RTE?

The **RTE (Runtime Environment)** is the **communication middleware** of an AUTOSAR ECU. It is the **concrete implementation of the Virtual Function Bus (VFB)** for a *specific* ECU.

> **Interview one-liner:**
> *"The RTE is the glue layer that realizes the VFB on a real ECU. It provides standardized APIs so SWCs can communicate with each other and with the BSW without knowing where the peer is located or how the communication physically happens."*

### What the RTE does
1. **SWC ↔ SWC communication** (intra-ECU: direct call/buffer; inter-ECU: via Com/bus).
2. **SWC ↔ BSW communication** (via Service Ports, e.g., to NvM, Dcm, Dem).
3. **Runnable activation/scheduling** — maps RTE events to OS tasks (with SchM/OS).
4. **Data consistency** — protects shared data (exclusive areas, copies).
5. **Decoupling** — SWC code is identical whether the peer is local or remote (relocatability).

The RTE is **mostly generated code** — you configure ports/interfaces in ARXML, and the **RTE Generator** produces `Rte.c/Rte.h`, `Rte_<SWC>.h`, etc.

---

## 3.2 Software Component (SWC) — building blocks

```
        ┌───────────────────── SWC "SpeedController" ─────────────────────┐
        │                                                                  │
  ─────►◯ RPort (receive)                              PPort (provide) ◯──────►
 (SR:   │  "WheelSpeed"                                  "TorqueReq"    │  (SR)
  data) │                                                              │
        │   ┌──────────────┐         ┌──────────────┐                  │
  ─────►◯   │  Runnable1   │         │  Runnable2   │                  │
  (CS:  │   │ (10ms cyclic)│         │(on data recv)│                  │
  op)   │   └──────────────┘         └──────────────┘                  │
        │        Internal Behavior (runnables, events, exclusive areas)│
        └──────────────────────────────────────────────────────────────┘
```

| Element | Meaning |
|---------|---------|
| **SWC** | Unit of application software |
| **Port** | Interaction point of a SWC (data/services in or out) |
| **PPort** (Provide) | The SWC **provides** data / a service |
| **RPort** (Require) | The SWC **requires** (consumes) data / a service |
| **PRPort** | Both provide & require (rare) |
| **Port Interface** | Defines *what* flows through a port (data elements / operations) |
| **Runnable (RunnableEntity)** | A function inside the SWC, executed by the RTE |
| **RTE Event** | The trigger that activates a runnable |
| **Internal Behavior** | Describes runnables, events, exclusive areas of a SWC |
| **Implementation** | The actual C source of the SWC |

---

## 3.3 Ports & Port Interfaces

A **port** is typed by a **port interface**. Two components can connect only if their interfaces are compatible.

### Types of Port Interfaces

| Interface type | Used for | Port kind |
|----------------|----------|-----------|
| **Sender-Receiver (S/R)** | Data-oriented (signals, values) | Sender = PPort, Receiver = RPort |
| **Client-Server (C/S)** | Function/operation invocation | Server = PPort, Client = RPort |
| **Mode-Switch** | Mode communication (e.g., ECU modes) | Mode manager provides, users require |
| **NV Data** | Access to non-volatile data | via NvM |
| **Parameter** | Calibration/constant parameters | Parameter component provides |
| **Trigger** | External trigger events | Trigger source/sink |

---

## 3.4 Sender-Receiver (S/R) Communication

**Data-oriented, one-way.** A sender writes a data element; one or more receivers read it. Think "publishing a signal."

```
  ┌──────────┐   Rte_Write_Speed_Value(200)   ┌──────────┐   ┌──────────┐
  │ Sender   │ ───────────────────────────►   │   RTE    │──►│ Receiver │
  │  SWC     │       PPort  "Speed"            │ (buffer) │   │   SWC    │
  └──────────┘                                 └──────────┘   └──────────┘
                                                     Rte_Read_Speed_Value(&x)
```

### S/R characteristics
- **Direction:** one-way (sender → receiver(s)); supports **1:N** (broadcast).
- **Semantics:**
  - **Last-is-Best (data semantics):** receiver always gets the latest value (implicit/explicit). No queue.
  - **Queued (event semantics):** each write is queued; receiver reads events FIFO (no loss until queue full).
- **Modes:**
  - **Explicit** access: `Rte_Write` / `Rte_Read` — SWC controls exactly when.
  - **Implicit** access: `Rte_IWrite` / `Rte_IRead` — RTE copies data at runnable start/end (coherent snapshot, avoids race conditions).

### Generated API examples
```c
/* Sender side (PPort "SpeedPort", data element "Value") */
Std_ReturnType Rte_Write_SpeedPort_Value(uint16 data);

/* Receiver side (RPort "SpeedPort", data element "Value") */
Std_ReturnType Rte_Read_SpeedPort_Value(uint16 *data);

/* Implicit variants (copied by RTE at runnable boundaries) */
void   Rte_IWrite_Run_SpeedPort_Value(uint16 data);
uint16 Rte_IRead_Run_SpeedPort_Value(void);
```

> **Return status:** `RTE_E_OK`, `RTE_E_LOST_DATA`, `RTE_E_MAX_AGE_EXCEEDED`, `RTE_E_NEVER_RECEIVED`, `RTE_E_TIMEOUT`, etc.

---

## 3.5 Client-Server (C/S) Communication

**Operation-oriented, request/response.** A client **invokes an operation** implemented by a server (like a remote function call / RPC).

```
  ┌──────────┐  Rte_Call_Diag_ReadDTC(&dtc)  ┌──────────┐   invoke    ┌──────────┐
  │ Client   │ ──────────────────────────►   │   RTE    │ ──────────► │ Server   │
  │  SWC     │ ◄──────────────────────────   │          │ ◄────────── │   SWC    │
  └──────────┘        return value           └──────────┘   result    └──────────┘
```

### C/S characteristics
- **Direction:** request → response (bidirectional).
- **Relationship:** **N:1** (many clients, one server) is typical.
- **Invocation types:**
  - **Synchronous:** client blocks until the server returns the result.
  - **Asynchronous:** client triggers the call, continues, and later collects the result (`Rte_Result_...`).
- The **server runnable** is activated by an **`OperationInvokedEvent`**.

### Generated API examples
```c
/* Client calls the server operation "ReadData" on port "DiagPort" */
Std_ReturnType Rte_Call_DiagPort_ReadData(uint8 id, uint16 *value);

/* Async result collection */
Std_ReturnType Rte_Result_DiagPort_ReadData(uint16 *value);

/* Server runnable signature (implemented by the server SWC) */
Std_ReturnType ReadData_Server(uint8 id, uint16 *value);
```

### S/R vs C/S (very common question)

| Aspect | **Sender-Receiver** | **Client-Server** |
|--------|---------------------|-------------------|
| Nature | Data exchange | Operation/function call |
| Direction | One-way (push) | Request/response |
| Cardinality | 1:N broadcast | N:1 |
| Blocking | No (read/write) | Can be sync (blocking) or async |
| Analogy | Publishing a variable | Remote procedure call (RPC) |
| Trigger on receiver | DataReceivedEvent | OperationInvokedEvent |
| Use case | Speed, temperature, status signals | Read DTC, request calibration, start routine |

---

## 3.6 Runnables & RTE Events

A **runnable** is executed when its associated **RTE event** occurs. The RTE maps runnables → OS tasks.

| RTE Event | Triggers a runnable when... |
|-----------|-----------------------------|
| **TimingEvent** | A periodic time elapses (e.g., every 10 ms) |
| **DataReceivedEvent** | Data arrives on an S/R RPort |
| **DataSendCompletedEvent** | A send completes (ack) |
| **OperationInvokedEvent** | A C/S server operation is called |
| **ModeSwitchEvent** | A mode changes (entry/exit/on-transition) |
| **DataReceiveErrorEvent** | Reception error/timeout |
| **InitEvent** | Component initialization |
| **BackgroundEvent** | Idle/background processing |
| **ExternalTriggerOccurredEvent** | An external trigger fires |

```c
/* Example: a 10ms cyclic runnable */
FUNC(void, RTE_APPL_CODE) SpeedController_10ms(void)
{
    uint16 speed;
    if (Rte_Read_WheelSpeed_Value(&speed) == RTE_E_OK) {
        uint16 torque = ComputeTorque(speed);
        (void)Rte_Write_Torque_Value(torque);
    }
}
```

---

## 3.7 Data consistency — Exclusive Areas

When multiple runnables access shared data, the RTE provides **Exclusive Areas** to guarantee consistency (implemented via interrupt disable, OS resources, or cooperative scheduling).

```c
Rte_Enter_MyExclusiveArea();   /* critical section start */
shared_counter++;              /* protected access       */
Rte_Exit_MyExclusiveArea();    /* critical section end   */
```
Other consistency mechanisms: **implicit read/write** (coherent copies at runnable boundaries), and **IRV** (Inter-Runnable Variables) for data shared *within* one SWC.

---

## 3.8 RTE generation & communication flow

### The RTE generation phases
1. **RTE Contract Phase** — generates the **component header** (`Rte_<SWC>.h`) from the SWC's interface (ports) only. Lets suppliers develop SWC code **independently**, before ECU integration.
2. **RTE Generation Phase** — after ECU configuration/connections are known, generates the **full RTE** (`Rte.c`, `Rte.h`, task bodies, S/R buffers, C/S routing). Requires the complete ECU extract.

```
   [SWC description ARXML] ──► RTE Generator (Contract) ──► Rte_<SWC>.h  (APIs)
                                                             │
                     develop SWC C code against these APIs   │
                                                             ▼
   [ECU config + connections] ─► RTE Generator (Generation) ─► Rte.c/Rte.h
                                                             (full glue + tasks)
```

### How the RTE decides local vs remote
```
Rte_Write_X(...)  ──►  Is receiver on same ECU?
                        ├─ YES → copy to internal RTE buffer (or direct call)
                        └─ NO  → call Com_SendSignal() → bus → remote ECU's Com
                                  → Rte_Read on the other ECU
```
This is what makes an SWC **relocatable**: the SWC code (`Rte_Write_X`) never changes; only the generated RTE differs.

---

## 3.9 Practical example — two SWCs, S/R + C/S

**Scenario:** `SensorSWC` sends vehicle speed to `ControllerSWC` (S/R). `ControllerSWC` asks `DiagSWC` to read a stored value (C/S).

```
  ┌────────────┐  Speed (S/R)   ┌──────────────┐  ReadCal (C/S)  ┌──────────┐
  │ SensorSWC  │───────────────►│ ControllerSWC│────────────────►│ DiagSWC  │
  │ (PPort)    │                │ (RPort/RPort)│◄────────────────│ (PPort)  │
  └────────────┘                └──────────────┘   value         └──────────┘
```

```c
/* --- SensorSWC: 10ms runnable, sends speed --- */
void Sensor_10ms(void) {
    uint16 raw = ReadWheelSpeedHw();
    (void)Rte_Write_SpeedOut_Value(raw);          /* S/R send  */
}

/* --- ControllerSWC: 10ms runnable, reads speed, calls diag --- */
void Controller_10ms(void) {
    uint16 speed, calib;
    if (Rte_Read_SpeedIn_Value(&speed) == RTE_E_OK) {
        if (Rte_Call_Diag_ReadCal(CAL_ID_TORQUE, &calib) == RTE_E_OK) {   /* C/S */
            uint16 torque = (uint16)((speed * calib) >> 8);
            (void)Rte_Write_TorqueOut_Value(torque);
        }
    }
}

/* --- DiagSWC: server runnable for ReadCal --- */
Std_ReturnType DiagSWC_ReadCal(uint8 id, uint16 *value) {
    *value = CalibrationTable[id];
    return RTE_E_OK;
}
```

---

## 3.10 RTE ↔ BSW: Service Ports

SWCs access **BSW services** (NvM, Dcm, Dem, EcuM modes) through the RTE using **Service Ports** and **Ports to standardized AUTOSAR Service Interfaces**.

Examples:
```c
Rte_Call_NvMService_WriteBlock(BlockId, &data);   /* store to NV memory */
Rte_Call_DemEvent_SetEventStatus(EventId, DEM_EVENT_STATUS_FAILED); /* report fault */
```
The BSW side is scheduled by the **SchM (Schedule Manager)**, which is the RTE's counterpart for BSW modules.

---

## 3.11 Interview questions from this chapter
1. What is the RTE and how is it related to the VFB?
2. Difference between Sender-Receiver and Client-Server communication?
3. Explain explicit vs implicit S/R access.
4. What are the RTE generation phases (contract vs generation)?
5. What is a runnable and what RTE events can trigger it?
6. How does the RTE ensure data consistency? What is an exclusive area?
7. How does an SWC access BSW services?
8. What makes an SWC relocatable?

*(Answers in [Chapter 9](Chapter_09_Answers.md).)*

---

> [◀ Chapter 2](Chapter_02_Classic_Architecture.md) | [Back to README](README.md) | [Next ▶ Chapter 4 — BSW Modules](Chapter_04_BSW_Modules.md)
