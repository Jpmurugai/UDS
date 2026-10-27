# CAN FD and UDS: Beginner-to-Practical Syllabus

For topic-by-topic explanations, worked examples, and exercises, read [CAN-FD-UDS-Lessons.md](CAN-FD-UDS-Lessons.md) alongside this syllabus.

## 1. Who This Course Is For

Recent college graduates starting in automotive embedded software, ECU diagnostics, integration, or testing. No automotive experience is required.

- Duration: 12 weeks, 8 hours per week, 96 hours total.
- Weekly pattern: 2 hours theory, 2 hours guided practice, 2 hours independent practice, and 2 hours testing and documentation.
- Starting knowledge: binary and hexadecimal numbers, basic electronics, and basic programming in C or Python. Use Week 1 to close gaps.
- Outcome: explain CAN FD, capture and interpret traffic, use ISO-TP, implement a small simulated UDS client/server, and demonstrate repeatable diagnostic tests.
- Scope: educational simulation and an optional isolated hardware bench, not production ECU certification or vehicle modification.

## 2. Understand the Layers First

| Layer | Responsibility | Example |
| --- | --- | --- |
| Application | Diagnostic meaning and service behavior | UDS ReadDataByIdentifier, service 0x22 |
| Transport | Carry complete diagnostic messages, including messages too long for one frame | ISO-TP segmentation and flow control |
| Data link | Frame delivery, arbitration, error detection | Classical CAN or CAN FD |
| Physical | Electrical signaling and wiring | CAN_H, CAN_L, transceivers, termination |

UDS is not a replacement for CAN FD. UDS can use ISO-TP over CAN or CAN FD; UDS can also use other transports, such as DoIP, outside this course.

## 3. Tools and Equipment

### Required: Simulation Path

- A PC with Git and a text editor.
- Recommended free environment: Ubuntu under WSL2 on Windows, or a Linux VM.
- Linux SocketCAN, a kernel supporting virtual CAN (`vcan`), and `can-utils`.
- A CAN FD-capable ISO-TP transport and UDS library or diagnostic simulator. Python options include `python-can`, `can-isotp`, and `udsoncan`.
- Read the documentation for the installed library versions; use library examples before writing application code. A client library alone does not provide an ECU server.
- Use a small simulator with explicit handlers for the selected UDS services, or an instructor-provided ECU simulator. Do not implement ISO-TP yourself.

WSL2 kernel features vary. If `vcan` or a selected transport backend is unavailable, use a compatible Linux VM or a documented in-process virtual CAN backend. Do not assume a `python-can` virtual bus is the same interface as SocketCAN `vcan0`.

### Optional: Physical Bench Path

- Two CAN FD-capable nodes: for example, a PC adapter and a development board, or two adapters.
- CAN FD-capable controllers AND suitable CAN FD transceivers.
- Documented drivers and CAN FD support for the chosen operating system.
- Twisted-pair cable, appropriate power supply, and termination at both ends of a linear bus.
- A multimeter; an oscilloscope is useful but optional.

A common MCP2515-based module supports Classical CAN, not CAN FD. Verify hardware capability before purchasing.

### Safety Rules

1. Use only a simulator or an isolated, authorized bench.
2. Do not connect exercises to a road vehicle or safety-critical network.
3. Power down before rewiring or measuring termination resistance.
4. Follow board pinouts, voltage limits, grounding, and isolation instructions.
5. Practice resets, writes, security access, and downloads only against your own simulator or an explicitly approved training ECU.
6. Never bypass production security, use real credentials, or flash unapproved firmware.

## 4. Weekly Learning Roadmap

Complete the weeks in order. Each lab depends on the previous lab's saved configuration and results.

### Week 1: Electronics, Numbers, and Environment

**Theory**

- Voltage, current, resistance, digital signals, and common ground.
- Bits, bytes, hexadecimal notation, masks, and endianness.
- Microcontroller, ECU, sensor, actuator, controller, and transceiver roles.
- Difference between a message identifier and a device address.

**Lab 1: Prepare a Reproducible Workspace**

1. Create a learning folder with notes, captures, configuration, and test results.
2. Record the OS, kernel, installed tool versions, and selected simulation backend.
3. Install `can-utils` and check that `candump` and `cansend` are available.
4. Convert 0x10, 0x22, 0x7F, and 0x7E0 into decimal and binary.
5. Decode bytes `12 34` as both big-endian and little-endian unsigned values.
6. Draw a PC-to-ECU diagram showing all four communication layers.

**Expected result / pass:** tools are available, conversions are correct, and the learner can identify which component owns each layer. Save a setup checklist.

### Week 2: Classical CAN Fundamentals

**Theory**

- Broadcast communication and 11-bit versus 29-bit identifiers.
- Dominant and recessive bits; nondestructive bitwise arbitration.
- Lower numeric identifiers win arbitration when comparing data frames of the same identifier format; mixed standard/extended cases also depend on arbitration fields.
- Classical CAN frame fields, up to 8 data bytes, ACK, CRC, and bit stuffing.
- An ACK indicates link-layer reception by another node, not application acceptance.

**Lab 2: Send and Capture Classical CAN Frames**

1. Create and bring up a virtual CAN interface using the setup below.
2. Start a capture in a second terminal.
3. Send identifier 0x123 with bytes `11 22 33 44`.
4. Repeat with a different identifier and payload.
5. Explain the timestamp, interface, identifier, and payload in each captured line.
6. Draw an arbitration example for identifiers 0x100 and 0x200. Do not claim that virtual CAN demonstrates electrical arbitration.

```bash
# Ubuntu/Linux commands, including inside a compatible WSL2 installation.
sudo apt update
sudo apt install can-utils
sudo modprobe vcan
sudo ip link add dev vcan0 type vcan
sudo ip link set dev vcan0 mtu 72
sudo ip link set dev vcan0 up

# Terminal A: leave running while sending from Terminal B.
candump -x vcan0

# Terminal B.
cansend vcan0 123#11223344
```

Run interface creation once; an existing interface does not need to be recreated. MTU 72 allows CAN FD frames on the virtual interface. Stop captures with Ctrl+C.

**Expected result / pass:** both frames appear with the exact IDs and bytes sent. Save a capture and annotated frame.

### Week 3: CAN FD Frames and Bit Timing

**Theory**

- Why CAN FD exists: larger payloads and an optional faster data phase.
- Nominal/arbitration bitrate versus data-phase bitrate.
- FDF, BRS, ESI, and changes to CRC and stuffing compared with Classical CAN.
- CAN FD has no remote frames.
- DLC is an encoded length, not always the byte count.
- Valid CAN FD payload lengths: 0 through 8, then 12, 16, 20, 24, 32, 48, and 64 bytes.
- Classical-only nodes must not be assumed compatible with FD traffic on an active mixed bus.

**Lab 3: Capture a CAN FD Frame**

1. Keep the Week 2 capture running or restart it.
2. Send the 12-byte FD frame below with BRS requested.
3. Compare it with the four-byte Classical CAN frame.
4. Build a DLC-to-length lookup table for DLC values 0 through 15.
5. Explain why 9 application bytes require an allowed larger frame length and padding when sent as one FD frame.
6. Record a proposed physical configuration, such as 500 kbit/s nominal and 2 Mbit/s data, subject to hardware and wiring capability.

```bash
cansend vcan0 123##100112233445566778899AABB
```

In `can-utils` syntax, `##` selects CAN FD and the following `1` is the flags nibble requesting BRS; the payload starts with `00`. Virtual CAN does not configure or measure physical bitrates.

**Expected result / pass:** capture shows an FD frame with 12 payload bytes and BRS metadata. Explain the flags and the difference between DLC and length.

### Week 4: Physical Network and Faults

**Theory**

- Differential signaling, linear topology, short stubs, and termination.
- Two 120-ohm end terminators typically measure about 60 ohms in parallel across an unpowered, otherwise appropriate bus.
- Oscillator tolerance, propagation delay, sample point, and bitrate agreement.
- Error-active, error-passive, bus-off, and controlled recovery.

**Lab 4: Verify a Bench or Analyze a Wiring Model**

1. Draw two nodes, both transceivers, power connections, and the two terminators.
2. For hardware, inspect pinouts and measure resistance with power off; record the actual value and explain deviations.
3. Configure both nodes with matching nominal/data timing and FD mode using their driver documentation.
4. Send a low-rate counter frame and verify reception on the other node.
5. On an authorized bench, introduce one reversible fault, such as mismatched bitrate, and record observations before restoring the configuration.
6. Without hardware, analyze missing-termination and bitrate-mismatch scenarios from supplied traces or documentation. Mark these as analysis, not measured results.

**Expected result / pass:** a wiring checklist plus a fault/result table. Virtual CAN evidence must not be presented as proof of electrical behavior or bus-off handling.

### Week 5: Application Signals and Trace Analysis

**Theory**

- CAN identifiers do not define payload meaning; an application specification or DBC does.
- Signal start bits, lengths, signedness, scaling, offsets, and units.
- Periodic versus event-driven traffic, cycle time, and timeout detection.
- Distinguish application payload throughput from actual bus load, which includes protocol overhead.

**Lab 5: Encode and Decode a Sensor Message**

1. Define a training frame ID 0x321 with a little-endian unsigned 16-bit temperature in bytes 0 and 1, using 0.1 degrees C per count and a -40 degrees C offset.
2. Use byte 2 as a rolling counter and byte 3 as a status field. Define reserved bytes and any FD padding explicitly.
3. Encode 25.0 degrees C: raw value 650, hexadecimal 0x028A, transmitted bytes `8A 02`.
4. Transmit the frame periodically at a documented interval, such as 100 ms.
5. Capture at least 20 frames and decode temperatures and counters.
6. Stop transmission and have the receiver report a timeout after a documented threshold, such as 500 ms.

**Expected result / pass:** decoded value is 25.0 degrees C, counters advance as specified, and the receiver detects stopped traffic. Save the signal specification and trace.

### Week 6: ISO-TP Transport

**Theory**

- ISO 15765-2 purpose and its relationship to UDS.
- Single Frame, First Frame, Consecutive Frame, and Flow Control.
- Sequence numbers, block size, STmin, transport timeouts, and addressing modes.
- CAN FD changes frame capacity and some ISO-TP length encodings; an FD single-frame header must not be assumed identical to a short Classical CAN header.
- More than 8 application bytes does not automatically mean segmentation on CAN FD.

**Lab 6: Reassemble a Long Message**

1. Configure two ISO-TP endpoints using a maintained library example.
2. Use normal 11-bit physical addressing: tester transmits on 0x7E0 and receives on 0x7E8; ECU uses the opposite mapping.
3. Treat these IDs as training choices, not mandatory UDS IDs.
4. Confirm a short payload arrives unchanged.
5. Send a 100-byte payload, which requires segmentation even on CAN FD.
6. Capture the exchange and identify First Frame, Flow Control, and Consecutive Frames.
7. Change block size and STmin and compare the trace. Remove the receiver and confirm a bounded transport timeout.

**Expected result / pass:** all 100 bytes arrive in order, and missing flow control produces a reported timeout. Record addressing, FD settings, padding, and library versions.

### Week 7: UDS Requests, Responses, and Timing

**Theory**

- Tester/client versus ECU/server; service IDs and positive responses.
- Many positive response SIDs equal request SID + 0x40; interpret the complete service definition.
- Negative response form: `7F <request SID> <NRC>`.
- Physical versus functional addressing; use physical requests for these labs.
- P2/P2* timing, NRC 0x78 response pending, S3 session timeout, and transport versus application timeout.
- For services supporting it, suppress-positive-response is a subfunction bit, not a general request flag.

**Lab 7: First Diagnostic Conversation**

1. Start the tester and simulated ECU over the Week 6 transport.
2. Implement DiagnosticSessionControl 0x10 and TesterPresent 0x3E in the simulator.
3. Send `10 03` to request extended session; inspect a successful response starting with `50 03` and decode its timing parameters.
4. Send `3E 00` and expect `7E 00` from the training ECU.
5. Send an unsupported service such as `99`; configure the training ECU to return `7F 99 11`.
6. Capture one response-pending exchange followed by the final response, with bounded client waiting based on configured timing.

Examples here are UDS application bytes BEFORE ISO-TP encapsulation, not literal payloads to paste into `cansend`. Simulated behavior is explicitly configured; real ECU support and responses depend on its specification and state.

**Expected result / pass:** distinguish positive response, negative response, response pending, and no response using saved traces.

### Week 8: Read and Write Diagnostic Data

**Theory**

- ReadDataByIdentifier 0x22 and WriteDataByIdentifier 0x2E.
- DIDs, data lengths, byte order, scaling, and read/write permissions.
- Standardized DID meanings versus manufacturer-specific data definitions.
- Preconditions, session restrictions, and write validation.

**Lab 8: Read Identification and Safely Write Training Data**

1. Configure DID 0xF190 as a 17-character synthetic VIN, for example `TESTVIN0000000001`; it is not a real vehicle identity.
2. Send `22 F1 90`; verify `62 F1 90` followed by the configured ASCII bytes.
3. Define a manufacturer-specific training DID for a simulated setpoint. Record its ID, units, encoding, allowed range, and writable session.
4. Read the initial value, write a valid value with 0x2E, and verify the response and readback.
5. Attempt an unsupported DID and an out-of-range value; define and check the applicable negative responses.
6. Confirm rejected writes leave stored data unchanged.

**Expected result / pass:** correct VIN decoding, successful valid readback, and no state change after rejected writes. Include multi-frame behavior where the configured transport requires it.

### Week 9: Sessions, Access Control, and Routines

**Theory**

- Default, extended, and programming sessions; support is ECU-specific.
- SecurityAccess 0x27 seed/key flow as an access-control concept, including lockout and attempt limits.
- SecurityAccess is not encryption; production algorithms and credentials are outside this course.
- Awareness of Authentication 0x29, without implementing a production authentication system.
- RoutineControl 0x31 and ECUReset 0x11.

**Lab 9: Test Diagnostic State Transitions**

1. Define a state table: session, simulated authorization state, permitted services, and timeout behavior.
2. Add a harmless routine that checks a simulated sensor and returns a result.
3. Confirm the routine is denied before its required session/access conditions are met.
4. Use a documented toy seed/key fixture only in the simulator; verify success, failed-key handling, and a configured attempt limit.
5. Run the routine and request results; test an invalid routine ID and invalid ordering.
6. Stop TesterPresent and confirm the configured S3 timeout returns the ECU to default session.
7. Simulate an ECU reset and verify the documented session/access-state reset behavior.

**Expected result / pass:** every transition matches the state table, including negative paths. Keep production credentials and algorithms out of the project.

### Week 10: Diagnostic Trouble Codes

**Theory**

- ReadDTCInformation 0x19 and ClearDiagnosticInformation 0x14.
- DTC format, status bits, availability masks, and supported subfunctions.
- Fault detection, debounce, operation cycles, snapshots, and extended data.
- Clearing stored information does not repair an active physical fault.

**Lab 10: Inject, Read, and Clear a Simulated Fault**

1. Define one synthetic sensor fault, a documented DTC encoding, supported status bits, and a simple debounce rule.
2. Confirm no matching DTC is initially reported.
3. Inject the simulated fault until the debounce threshold is reached.
4. Use a supported 0x19 subfunction, such as reportDTCByStatusMask 0x02, and decode the returned records.
5. Remove the fault and observe the configured status transitions.
6. Clear the supported DTC group with 0x14 and re-read it.
7. Reinject the fault and show that it can be detected again.

**Expected result / pass:** saved evidence shows detection, status interpretation, clearing, and redetection. Document the simplified simulator lifecycle rather than claiming production-complete DTC behavior.

### Week 11: Download Workflow and Robustness

**Theory**

- RequestDownload 0x34, TransferData 0x36, and RequestTransferExit 0x37.
- Address/length encoding, negotiated maximum block length, and block sequence counters.
- Transfer completion versus image validation, activation, authenticity, and rollback.
- NRCs such as incorrectMessageLengthOrInvalidFormat 0x13, conditionsNotCorrect 0x22, requestOutOfRange 0x31, and wrongBlockSequenceCounter 0x73.

**Lab 11: Transfer a Dummy Image to Simulator Memory**

1. Use only a simulator with an in-memory buffer; do not write executable flash.
2. Define supported session/access conditions, memory range, maximum size, and transfer format.
3. Request a download for a small dummy byte array and parse the negotiated block-length limit, accounting for service overhead.
4. Transfer blocks in order and verify each response counter.
5. Finish with 0x37 and compare the received bytes with the original array.
6. Test an incorrect sequence counter, oversized transfer, and interrupted transfer. Define whether each case is rejected or safely restarted.
7. Explain which production bootloader concerns this exercise does not cover.

**Expected result / pass:** byte-for-byte equality for the valid transfer, bounded failure handling for invalid transfers, and no activation or execution of downloaded data.

### Week 12: Integration and Final Demonstration

**Theory**

- Requirements-to-test traceability, repeatability, and evidence quality.
- Distinguishing physical, CAN, transport, UDS timing, and application-state failures.
- Simulation limitations and a sensible next step toward a supervised hardware project.

**Lab 12: Demonstrate a Mini Diagnostic ECU**

1. Start from a documented clean configuration and launch the tester and simulator.
2. Show periodic CAN FD sensor traffic and decoded values.
3. Read the synthetic VIN and setpoint, then perform an allowed setpoint write.
4. Demonstrate session entry, TesterPresent, access restrictions, and the harmless routine.
5. Inject, read, and clear the synthetic DTC.
6. Complete the dummy download, then demonstrate at least three controlled failures: unsupported DID, incorrect transfer counter, and missing response.
7. Re-run the tests and explain one trace from application bytes down to CAN frames.

**Expected result / pass:** a repeatable demonstration, matching request/response evidence, and a test report that clearly separates passed, failed, and not-run cases.

## 5. Assessment and Completion Criteria

| Area | Weight | Evidence |
| --- | --- | --- |
| Theory and layer understanding | 20% | Short quizzes, frame decoding, timing explanations |
| CAN FD and ISO-TP labs | 25% | Captures, configuration, reassembly and timeout tests |
| UDS service behavior | 30% | Positive, negative, state-dependent, and timing tests |
| Final integration | 15% | Reproducible end-to-end demo |
| Documentation and engineering practice | 10% | Setup notes, traceable tests, limitations, Git history |

Suggested completion threshold: 70% overall, all safety rules followed, and a successful final demonstration. Hardware outcomes may be marked not run when completing the simulation path, but must not be claimed as verified.

For every lab, record:

| Test ID | Requirement | Setup and stimulus | Expected result | Actual result | Evidence | Status |
| --- | --- | --- | --- | --- | --- | --- |
| UDS-READ-01 | Read synthetic VIN | Physical request `22 F1 90` in supported session | `62 F1 90` plus configured 17 ASCII bytes | Fill after execution | Capture filename and timestamp | Not run |

Final submission checklist:

- [ ] Tool versions and repeatable startup instructions.
- [ ] Layer diagram, addressing configuration, and optional wiring diagram.
- [ ] CAN signal definitions and CAN FD frame captures.
- [ ] ISO-TP segmentation and timeout evidence.
- [ ] Supported UDS services, DID definitions, and state/access table.
- [ ] Positive and negative test results, including timing behavior.
- [ ] DTC and dummy-download evidence.
- [ ] Known limitations and a short demonstration recording or live demo.

## 6. Quick Revision Questions

1. What changes between Classical CAN and CAN FD?
2. Why can a CAN FD DLC of 9 represent 12 payload bytes?
3. What does BRS change, and what does it not change?
4. Why does a CAN ACK not prove a UDS request succeeded?
5. Which layer segments a 100-byte diagnostic payload?
6. How do you distinguish `62 F1 90 ...` from `7F 22 31`?
7. What is the difference between P2, P2*, S3, and an ISO-TP timeout?
8. Why can the same UDS request succeed in one session and fail in another?
9. Why might a cleared DTC appear again?
10. Which claims require physical hardware rather than virtual CAN?

## 7. Reference Material

- Detailed ISO/SAE scope, part numbers, reading order, and official catalogue links: [CAN-FD-UDS-References.md](CAN-FD-UDS-References.md).
- Next step after simulation, including a budget parts list and open-source options: [CAN-FD-UDS-Hardware-Guide.md](CAN-FD-UDS-Hardware-Guide.md).
- Additional lessons: Lesson 21 compares UDS and OBD; Lesson 22 introduces hardware bring-up in [CAN-FD-UDS-Lessons.md](CAN-FD-UDS-Lessons.md).
- ISO 11898 family: CAN data link and physical-layer specifications. Select an edition covering CAN FD for this course.
- ISO 15765-2: diagnostic transport and network-layer services, including CAN FD support in applicable editions.
- ISO 14229-1: UDS application-layer services; ISO 14229-2: session-layer services; ISO 14229-3: UDS on CAN implementation.
- Linux SocketCAN and ISO-TP documentation: <https://docs.kernel.org/networking/can.html> and <https://docs.kernel.org/networking/iso15765-2.html>.
- `can-utils` examples: <https://github.com/linux-can/can-utils>.
- Library documentation: <https://python-can.readthedocs.io/>, <https://can-isotp.readthedocs.io/>, and <https://udsoncan.readthedocs.io/>.
- Controller, transceiver, adapter, and driver manuals for the exact bench hardware.

Use the applicable standards editions and ECU specification when details differ. This syllabus and its training examples are not substitutes for normative requirements.