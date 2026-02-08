# Project Requirements Specification

## 1. Stakeholder Requirements
Stakeholders require a system that:
- **Responds** to external events within bounded, deterministic timeframes.
- **Detects and handles** fault conditions (data loss, corruption) safely.
- **Operates reliably** over extended periods in an industrial environment.
- **Minimizes Operator Fatigue** by prioritizing critical alerts over minor warnings.

---

## 2. Requirements Traceability Matrix

| ID | Requirement (Single Declarative Sentence) | Classification | Rationale | Test Intent / Testability |
|:---|:---|:---|:---|:---|
| **REQ-01** | The system shall update its internal state machine within **50ms** of receiving a valid UART frame. | **Non-Functional** | Latency leads to delayed safety interventions in high-speed processes. | Inject a timestamped UART frame; verify internal state variable updates within 50ms. |
| **REQ-02** | The system shall classify Sensor S1 as **"Critical"** if the reported value exceeds **200°C**. | **Functional** | Identifies the absolute unsafe operating limit for the primary reactor. | Inject UART frame (ID=S1, VALUE=201.0); verify state transitions to "Critical". |
| **REQ-03** | The system shall trigger a **"Thermal Runaway"** alarm (P1) if S1 > **180°C** AND S4 < **10 kg/s**. | **Safety** | Combined "Caution" states can represent a catastrophic failure mode requiring intervention. | Inject S1=181 and S4=9; verify Alarm A1 (Priority 1) triggers within 20ms. |
| **REQ-04** | The system shall activate a Priority-1 alarm signal within **100ms** of any sensor entering a Critical state. | **Safety** | Critical hazards require near-real-time notification to prevent equipment failure. | Measure delta between UART EOL character and the P1 alarm pin/log activation. |
| **REQ-05** | The system shall apply a **100ms** temporal debounce period to all sensor state transitions. | **Functional** | Prevents "alarm chattering" and fills logs with redundant data during signal noise. | Input 10Hz oscillating signal; verify alarm state remains stable for <100ms pulses. |
| **REQ-06** | The system shall suppress P2/P3 alarms from the display if a P1 alarm is currently active. | **Functional** | Prevents "Alarm Flooding"; ensures the operator focuses only on the most critical hazard. | Activate 10+ P3 alarms, then 1 P1; verify only the P1 alarm is visible to the operator. |
| **REQ-07** | The system shall transition a sensor to **"Critical"** if no valid data is received for **>2000ms**. | **Safety** | Silence is a failure mode; the system must assume the worst-case scenario if data is lost. | Stop the UART stream for S1; verify escalation to "Critical" after the 2-second timeout. |
| **REQ-08** | The system shall discard frames with checksum errors and log a **"Caution"** fault after 2 consecutive failures. | **Non-Functional** | Corrupted data (transient noise) must not trigger false alarms or mask real hazards. | Inject 2 frames with bad checksums; verify data is ignored and "Comm Fault" is logged. |
| **REQ-09** | An alarm shall latch in **"Active-Unacknowledged"** until a manual reset is received. | **Functional** | Operators must be aware that a transient excursion occurred, even if it resolved itself. | Trigger S1 > 200, then return S1 to 150; verify alarm remains "Active" until reset. |
| **REQ-10** | The system shall maintain the last known valid state for a sensor during a single frame corruption event. | **Non-Functional** | Prevents system flickering or state resets due to momentary communication glitches. | Send valid S3 frame, then one corrupted frame; verify S3 state remains unchanged. |

---

## 3. Design Constraints & Verification
- **Timing:** All response times are deterministic (Worst-Case Execution Time).
- **Fail-Safe:** Any loss of monitoring capability results in an immediate "Critical" system state.
- **Maintainability:** The codebase must be modular to allow for independent testing of the logic and IO drivers.
