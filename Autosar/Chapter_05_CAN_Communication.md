# Chapter 5 — AUTOSAR CAN Communication (Full Flow)

> [◀ Chapter 4](Chapter_04_BSW_Modules.md) | [Back to README](README.md) | [Next ▶ Chapter 6 — Configuration & Tools](Chapter_06_Configuration_and_Tools.md)

This is the **single most asked** interview topic. Master the **Tx and Rx flows** end-to-end.

---

## 5.1 The Communication Stack (bird's-eye)

```
        APPLICATION (SWC)
             ▲   │
   Rte_Read  │   │  Rte_Write
             │   ▼
   ┌─────────────────────┐
   │        RTE          │   ← works with SIGNALS
   └─────────────────────┘
             ▲   │
 Com_RxIndic │   │ Com_SendSignal
             │   ▼
   ┌─────────────────────┐
   │        Com          │   ← SIGNAL ⇄ I-PDU (pack/unpack)
   └─────────────────────┘
             ▲   │
             │   ▼
   ┌─────────────────────┐
   │       PduR          │   ← routes I-PDUs (config table)
   └─────────────────────┘
             ▲   │              ┌────────────┐
             │   ▼              │   CanTp    │ (only for large/diagnostic PDUs)
   ┌─────────────────────┐◄────►└────────────┘
   │       CanIf         │   ← PDU ⇄ HW object (HTH/HRH)
   └─────────────────────┘
             ▲   │
   RxIndic   │   │ Can_Write
   TxConfirm │   ▼
   ┌─────────────────────┐
   │    Can (MCAL)       │   ← controller registers / mailboxes
   └─────────────────────┘
             ▲   │
             │   ▼
        CAN CONTROLLER  ──────►  CAN BUS
```

Additional helpers on the CAN channel: **CanSM** (communication state), **CanNm** (network management), **ComM** (communication manager), **IpduM** (I-PDU multiplexing).

---

## 5.2 Signal ↔ I-PDU ↔ PDU ↔ Frame relationship

This hierarchy is a **classic interview question**. Understand each term:

| Term | Meaning | Owner |
|------|---------|-------|
| **Signal** | The smallest piece of application data (e.g., `VehicleSpeed`, 16-bit). Has a bit position, length, endianness, factor/offset. | Com / RTE |
| **I-PDU** (Interaction Layer PDU) | A group of signals packed into one data unit (the payload). | Com |
| **PDU** (Protocol Data Unit) | Generic "unit of data passed between layers." Named by layer: I-PDU (Com), N-PDU (CanTp), L-PDU (CanIf/Can). | Each layer |
| **L-PDU** | The PDU at the CanIf/Can level (includes CAN ID + DLC + data). | CanIf/Can |
| **Frame** | The actual **CAN frame** on the wire (ID, control, data, CRC, ACK). | CAN controller |

```
  ┌ Signal: VehicleSpeed (16b) ┐  ┌ Signal: EngineTemp (8b) ┐  ┌ Flags (8b) ┐
  └────────────┬───────────────┘  └───────────┬─────────────┘  └─────┬──────┘
               └──────────────── packed by Com ┴───────────────────────┘
                                   ▼
                     ┌──── I-PDU (payload, e.g. 8 bytes) ────┐
                                   ▼  (PduR routes)
                     ┌──── L-PDU (CAN ID 0x120 + 8 bytes) ───┐
                                   ▼  (Can driver)
        ┌ SOF │ ID 0x120 │ Ctrl/DLC │ DATA (8B) │ CRC │ ACK │ EOF ┐  ← CAN FRAME on bus
```

> **PDU naming by layer:** SDU (Service Data Unit = payload) + PCI (Protocol Control Info = header) = PDU. Each layer adds/removes its own PCI.
> - **I-PDU** at Com, **N-PDU** at CanTp, **L-PDU** at CanIf/Can.

---

## 5.3 TRANSMISSION Flow (Tx) — step by step

**Scenario:** An SWC computes `VehicleSpeed` and it must go out on CAN ID `0x120`.

```
STEP  MODULE        ACTION / API
────  ────────────  ─────────────────────────────────────────────────────────
 1    SWC           Computes value; calls Rte_Write_SpeedPort_Value(200)
 2    RTE           Maps the S/R write to a Com signal; calls Com_SendSignal(SigId,&200)
 3    Com           Stores signal in its I-PDU buffer (packs bits, applies endianness).
                    On Tx trigger (cyclic timer OR on-change), calls
                    PduR_ComTransmit(IpduId, PduInfo)  → i.e. Com_TriggerIPDUSend
 4    PduR          Looks up routing table; routes the I-PDU to CanIf:
                    CanIf_Transmit(TxPduId, PduInfo)
 5    CanIf         Maps TxPduId → HTH (Hardware Transmit Handle); if HW busy,
                    buffers L-PDU. Calls Can_Write(HTH, PduInfo)
 6    Can (MCAL)    Writes CAN ID + DLC + data into a hardware TX mailbox;
                    requests transmission.
 7    CAN Ctrl      Arbitrates on bus, transmits the FRAME.
 8    Can (MCAL)    On Tx-complete interrupt → CanIf_TxConfirmation(TxPduId)
 9    CanIf         → PduR_CanIfTxConfirmation → Com_TxConfirmation(IpduId)
10    Com           Marks I-PDU as sent; may notify SWC (Tx confirmation) via RTE.
```

**Compact chain:**
```
SWC → RTE → Com → PduR → CanIf → Can → CAN Controller → BUS
       (Rte_Write)(SendSignal)(Transmit)(Can_Write)(mailbox)
Confirmation flows back: Can → CanIf → PduR → Com → (RTE → SWC)
```

### Transmission modes (configured in Com)
| Mode | When I-PDU is sent |
|------|--------------------|
| **Periodic / Cyclic** | Every configured cycle time (e.g., 10 ms), regardless of change |
| **Direct / Event (n-times)** | Only when a signal changes (with optional repetitions) |
| **Mixed** | Cyclic **plus** immediate send on change |
| **None** | Never auto-sent (triggered externally) |

---

## 5.4 RECEPTION Flow (Rx) — step by step

**Scenario:** A frame with CAN ID `0x120` arrives; the SWC needs `VehicleSpeed`.

```
STEP  MODULE        ACTION / API
────  ────────────  ─────────────────────────────────────────────────────────
 1    CAN Ctrl      Receives FRAME, stores in HW receive object (mailbox), IRQ.
 2    Can (MCAL)    In Rx ISR / poll, reads mailbox; identifies HRH;
                    calls CanIf_RxIndication(Mailbox, &PduInfo)
 3    CanIf         Maps HRH + CAN ID → RxPduId; optional SW filtering & DLC check;
                    dispatches to upper layer: PduR_CanIfRxIndication(RxPduId, PduInfo)
 4    PduR          Routes I-PDU to Com: Com_RxIndication(IpduId, PduInfo)
                    (or to CanTp for segmented/diagnostic PDUs)
 5    Com           Unpacks signals from the I-PDU (bit extraction, endianness,
                    factor/offset). Updates signal buffers. Starts Rx deadline monitor.
 6    Com→RTE       Notifies RTE (Rx indication) or RTE reads on next cycle.
 7    RTE           SWC runnable triggered by DataReceivedEvent OR reads cyclically:
                    Rte_Read_SpeedPort_Value(&speed)
 8    SWC           Uses the value.
```

**Compact chain:**
```
BUS → CAN Controller → Can → CanIf → PduR → Com → RTE → SWC
     (mailbox)(RxIndication)(route)(unpack)(Rte_Read)
```

### Rx deadline monitoring
If an expected cyclic I-PDU doesn't arrive within its timeout, **Com** applies the **replacement (substitute) value** and can notify the SWC → the application knows the signal is stale/lost.

---

## 5.5 Diagnostic (segmented) flow — where CanTp fits

For **diagnostics (UDS)** the message can exceed one CAN frame, so **CanTp** is inserted:

```
TESTER ──► BUS ──► CAN Ctrl ──► Can ──► CanIf ──► CanTp ──► PduR ──► Dcm
                                              (reassemble SF/FF/CF)
Response: Dcm ──► PduR ──► CanTp ──► CanIf ──► Can ──► CAN Ctrl ──► BUS ──► TESTER
                        (segment into SF/FF/CF, honor FC/BS/STmin)
```
> Note: normal signal I-PDUs go **CanIf ↔ PduR ↔ Com**; diagnostic PDUs go **CanIf ↔ CanTp ↔ PduR ↔ Dcm**. PduR decides the path based on its routing config.

---

## 5.6 Hardware objects: HTH / HRH

| Term | Meaning |
|------|---------|
| **HTH** (Hardware Transmit Handle) | Reference to a hardware **transmit** mailbox/buffer used by `Can_Write` |
| **HRH** (Hardware Receive Handle) | Reference to a hardware **receive** mailbox/filter that received a frame |
| **L-PDU** | The CAN message (ID + DLC + data) handled at CanIf/Can level |

CanIf's core job: **map software PDU IDs ↔ hardware handles (HTH/HRH)**.

---

## 5.7 Supporting modules on a CAN channel

| Module | Role |
|--------|------|
| **ComM** (Communication Manager) | Coordinates communication requests from users; manages "full/silent/no communication" modes |
| **CanSM** (CAN State Manager) | Manages CAN network state (start/stop comm, bus-off recovery) per channel |
| **CanNm** (CAN Network Management) | Coordinated sleep/wakeup across the network (keep-awake, bus-sleep) |
| **CanTrcv** | CAN transceiver driver (wakeup, standby) |
| **IpduM** (I-PDU Multiplexer) | Multiple layouts (multiplexed signals) in one I-PDU |

```
   ComM ──requests──► CanSM ──controls──► CanIf ──► Can (Start/Stop/Sleep)
   CanNm ──coordinates network sleep/wakeup──► CanSM
```

---

## 5.8 Bus-off handling (common scenario question)

```
  Can detects bus-off ─► CanIf_ControllerBusOff() ─► CanSM
       │
       ▼
  CanSM starts bus-off recovery (T1/L1 delay, retries) ─► restart controller
       │
       ▼
  Notifies ComM/BswM; Dem may log a communication DTC
```
> **Interview answer:** *"On bus-off, the Can driver notifies CanIf → CanSM. CanSM runs the recovery algorithm (configurable delay & retry counts), restarts the controller, and reports status to ComM and Dem."*

---

## 5.9 Practical DaVinci configuration example (end-to-end)

**Goal:** Transmit signal `VehicleSpeed` inside I-PDU `EngineData` (CAN ID `0x120`, 8 bytes, cyclic 10 ms) on channel `CAN0`.

### Step-by-step (DaVinci Configurator / Developer)

1. **Import the communication matrix (DBC/ARXML/Fibex).**
   - The **DBC/communication cluster** defines messages (`EngineData` @ 0x120), signals (`VehicleSpeed`: start bit 0, length 16, Intel/little-endian, factor 0.01, offset 0).
   - Import creates the **System / ECU extract**.

2. **Map the ECU signals to the SWC ports.**
   - In **DaVinci Developer**, connect the SWC's `SpeedPort` (S/R signal) to the system signal `VehicleSpeed`.

3. **Configure Com** (in Configurator):
   ```
   ComSignal: VehicleSpeed
     - ComBitPosition = 0
     - ComBitSize = 16
     - ComSignalEndianness = LITTLE_ENDIAN
     - ComSignalType = UINT16
     - ComTransferProperty = TRIGGERED / PENDING
   ComIPdu: EngineData
     - ComIPduDirection = SEND
     - ComTxModeMode = PERIODIC
     - ComTxModeTimePeriod = 0.010   (10 ms)
     - contains ComSignal VehicleSpeed
   ```

4. **Configure PduR** — add a routing path:
   ```
   PduRRoutingPath:
     Source = Com  (EngineData I-PDU)
     Dest   = CanIf (EngineData Tx L-PDU)
   ```

5. **Configure CanIf** — Tx PDU + HTH:
   ```
   CanIfTxPduCfg: EngineData_Tx
     - CanIfTxPduId = <handle>
     - CanIfTxPduCanId = 0x120
     - CanIfTxPduDlc = 8
     - CanIfTxPduHthRef = CAN0_HTH_0
     - CanIfTxPduUserTxConfirmationName = PduR
   ```

6. **Configure Can (MCAL)** — controller + mailbox:
   ```
   CanController: CAN0
     - Baudrate = 500 kbps
     - CanHardwareObject (HTH) for Tx, (HRH) for Rx
   ```

7. **Configure OS/SchM** — ensure `Com_MainFunctionTx` runs (e.g., in a 5 ms task).

8. **Generate** — run **RTE + BSW generation**; compile & link.

9. **Verify** — on target/CANoe, confirm `0x120` appears every 10 ms with `VehicleSpeed` correctly scaled.

```
   [DBC import] → [signal↔port mapping] → [Com] → [PduR] → [CanIf] → [Can]
                                    → GENERATE (RTE+BSW) → BUILD → TEST on CANoe
```

> **Interview tip:** When asked "how did you add a new signal?", walk through exactly these steps: *import/extend comm matrix → map signal → Com signal+I-PDU → PduR route → CanIf Tx PDU/HTH → Can mailbox → generate → build → test on CANoe/CANalyzer.*

---

## 5.10 One-glance Tx vs Rx summary

| | **Transmit** | **Receive** |
|-|--------------|-------------|
| Start | `Rte_Write` → `Com_SendSignal` | Frame arrives → Can ISR |
| Com | packs signal → I-PDU | unpacks I-PDU → signals |
| PduR | Com → CanIf | CanIf → Com |
| CanIf | maps PDU → **HTH**, `Can_Write` | maps **HRH** → PDU, `RxIndication` |
| Can | writes mailbox | reads mailbox |
| Confirm | `TxConfirmation` back up | `Rte_Read` by SWC |

---

## 5.11 Interview questions from this chapter
1. Explain the complete CAN transmission flow with API names.
2. Explain the reception flow from bus to SWC.
3. What is the relationship between signal, I-PDU, PDU, and frame?
4. Where does CanTp fit, and why is it needed for diagnostics?
5. What are HTH and HRH?
6. What are the Com transmission modes?
7. How is bus-off handled in the AUTOSAR CAN stack?
8. Walk me through adding a new CAN signal in DaVinci.
9. What is Rx deadline monitoring?
10. Difference between ComM, CanSM, and CanNm?

*(Answers in [Chapter 9](Chapter_09_Answers.md).)*

---

> [◀ Chapter 4](Chapter_04_BSW_Modules.md) | [Back to README](README.md) | [Next ▶ Chapter 6 — Configuration & Tools](Chapter_06_Configuration_and_Tools.md)
