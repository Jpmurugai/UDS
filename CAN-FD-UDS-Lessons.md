# CAN, CAN FD, and UDS: Step-by-Step Lessons

## How to Study

This guide explains the topics in [CAN-FD-UDS-Syllabus.md](CAN-FD-UDS-Syllabus.md). Here, "CAN class" means **Classical CAN**, also commonly called Classic CAN.

Supporting documents: [CAN-FD-UDS-References.md](CAN-FD-UDS-References.md) maps the ISO/SAE standards; [CAN-FD-UDS-Hardware-Guide.md](CAN-FD-UDS-Hardware-Guide.md) covers low-cost hardware and open-source software.

Study the lessons in order. For each lesson, read the explanation, work through the example, perform the exercise, and answer the check question before continuing.

| Syllabus week | Lessons |
| --- | --- |
| 1 | 1-2: numbers and ECU components |
| 2 | 3-5: Classical CAN, frames, and arbitration |
| 3 | 6-8: CAN FD, comparison, and DLC |
| 4 | 9: wiring, timing, and faults |
| 5 | 10: application signals |
| 6 | 11: ISO-TP |
| 7 | 12-14: UDS basics, responses, and timing |
| 8 | 15: reading and writing DIDs |
| 9 | 16-17: sessions, access, routines, and reset |
| 10 | 18: DTCs |
| 11 | 19: simulated download |
| 12 | 20: integration and troubleshooting |
| Extension | 21: UDS versus OBD and OBD-on-UDS |
| Hardware extension | 22: move from simulation to a real CAN FD bench |

**Lab boundary:** use virtual CAN or your own isolated training bench. Do not send these examples to a road vehicle. The diagnostic IDs, supported services, and state rules below are training choices, not a universal ECU specification.

## Lesson 1: Bits, Bytes, and Hexadecimal

**What is it?** A bit is 0 or 1. A byte contains 8 bits. Hexadecimal is a compact way to write binary values: one hex digit represents 4 bits, so two hex digits represent one byte.

**Why do we need it?** CAN tools and diagnostic specifications normally display IDs and data in hexadecimal. You must recognize the difference between a number, its representation, and text characters.

**How do we use it?** Read each pair of hex digits as one byte. `0x` is a notation prefix; it is not transmitted.

| Written value | Meaning |
| --- | --- |
| `0x10` | Decimal 16, binary `00010000` |
| `0x22` | Decimal 34; also the UDS ReadDataByIdentifier service ID when used in that field |
| `0x7F` | Decimal 127; also the UDS negative-response SID when used in that field |
| `12 34` | Two bytes; unsigned big-endian value 4660, little-endian value 13330 |
| ASCII `A` | Encoded as byte `41`, not byte `0A` |

**Practice, step by step**

1. Convert hex `2A` to decimal: 2 times 16 plus 10 equals 42.
2. Count the bytes in `22 F1 90`: there are three.
3. Decode `01 00` as unsigned big-endian and little-endian values.

**Check:** the last answers are 256 and 1. Byte order changes the interpretation, not the transmitted byte sequence.

## Lesson 2: ECU, Controller, and Transceiver

**What is it?** An Electronic Control Unit (ECU) is an embedded computer that reads inputs, makes decisions, and controls outputs. A CAN controller handles protocol frames. A transceiver converts logic-level controller signals into electrical bus signals and back.

**Why do we need it?** Software cannot drive CAN_H and CAN_L directly through ordinary microcontroller GPIO pins. Knowing the components helps you select hardware and find faults.

**How do we use it?** Follow this signal path:

```text
Application -> CAN controller -> CAN transceiver -> CAN_H / CAN_L
                                                   |
Other application <- CAN controller <- transceiver <-+
```

Some microcontrollers contain a CAN controller, but still need an external transceiver. A PC adapter normally packages controller and transceiver functions together.

**Practice, step by step**

1. Choose a development board or PC adapter from its documentation.
2. Identify its controller, transceiver, connector pinout, and supported frame formats.
3. Confirm both controller and transceiver are suitable for the intended CAN FD data rate.

**Check:** a Classical CAN-only controller, such as MCP2515, does not become CAN FD-capable by adding an FD transceiver.

## Lesson 3: What Is Classical CAN?

**What is it?** Controller Area Network (CAN) is a message-based communication system. On a shared bus, a transmitted frame is available to all connected nodes; receivers decide which messages they need.

**Why do we need it?** Several ECUs can share measurements and commands over one bus instead of needing a separate signal wire for every value. CAN provides arbitration, error detection, and fault confinement.

**How do we use it?** Define an identifier and payload layout for each message. For example, a sensor ECU periodically sends a temperature message; a display ECU receives and decodes it.

The ID usually identifies a message and its priority, not a destination device. Application protocols may establish additional addressing conventions.

**Practice, step by step**

1. Draw three nodes: temperature sensor, display, and logger.
2. Define ID `0x321` as the temperature message.
3. Show the sensor broadcasting once, with both display and logger receiving.
4. Explain why the sensor does not need to transmit a separate copy for each receiver.

**Check:** CAN itself does not define whether bytes mean temperature, speed, or voltage. That meaning comes from the application specification.

## Lesson 4: Classical CAN Frames and Your First Capture

**What is it?** A Classical CAN data frame contains protocol fields including an identifier, control information, zero to eight data bytes, CRC, acknowledgment, and end-of-frame bits. Identifiers can be 11-bit or 29-bit.

**Why do we need it?** A frame is the unit the CAN controller sends. Understanding its fields lets you distinguish the message ID, payload length, payload data, and link-level checks.

**How do we use it?** Start with virtual CAN. The following commands run in Ubuntu/Linux, including compatible WSL2 installations, not directly in PowerShell. Use a Linux VM if your WSL2 kernel lacks `vcan` support.

**Practice, step by step**

1. Install the tools and create the interface once:

```bash
sudo apt update
sudo apt install can-utils
sudo modprobe vcan
sudo ip link add dev vcan0 type vcan
sudo ip link set dev vcan0 mtu 72
sudo ip link set dev vcan0 up
```

2. In terminal A, start a capture:

```bash
candump -x vcan0
```

3. In terminal B, send a Classical CAN frame:

```bash
cansend vcan0 123#11223344
```

4. Find ID `123` and the four bytes `11 22 33 44` in the capture.
5. Change the last byte to `55` and send again.

An existing interface does not need to be created again. MTU 72 permits FD frames as well as Classical frames on this virtual interface. Stop the capture with Ctrl+C.

**Check:** ID `0x123` is not part of the four-byte payload. The `cansend` text syntax is not the literal bit sequence placed on a physical bus.

## Lesson 5: Arbitration, ACK, and Errors

**What is it?** Arbitration selects which competing frame continues without corrupting the winning frame. A dominant 0 overrides a recessive 1 on the bus. A sender transmitting recessive while observing dominant during arbitration loses arbitration and retries later.

**Why do we need it?** Multiple nodes may start transmitting together. Priority-based arbitration makes this manageable without requiring a central bus master.

**How do we use it?** Assign identifiers with message priority in mind. For data frames of the same identifier format, a lower numeric ID wins arbitration. Standard versus extended frame comparisons also involve other arbitration bits.

Example: 11-bit IDs `0x100` and `0x200` begin with `00100000000` and `01000000000`. At the first differing bit, `0x100` transmits dominant 0 while `0x200` transmits recessive 1, so `0x100` continues.

ACK means at least one other node received a valid frame at the link layer; it does not mean the target application accepted a command. CRC detects transmission errors; it is not encryption or sender authentication.

**Practice, step by step**

1. Compare 11-bit data-frame IDs `0x080` and `0x120` in binary.
2. Mark the first bit where they differ and identify the winner.
3. Explain why a lower-priority message can be delayed by heavy higher-priority traffic.
4. Explain why seeing a successful CAN transmission does not prove a diagnostic operation succeeded.

**Check:** `0x080` wins. Virtual CAN does not demonstrate physical arbitration, ACK signaling, or electrical error handling.

## Lesson 6: What Is CAN FD, and Why Do We Need It?

**What is it?** CAN with Flexible Data-Rate (CAN FD) extends CAN with payloads up to 64 bytes and an optional faster bit rate during the data phase.

**Why do we need it?** Classical CAN's eight-byte limit means larger application messages require multiple frames. Larger FD payloads reduce per-frame overhead, and a faster data phase can increase throughput. Neither feature removes the need to analyze latency and bus load.

**How do we use it?** Select FD-capable hardware and drivers, configure the nominal and data-phase timing, enable FD frames, and optionally enable Bit Rate Switching (BRS).

Example configuration: 500 kbit/s nominal bitrate and 2 Mbit/s data bitrate. These are training values, not universal requirements. Arbitration uses nominal timing. With BRS enabled, the frame switches to data-phase timing at the defined point and returns to nominal timing before acknowledgment.

**Practice, step by step**

1. Keep the Lesson 4 capture running.
2. Send 12 bytes in an FD frame with BRS requested:

```bash
cansend vcan0 123##100112233445566778899AABB
```

3. Parse the command: `123` is the ID, `##` selects FD, the next `1` is the flags nibble requesting BRS, and the payload is `00 11 22 33 44 55 66 77 88 99 AA BB`.
4. Compare the captured frame format and length with Lesson 4.

**Check:** BRS is optional; an FD frame can use nominal timing throughout. Virtual CAN carries FD metadata but does not measure a real 2 Mbit/s data phase.

## Lesson 7: Classical CAN Versus CAN FD

**What is the difference?** Both use CAN's message-based model and identifier arbitration, but FD changes the frame format, payload capacity, and optionally the data-phase speed.

| Feature | Classical CAN | CAN FD |
| --- | --- | --- |
| Maximum data payload | 8 bytes | 64 bytes |
| Identifier formats | 11-bit and 29-bit | 11-bit and 29-bit |
| Bitrate within a frame | One nominal bitrate | Nominal bitrate plus optional faster data phase |
| Common high-speed example | 500 kbit/s or 1 Mbit/s, subject to physical limits | 500 kbit/s nominal, 2 Mbit/s data, subject to hardware and bus design |
| Payload length choices | 0-8 bytes | 0-8, 12, 16, 20, 24, 32, 48, 64 bytes |
| Remote frames | Supported by the protocol | Not supported |
| CRC | 15-bit | 17-bit or 21-bit depending on payload length |
| Special FD fields | Not applicable | Includes FDF, BRS, and ESI |
| Typical choice | Small messages or existing Classical networks | Larger messages or higher-throughput networks designed for FD |

FDF identifies the FD format; BRS controls bitrate switching; ESI reports the transmitting node's error-state indication. FD-capable controllers can generally handle Classical frames too, but ordinary Classical-only nodes can disrupt FD traffic by treating it as erroneous. Do not assume safe coexistence without an explicitly supported network design.

**Why choose one?** Use Classical CAN when the existing network and bandwidth requirements suit it. Choose FD when the extra capacity justifies compatible hardware, timing configuration, and network validation. FD is not automatically needed for every small sensor.

**Practice, step by step**

1. Consider 32 application bytes: Classical CAN needs at least four eight-byte data frames, while FD can hold those bytes in one 32-byte frame before any higher-layer overhead.
2. Explain why that is not a guaranteed fourfold speed improvement: arbitration, overhead, stuffing, bitrates, and application/transport headers also matter.
3. Choose a format for a legacy eight-byte sensor network and for a new FD-capable diagnostic bench. Justify both decisions.

**Check:** UDS can operate over either Classical CAN or CAN FD. FD is a transport-capacity choice, not a diagnostic service.

## Lesson 8: DLC, Padding, and Payload Length

**What is it?** The Data Length Code (DLC) is a four-bit encoded field describing the frame's data length. In FD, values above 8 map to selected larger lengths.

**Why do we need it?** Confusing DLC with byte count causes incorrect buffers, parsing, and padding. Nine application bytes do not correspond to an FD DLC of 9 with nine wire data bytes.

| DLC value, decimal | CAN FD data bytes |
| --- | --- |
| 0-8 | Same as DLC |
| 9 | 12 |
| 10 | 16 |
| 11 | 20 |
| 12 | 24 |
| 13 | 32 |
| 14 | 48 |
| 15 | 64 |

**How do we use it?** Pick a supported frame length and define how the receiver distinguishes meaningful data from padding. Higher-layer protocols can carry their own application length.

**Practice, step by step**

1. Place nine application bytes into a 12-byte FD payload.
2. Define the last three bytes as zero padding in your training protocol.
3. Record that the wire payload length is 12 and the DLC is 9.
4. Repeat for 21 application bytes: use 24 wire bytes, three padding bytes, and DLC 12.

**Check:** DLC 15 means 64 data bytes in FD, not 15. A Classical CAN frame still cannot carry more than eight data bytes.

## Lesson 9: Wiring, Bit Timing, and Fault Confinement

**What is it?** High-speed CAN uses differential signaling on CAN_H and CAN_L. Controllers must agree on compatible timing, and the physical network must support the chosen rates.

**Why do we need it?** Correct application code cannot compensate for incorrect wiring, unsuitable termination, or mismatched timing.

**How do we use it?** Use a linear bus with termination at its two ends, short stubs, and appropriate grounding or isolation according to the hardware manuals. Nominal bitrate, data bitrate, sample points, and propagation constraints all matter. Cable length and topology can limit usable data rates.

```text
120-ohm termination ==== CAN_H / CAN_L bus ==== 120-ohm termination
                          |           |
                        Node A      Node B
```

Repeated communication errors affect error counters. A controller can become error-passive and eventually bus-off, where it stops participating until the appropriate recovery procedure. Losing arbitration is not a transmission error.

**Practice, step by step**

1. Power off the isolated bench and inspect connectors and pinouts.
2. Measure resistance across CAN_H and CAN_L only when safe for the hardware. Two 120-ohm terminators in parallel typically give about 60 ohms; other circuitry can affect the reading.
3. Configure both endpoints with compatible FD timing and transmit at a low message rate.
4. Observe reception and the adapter's error/status counters.
5. If authorized, introduce a reversible bitrate mismatch, record the result, and restore the configuration.

**Check:** on a simulation-only path, draw the circuit and predict the fault instead. Mark physical measurements and bus-off observations as not run.

## Lesson 10: Turn Payload Bytes into Useful Signals

**What is it?** A signal definition explains where a value is stored and how to interpret it: byte/bit position, length, signedness, endianness, scaling, offset, and units. A DBC file can describe these definitions for compatible tools.

**Why do we need it?** Bytes such as `8A 02` have no built-in temperature meaning. All participants must share the same interpretation.

**How do we use it?** Define this training message on ID `0x321`:

| Bytes | Meaning |
| --- | --- |
| 0-1 | Unsigned little-endian temperature; physical value = raw times 0.1 minus 40 degrees C |
| 2 | Counter incrementing modulo 256 |
| 3 | Status: 0 = valid, 1 = invalid |

For 25.0 degrees C, raw = (25 + 40) / 0.1 = 650 = `0x028A`. The payload with counter 0 and valid status is `8A 02 00 00`.

**Practice, step by step**

1. Send `cansend vcan0 321#8A020000`.
2. Decode the first two bytes and verify 25.0 degrees C.
3. Calculate the payload for 30.0 degrees C: raw 700 = `0x02BC`, transmitted `BC 02`.
4. Define a 100 ms transmission period and a 500 ms receiver timeout for a later sender/receiver implementation.
5. Specify that invalid status prevents the receiver from treating the temperature as a valid measurement.

**Check:** timeout detection needs receiver logic; `candump` alone does not implement that application behavior.

## Lesson 11: ISO-TP, the Transport Layer

**What is it?** ISO-TP, specified by ISO 15765-2, carries complete higher-layer messages over CAN frames. It provides segmentation, reassembly, flow control, sequence checking, and transport timeouts.

**Why do we need it?** A diagnostic response may exceed one frame. Even FD's 64-byte limit cannot hold every diagnostic message.

**How do we use it?** Configure a maintained ISO-TP implementation at both endpoints. A Single Frame carries a short message. A First Frame announces a longer message; the receiver sends Flow Control, and the sender sends Consecutive Frames. Block size and STmin control continuation and spacing.

Worked example on Classical CAN with normal addressing:

```text
UDS application request:       22 F1 90
CAN payload with ISO-TP:      03 22 F1 90 00 00 00 00
                             ^^          padding ----^
                             three application bytes
```

The trailing zeros are optional/configured padding here. FD single-frame length encoding differs for longer payloads; do not copy this short-frame layout for all FD messages.

**Practice, step by step**

1. Use an ISO-TP library example with two endpoints on `vcan0`.
2. Configure tester TX ID `0x7E0`, tester RX ID `0x7E8`, and the reversed mapping at the ECU.
3. Record normal 11-bit addressing, Classical/FD mode, frame data length, padding, block size, STmin, and timeouts.
4. Exchange a short message, then a 100-byte payload.
5. Capture and identify First Frame, Flow Control, and Consecutive Frames for the long transfer.
6. Remove the receiver and confirm the sender reports a bounded flow-control timeout.

**Check:** 100 bytes require segmentation even on FD. The chosen IDs are examples, not mandatory UDS IDs. Use documentation matching the installed transport library and backend.

## Lesson 12: UDS, the Diagnostic Application Layer

**What is it?** Unified Diagnostic Services (UDS) defines requests and responses for functions such as reading identification, changing diagnostic sessions, retrieving faults, and transferring data.

**Why do we need it?** CAN explains how frames are delivered, not how a tester asks an ECU for its identity or stored faults. UDS defines that application-level conversation.

**How do we use it?** A tester sends a service request; a supporting ECU validates the request and its current conditions, then responds. The stack for this course is:

```text
Tester application: UDS service request
        |
ISO-TP: complete message -> one or more CAN frames
        |
Classical CAN or CAN FD -> physical or virtual interface
        |
ECU transport reassembly -> UDS handler -> response
```

Physical addressing targets a particular ECU. Functional addressing targets a function and can involve multiple ECUs, with service-specific response restrictions. Start with physical addressing.

**Practice, step by step**

1. Select an instructor-provided simulator or implement handlers using an existing transport library.
2. Configure the simulator to support `0x22` and DID `0xF190`.
3. Send UDS bytes `22 F1 90` through ISO-TP, not as a raw unencapsulated CAN payload.
4. Capture both the application request and transport frames.

**Check:** `cansend vcan0 7E0#22F190` is not the ISO-TP-encoded request in Lesson 11. The remaining UDS lessons require a configured ECU simulator; `can-utils` alone does not create one.

## Lesson 13: Positive and Negative Responses

**What is it?** A positive response reports successful handling of a service. Many positive-response SIDs are the request SID plus `0x40`. A negative response has format `7F <request SID> <negative response code>`.

**Why do we need it?** A received CAN frame is not enough to decide whether a diagnostic operation succeeded. You must decode the service response.

| Request or response | Interpretation |
| --- | --- |
| `22 F1 90` | Read DID 0xF190 |
| `62 F1 90 ...` | Successful read response followed by data |
| `7F 22 31` | Read request rejected: requestOutOfRange |
| `7F 99 11` | Service 0x99 rejected: serviceNotSupported |
| `7F 22 78` | Request still being processed: response pending |

**How do we use it?** Decode the complete response, including echoed identifiers and data lengths. Match the request and service-specific conditions instead of assuming every failure uses the same NRC.

**Practice, step by step**

1. Configure a successful read of the training DID.
2. Request an unsupported DID and check the simulator's specified NRC.
3. Send unsupported service `99` and configure `7F 99 11`.
4. Submit an incorrectly sized request and check the specified response, commonly NRC `0x13` where applicable.

**Check:** NRC `0x78` is not final success. The tester must wait for the final response according to the timing rules and a bounded overall policy.

## Lesson 14: P2, P2*, S3, and TesterPresent

**What is it?** P2 bounds the initial server response timing. After a response-pending NRC, P2* governs the extended response wait. S3 is the server's non-default-session inactivity timeout. ISO-TP has separate transport timers.

**Why do we need it?** A client must distinguish slow processing from a missing response, and an ECU should not remain in a special diagnostic session indefinitely without appropriate activity.

**How do we use it?** Read supported timing values from the session response and apply the relevant client timing margins. Send TesterPresent when needed to maintain a non-default session. Use a bounded overall timeout even if pending responses continue.

A successful extended-session response begins `50 03`, followed by timing parameters. P2Server_max is encoded in milliseconds; P2*Server_max uses 10 ms units in the standard session response.

**Practice, step by step**

1. Enter the simulator's extended session with `10 03` and decode the returned timing parameters.
2. Send `3E 00`; the configured positive response is `7E 00`.
3. Configure a slow operation that sends NRC `0x78` before its final response.
4. Verify the tester waits according to P2* rather than incorrectly declaring initial-response timeout.
5. Stop diagnostic activity and verify the configured S3 timeout returns the simulator to default session.

**Check:** `3E 80` requests suppression of the positive response for TesterPresent. It does not mean that every negative response must also be suppressed; apply addressing and service rules.

## Lesson 15: Read and Write Data Identifiers

**What is it?** A Data Identifier (DID) identifies a diagnostic data item. Service `0x22` reads supported DIDs; `0x2E` writes supported writable DIDs.

**Why do we need it?** Testers need a defined interface for identification, measurements, and permitted configuration values without knowing internal ECU memory layouts.

**How do we use it?** Define each DID's data format, length, read/write access, and required state. Do not assume every DID can be written.

Worked examples for this simulator:

```text
Read synthetic VIN:       22 F1 90
Positive response:        62 F1 90 <17 configured ASCII bytes>

Write training DID F200:  2E F2 00 00 FA
Positive response:        6E F2 00
Read back:                22 F2 00
Positive response:        62 F2 00 00 FA
```

For this exercise only, define `0xF200` as a manufacturer-specific training setpoint: unsigned big-endian, 0.1 degrees C per count, allowed range 0-50 degrees C, writable in extended session. Thus `00 FA` means 25.0 degrees C. This is not a standardized meaning for that DID.

**Practice, step by step**

1. Configure DID `0xF190` with synthetic text `TESTVIN0000000001`, exactly 17 characters and not a real vehicle identity.
2. Read and decode it; observe segmentation if the configured CAN/ISO-TP frame size requires it.
3. Enter extended session and perform the example setpoint write.
4. Read it back and compare the bytes and physical value.
5. Try an out-of-range value and a write in the wrong session; verify rejection without data mutation.

**Check:** positive write response `6E F2 00` echoes the DID, not the written value. Readback supplies independent evidence of the stored value.

## Lesson 16: Sessions and SecurityAccess

**What is it?** A diagnostic session selects a set of supported behaviors. SecurityAccess `0x27` can unlock additional operations using an ECU-specific seed/key exchange. It is not encryption. Authentication `0x29` is a separate service, outside the implementation scope here.

**Why do we need it?** Reading identification and changing a protected configuration should not necessarily have the same prerequisites.

**How do we use it?** Specify session and access rules, then enforce them before changing state. Keep any toy access mechanism strictly inside the simulator.

| Training operation | Required session | Toy access required? |
| --- | --- | --- |
| Read synthetic VIN | Default or extended | No |
| Write training setpoint | Extended | No |
| Run training self-test | Extended | Yes |
| Dummy download | Programming, if implemented | Yes |

**Practice, step by step**

1. Implement or configure the table in the simulator.
2. Request extended session with `10 03` and confirm `50 03` plus timing parameters.
3. Request a toy seed at one configured level, for example `27 01`; expect `67 01 <seed>`.
4. Supply the documented test-fixture key with `27 02 <key>`; successful response is `67 02`.
5. Test a wrong key, configured attempt limit, and retry delay without bypassing them.
6. Document how session transitions and resets affect access state, and test those rules.

**Check:** session entry does not automatically unlock security. Never use production keys or attempt to infer or bypass a real ECU's algorithm.

## Lesson 17: RoutineControl and ECUReset

**What is it?** RoutineControl `0x31` invokes ECU-defined procedures. Subfunctions include start `0x01`, stop `0x02`, and request results `0x03`, where supported. ECUReset `0x11` requests a supported reset behavior.

**Why do we need it?** Some operations are procedures rather than simple reads or writes: for example, a simulated sensor self-test.

**How do we use it?** Specify a routine ID, allowed states, request parameters, result format, and whether it completes immediately or asynchronously. Support only the subfunctions you define.

```text
Start training routine:     31 01 F0 01
Positive response prefix:  71 01 F0 01
Request results:           31 03 F0 01
Positive response prefix:  71 03 F0 01
```

Routine ID `0xF001` is an example defined by this training simulator, not a universal self-test ID. Any additional status bytes require a documented format.

**Practice, step by step**

1. Configure a harmless self-test of a simulated sensor.
2. Attempt it before session/access prerequisites are met; verify the specified rejection.
3. Meet the prerequisites, start it, and obtain its results.
4. Request results before starting and test the documented sequence-error behavior.
5. Simulate a supported reset, such as the training ECU's `11 01` hard-reset request, and verify state reinitialization. Do not reset physical equipment without explicit authorization.

**Check:** accepting a routine-start request does not always mean the routine has finished successfully; interpret the defined result/status and timing.

## Lesson 18: Diagnostic Trouble Codes

**What is it?** A Diagnostic Trouble Code (DTC) records a detected fault according to the ECU's diagnostic rules. UDS `0x19` reads DTC information; `0x14` clears supported diagnostic information.

**Why do we need it?** A fault may be intermittent or no longer present when a technician connects. Stored status and contextual data can support investigation.

**How do we use it?** Define detection, debounce, DTC format, supported status bits, operation-cycle behavior, and clear behavior. A status byte can indicate testFailed, pendingDTC, confirmedDTC, and other states; these are not interchangeable.

Example: `19 02 01` requests reportDTCByStatusMask with the testFailed bit selected. A successful response begins `59 02`, then includes the status availability mask and matching DTC/status records as applicable. `14 FF FF FF` requests clearing the all-DTC group where supported and permitted; successful response is `54`.

**Practice, step by step**

1. Define one synthetic sensor fault that activates after three consecutive invalid samples.
2. Inject one invalid sample and verify the configured debounce behavior.
3. Inject enough invalid samples and read matching DTC records.
4. Decode the DTC bytes and supported status bits using the simulator specification.
5. Remove the fault, clear the supported group, and read again.
6. Reinject the fault and verify it is detected again.

**Check:** clearing a DTC does not repair the cause. Report the simulator's simplified lifecycle, not production-complete fault management.

## Lesson 19: Download and TransferData

**What is it?** RequestDownload `0x34` negotiates a transfer to ECU memory, TransferData `0x36` carries blocks, and RequestTransferExit `0x37` ends the transfer according to the ECU's rules.

**Why do we need it?** Diagnostic systems may need to transfer software or calibration data. A real update also needs authorization, validation, safe activation, and recovery beyond these three services.

**How do we use it?** In this course, transfer dummy bytes only into a simulator's memory buffer. Never flash or execute the data.

Worked toy transfer, assuming a simulator that explicitly supports this format and memory range:

```text
34 00 22 10 00 00 04   Request uncompressed/unencrypted download:
                       address length = 2 bytes, size length = 2 bytes,
                       address = 0x1000, size = 4 bytes.
74 10 06               Accept; maximum TransferData request length = 6 bytes.
36 01 DE AD BE EF      Transfer block 1: SID + counter + four data bytes.
76 01                  Accept block 1.
37                     Request transfer exit with no extra training parameters.
77                     Positive transfer-exit response.
```

The server's maximum block length includes service and counter overhead: six request bytes here permit four data bytes. TransferData's block counter is separate from ISO-TP's consecutive-frame sequence counter.

**Practice, step by step**

1. Configure a four-byte in-memory region, supported session/access conditions, and the exact request format above.
2. Send the download request and parse the negotiated length instead of assuming it.
3. Send block 1 and verify the echoed counter.
4. Exit the transfer and compare the buffer with `DE AD BE EF`.
5. Restart with an incorrect block counter and check the specified error, such as NRC `0x73` where applicable.
6. Test an out-of-range size and interrupted transfer; verify bounded failure handling and no activation.

**Check:** a successful `77` does not independently prove firmware authenticity, safe installation, or bootability. Those are outside this toy transfer.

## Lesson 20: Build, Test, and Troubleshoot the Complete System

**What is it?** Integration combines the message definitions, CAN interface, transport, diagnostic client, ECU handlers, timing, and state rules into one repeatable system.

**Why do we need it?** Individual features may work separately but fail together because of mismatched IDs, transport settings, timing, or session state.

**How do we use it?** Start small and keep evidence at each layer.

**Practice, step by step**

1. Save versions, interface setup, message definitions, IDs, transport parameters, and state tables.
2. Start the interface and capture first; then start the ECU simulator and tester.
3. Verify a simple frame, then ISO-TP exchange, then UDS read in that order.
4. Demonstrate the setpoint write/readback, routine, DTC lifecycle, and dummy download.
5. Repeat with an unsupported DID, wrong state, incorrect block counter, and missing ECU.
6. Record expected result, actual result, timestamped trace, and pass/fail for each case.
7. Restart from a clean configuration and repeat to check reproducibility.

| Symptom | First checks |
| --- | --- |
| No frames in capture | Interface name/state, process startup, send errors, capture filters |
| Hardware errors or bus-off | Wiring, termination, transceiver power, compatible timing, other active node |
| Frames visible, no ISO-TP message | Reversed TX/RX IDs, addressing mode, FD frame length, padding, flow control, transport timeout |
| ISO-TP complete, UDS negative response | SID, request length, DID/routine support, session, access, preconditions |
| Pending response then timeout | P2* handling, server progress, bounded overall timeout policy |
| Correct response bytes, wrong displayed value | Encoding, endianness, signedness, scaling, units |

**Check:** distinguish "not supported," "rejected," "timed out," and "not run." A negative response can be a passing test when rejection is the expected behavior.

## Lesson 21: UDS Versus OBD

**What is it?** On-Board Diagnostics (OBD) is a vehicle diagnostic system and regulatory framework. OBD-II and EOBD provide standardized access to emissions-related information. UDS is a general diagnostic service set for ECU development, manufacturing, service, and other supported diagnostic functions.

**Why do we need both?** A generic inspection tool needs standardized emissions information across manufacturers. An engineering or service tester may also need ECU-specific identification, routines, configuration, and authorized programming. These needs overlap, but are not identical.

| Question | OBD-II / EOBD | UDS |
| --- | --- | --- |
| Main purpose | Standardized regulated diagnostics, mainly emissions-related | General ECU diagnostics, depending on ECU implementation |
| Typical data | Supported PIDs, engine data, readiness, emissions DTCs, vehicle information | DIDs, diagnostic sessions, routines, DTC information, transfers |
| Application specification | Legacy services: ISO 15031-5 / SAE J1979; newer OBD-on-UDS: SAE J1979-2 | ISO 14229 family |
| Data identifiers | Legacy modes and PIDs, plus other service-specific records | DIDs, routine IDs, and service-specific parameters |
| CAN transport | ISO 15765-4 defines the regulated CAN profile; uses ISO-TP | ISO-TP with the applicable UDS-on-CAN profile |
| Access scope | Required supported emissions information, not unrestricted ECU access | ECU-specific support and session/access restrictions |
| Connector | Standardized diagnostic connector requirements apply to relevant vehicles | UDS itself does not require the OBD connector |
| CAN FD support | Do not assume it from an OBD label or 16-pin connector; verify the applicable profile | Can operate over CAN FD with compatible transport and endpoints |

**How do we use it?** Identify which application protocol and vehicle profile are supported before sending requests. An adapter, a connector, a transport protocol, and an application protocol are different things. A typical low-cost ELM327-type OBD adapter is not a general CAN FD adapter or an embedded UDS server.

Worked legacy OBD example, with application bytes shown before ISO-TP:

```text
01 0C            Request Mode 01, PID 0C: engine speed.
41 0C 1A F8      Successful example response.
                 RPM = (256 * 0x1A + 0xF8) / 4 = 1726 rpm.

22 F1 90         UDS request: read DID F190, vehicle identification number.
62 F1 90 ...     UDS positive response with the configured identification bytes.
```

The PID `0x0C` and DID `0xF190` belong to different data models; substituting one into the other request is not a conversion. PID support must be queried or established by the relevant specification.

For a Classical CAN, normal-addressing training example, the OBD request CAN data could be `02 01 0C 00 00 00 00 00`: `02` is the ISO-TP Single Frame application length. Functional ID `0x7DF` and responses in `0x7E8`-`0x7EF` are familiar 11-bit OBD conventions, but are not universal across all addressing formats and protocols. Use one configured simulator, not a vehicle, for this exercise.

**Important: OBD-on-UDS.** OBD and UDS are not always competing wire protocols. SAE J1979-2 defines OBD requirements using UDS services and standardized data definitions. ISO 14229 supplies diagnostic service mechanisms; the OBD profile supplies the regulated meanings and requirements. Do not equate older Mode/PID requests with all modern OBD implementations, or assume every UDS ECU implements OBD-on-UDS. Market, vehicle model year, and regulation determine applicability.

**Practice, step by step**

1. Configure a simulator with legacy Mode 01 PID 0C returning `1A F8`; retain the UDS VIN handler from Lesson 15.
2. Record separate supported-service/data tables for the OBD and UDS handlers.
3. Send `01 0C` through ISO-TP and decode the response as 1726 rpm.
4. Send `22 F1 90` and decode the synthetic VIN as ASCII, not as a PID response.
5. Request an unsupported PID and DID; check their own specified response behavior rather than assuming identical negative-response rules.
6. Draw two stacks: legacy OBD over ISO-TP/CAN, and UDS over ISO-TP/CAN FD. Add OBD-on-UDS as an OBD profile using UDS services.

**Check:** a tester that can read generic emissions data does not automatically support ECU programming, CAN FD, or every UDS service. OBD-on-UDS still needs its required profile and data definitions.

## Lesson 22: Move to Real Hardware

**What is it?** Replace the virtual bus with two active CAN FD endpoints, physical transceivers, wiring, and termination. The ECU application can still be a harmless training ECU rather than a vehicle module.

**Why do we need it?** Simulation does not verify voltages, physical bit timing, ACK behavior, termination, electrical faults, or real driver compatibility.

**How do we use it?** Follow [CAN-FD-UDS-Hardware-Guide.md](CAN-FD-UDS-Hardware-Guide.md). Start with an STM32G431-based development board and an FD transceiver. Use either a second such board or a verified USB-to-CAN FD adapter as the other endpoint. A controller needs firmware: buying FD-capable hardware does not install UDS or ISO-TP.

**Practice, step by step**

1. Select the two-board budget path or PC-adapter path and complete the guide's purchase checklist.
2. Record board revisions, exposed FDCAN pins, transceiver supplies, termination, and compatible timing.
3. Program a blink/logging example, then verify internal CAN loopback before wiring the bus.
4. With power off, wire and inspect the isolated bench. Bring up low-rate Classical CAN first.
5. Test FD without BRS, then FD with BRS; check received bytes and error counters at each stage.
6. Add a verified ISO-TP backend and UDS stack, then repeat the VIN and setpoint tests. Prove a 100-byte segmented exchange before claiming UDS-over-FD support.

**Check:** save hardware evidence separately from simulation evidence. The guide provides a bring-up procedure, not prebuilt firmware for a particular board revision.

## Final Learning Check

Before moving to a supervised hardware project, explain these without copying the lesson text:

1. What does CAN provide, and what must the application define?
2. When is Classical CAN sufficient, and why might you choose FD?
3. Why does FD not guarantee compatibility with Classical-only nodes?
4. How do DLC, application length, padding, and ISO-TP length differ?
5. How does a UDS request become one or more CAN frames?
6. Why are CAN ACK, ISO-TP completion, and UDS success different outcomes?
7. How do timing, session, and access state affect a request?
8. What physical and production-security claims have not been verified by simulation?
9. How do legacy OBD Mode/PID requests differ from UDS services/DIDs, and where does OBD-on-UDS fit?

Use the standards and library references in the syllabus for deeper study. Exact supported services, formats, timing, and state behavior must come from the applicable ECU specification and standards edition.