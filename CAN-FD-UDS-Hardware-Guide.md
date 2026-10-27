# CAN FD and UDS: Low-Cost Real-Hardware Guide

## 1. Recommended Starting Point

For a beginner buying new hardware, start with an **STM32G431-based development board, such as NUCLEO-G431KB, plus an external CAN FD transceiver**. It has a native FDCAN controller and an onboard ST-LINK programmer/debugger. You do not need a separate SPI CAN controller for this MCU.

You need two active bus endpoints. Choose either a second MCU board or a PC USB-to-CAN FD adapter. One MCU alone can test internal loopback but cannot demonstrate a real two-node bus exchange. An adapter used only in listen-only mode will not ACK frames.

This recommendation favors low total learning cost and documented tools. It is not a verified claim of the world's cheapest current listing. Reuse an existing compatible board before buying another.

Read [CAN-FD-UDS-Lessons.md](CAN-FD-UDS-Lessons.md), especially Lessons 9, 11, and 22, and the standards map in [CAN-FD-UDS-References.md](CAN-FD-UDS-References.md).

## 2. Controllers That Support CAN FD

| Controller / board | CAN FD capability | What else is required? | Budget decision |
| --- | --- | --- | --- |
| STM32G431KB / NUCLEO-G431KB | One native FDCAN controller | External FD transceiver, wiring, firmware | Recommended low-cost documented development-board option |
| STM32G0B1RE / suitable G0B1 board | Two native FDCAN controllers | Confirm exact board pin access and whether transceiver(s) are populated | Good alternative; compare complete board cost, not just MCU price |
| NXP i.MX RT1062 / Teensy 4.0 | Three CAN controllers, one FD-capable | Use the FD-capable instance/pins and an external FD transceiver | Good if already owned; usually not the lowest-cost new option |
| Microchip MCP2518FD | External SPI CAN FD controller, not an MCU | Host MCU, clock, SPI driver, FD transceiver unless module includes it | Useful when reusing an existing MCU without native FD |
| MCP2515 modules | Classical CAN only | Cannot be upgraded to FD with firmware | Do not buy for FD |
| Original ESP32, ESP32-S3, or STM32F103 boards | Not native FD choices for this guide | Add a supported external FD controller and transceiver if reusing | Check exact chip variant; a family name or "CAN" label is insufficient |

Manufacturer evidence:

- [STM32G431KB](https://www.st.com/en/microcontrollers-microprocessors/stm32g431kb.html) and [NUCLEO-G431KB](https://www.st.com/en/evaluation-tools/nucleo-g431kb.html).
- [STM32G0B1RE](https://www.st.com/en/microcontrollers-microprocessors/stm32g0b1re.html) and [NUCLEO-G0B1RE](https://www.st.com/en/evaluation-tools/nucleo-g0b1re.html). Generic Nucleo feature lists cover multiple variants: verify the exact schematic/BOM before assuming an onboard FD transceiver.
- [Teensy 4.0](https://www.pjrc.com/store/teensy40.html).
- [MCP2518FD controller](https://www.microchip.com/en-us/product/mcp2518fd).

Bare chips can cost much less than development boards, but need PCB design, power, clocking, programming access, and assembly. They are not the cheapest complete beginner setup unless that infrastructure already exists.

## 3. Shopping List and Approximate Budget

The amounts below are **planning estimates in USD, not checked seller quotations**. Shipping, VAT, import charges, region, stock, and tool ownership can change the total. Check local distributors before purchase. No hardware was purchased or tested while preparing this guide.

### Path A: Two MCU Boards, No Dedicated PC CAN Adapter

| Item | Quantity | Estimated total |
| --- | --- | --- |
| NUCLEO-G431KB or comparable documented FD MCU board with programmer | 2 | $30-50 |
| FD transceiver breakout compatible with MCU logic levels | 2 | $6-20 |
| Two 120-ohm terminators, short twisted-pair bus, headers, and USB data cables | 1 set | $5-15 |
| Multimeter and suitable USB power | Reuse existing | Not included |
| Planning total | | $41-85 |

Board A runs a tester and logs results over USB serial; Board B runs the training ECU. ST-LINK virtual COM logging is not a SocketCAN adapter. You must develop/port the tester firmware as well as ECU firmware. A PC adapter can be added later for independent captures.

### Path B: PC Tester Plus One MCU ECU

| Item | Quantity | Estimated total |
| --- | --- | --- |
| NUCLEO-G431KB or comparable board | 1 | $15-25 |
| FD transceiver breakout | 1 | $3-10 |
| Verified low-cost USB-to-CAN FD adapter | 1 | $15-50 target allowance |
| Termination, short cable, headers, USB data cables | 1 set | $5-15 |
| Planning total, if adapter target price is available | | $38-100 |

This is usually easier for PC-side captures and `udsoncan` exercises, provided the adapter/firmware/driver combination is known to work. A supported branded or isolated adapter may cost substantially more than this target allowance.

**Lowest-cash choice:** compare Path A's complete parts total with a locally verified Path B quote. Path A avoids adapter-compatibility purchases but requires more firmware work. A suspiciously cheap "USB CAN" listing may support Classical CAN only.

Optional additions: an isolated adapter, oscilloscope, known-good reference node, and a more robust harness. An OBD connector, vehicle ECU, engine, and 12 V vehicle battery are not needed for this training bench.

## 4. Pick the Correct Transceiver and Adapter

An FD **controller** creates protocol frames. An FD **transceiver** creates the bus electrical signals. A transceiver-only module cannot add CAN FD capability to a Classical-only MCU.

One example is an [MCP2562FD-based breakout](https://www.microchip.com/en-us/product/mcp2562fd). The chip uses a nominal 5 V VDD supply and a separate VIO supply for MCU logic compatibility; use the datasheet limits and breakout schematic. For a 3.3 V MCU, configure VIO appropriately. Do not assume the whole chip runs from 3.3 V just because its digital interface can.

Pre-purchase checklist:

- [ ] Exact MCU/controller supports ISO CAN FD and exposes usable TX/RX pins.
- [ ] Transceiver supports the intended FD rate and has documented supply/logic levels.
- [ ] Breakout includes the needed decoupling and exposes standby/enable controls.
- [ ] Adapter explicitly supports FD transmit AND receive, BRS, and required payload lengths.
- [ ] Exact adapter hardware revision, firmware, OS driver, and host API are documented together.
- [ ] The chosen Linux path exposes a real FD-capable SocketCAN interface, if using the commands below.
- [ ] Termination jumpers are understood, so the finished bus has two end terminations rather than one per attached device.
- [ ] USB cables carry data, not just power; the programmer and headers are included or budgeted.

CANable 2.0-style hardware can be inexpensive, but clones and firmware are not interchangeable. The [CANable 2.0 firmware](https://github.com/normaldotcom/canable2-fw) documents **non-standard SLCAN FD extensions**; a generic Classical SLCAN host tool may not support them. The [mainline candleLight firmware](https://github.com/candle-usb/candleLight_fw) lists hardware limitations, including unsupported board variants. Do not assume its familiar name guarantees working FD `gs_usb` support for your adapter. Require evidence for the exact combination, or choose another documented adapter.

## 5. Open-Source UDS and Transport Options

Yes, open-source UDS implementations exist. Select software for its role, not just a repository name.

| Project | Role | License / integration note |
| --- | --- | --- |
| [iso14229](https://github.com/driftregion/iso14229) | Embedded C UDS server AND client; examples include STM32 and other platforms | MIT; application handlers and hardware integration still required; pin a release because the API is pre-1.0 |
| [udsoncan](https://github.com/pylessard/python-udsoncan) | Python UDS tester/client and payload encoding/decoding | MIT; not a ready-made ECU server |
| [can-isotp](https://github.com/pylessard/python-can-isotp) | Python ISO-TP and kernel-socket integration | Check release license; documented FD parameters, not MCU firmware |
| [python-can](https://github.com/hardbyte/python-can) | Host CAN adapter abstraction | Check release license and selected backend's FD support |
| [Linux ISO-TP](https://docs.kernel.org/networking/iso15765-2.html) | Kernel ISO-TP over SocketCAN, with FD link-layer options | Linux environment and compatible kernel/interface required |
| [can-utils](https://github.com/linux-can/can-utils) | Capture, transmit, and transport utilities | Tool-specific licenses; not a UDS ECU implementation |

**Critical FD check:** `iso14229` separates UDS from the transport and offers several transport choices. Do not assume its embedded `isotp-c` backend or a sample written for Classical CAN supports FD segmentation. Before committing to an embedded backend, inspect its documented FD support and prove >8-byte frames and a 100-byte ISO-TP exchange on the exact build. A working C UDS library alone does not establish this.

Suggested integration order:

1. PC simulation: `udsoncan` tester plus a UDS server over a known compatible ISO-TP backend.
2. MCU: use vendor FDCAN examples to validate raw frames before adding UDS.
3. Embedded UDS: use `iso14229` handlers and a transport that has been verified for the selected mode. Bring up Classical CAN UDS first if the available embedded transport only supports Classical frames.
4. FD diagnostics: connect an existing FD-capable ISO-TP implementation through the UDS transport interface; if unavailable for the selected MCU/toolchain, keep that stage blocked rather than claiming FD transport works. Do not hand-roll ISO-TP as a beginner exercise.

This document does not supply or certify a ready-to-flash STM32 FD UDS port. Check upstream examples, licenses, release notes, and test coverage; preserve required notices when reusing source. Open-source availability is not proof of ISO conformance, functional safety, or production cybersecurity.

## 6. Safe Wiring for the Recommended Bench

For Path A, both ends have an MCU and transceiver. For Path B, the adapter normally includes its own controller/transceiver.

```text
PC -- USB -- CAN FD adapter ===== twisted pair ===== FD transceiver -- MCU
              termination          CAN_H/CAN_L          termination
                120 ohm                                  120 ohm
```

MCU-side logical connections, not universal connector pin numbers:

| Connection | Action |
| --- | --- |
| MCU FDCAN_TX | Connect to transceiver TXD |
| MCU FDCAN_RX | Connect to transceiver RXD |
| Transceiver CAN_H | Connect to bus CAN_H |
| Transceiver CAN_L | Connect to bus CAN_L |
| VDD, VIO, GND | Connect according to chip and board supply requirements |
| STBY / EN | Select normal operation according to the transceiver truth table |
| Bus reference / isolation | Follow both endpoints' grounding/isolation instructions |

Do not connect MCU TX/RX directly to CAN_H/CAN_L. Do not tie independent power outputs together or backfeed USB supplies. Use the exact Nucleo user manual, schematic, MCU alternate-function table, and CubeMX pin assignment: header labels are not interchangeable with MCU package pin numbers.

With power off and the hardware safe to measure, two 120-ohm end terminators typically measure about 60 ohms across CAN_H/CAN_L. Keep the initial bus short, such as under 1 m, with short stubs. This is a conservative training layout, not a guarantee of any bitrate.

## 7. Step-by-Step Bring-Up

### Stage 1: Prove the Board and Pin Configuration

1. Install STM32CubeIDE/CubeMX or the board's documented toolchain and a matching MCU package.
2. Program a blink and serial-log example through the onboard ST-LINK.
3. Select an exposed FDCAN TX/RX alternate-function pair; check board conflicts and solder bridges.
4. Record the FDCAN kernel clock, nominal timing, data timing, filters, and message RAM/FIFO configuration.
5. Run the vendor's internal loopback example and compare sent/received bytes.

Pass: reliable loopback and logs. Internal loopback does not verify the transceiver or wiring.

### Stage 2: Prove Two-Node Classical CAN

1. Power off, wire both endpoints, check termination and supplies, then power up.
2. Configure both endpoints for 500 kbit/s Classical CAN and normal active mode.
3. Send a four-byte counter frame at approximately 10 frames per second, then reverse direction.
4. Check exact payloads, received counts, and controller error status. Do not confuse a local transmit echo with reception at the other node.

Pass: each endpoint independently receives frames with stable error counters and no bus-off. Do not advance if this fails.

### Stage 3: Prove Raw CAN FD

1. Configure both endpoints for ISO CAN FD and matching nominal/data timing.
2. Send 12, 32, and 64-byte test patterns without BRS first.
3. Enable BRS at both endpoints, initially targeting 500 kbit/s nominal and 2 Mbit/s data if the hardware/timing permits.
4. Repeat both directions and compare every byte. Check FD/BRS metadata and error counters.

For an adapter already exposed as FD-capable SocketCAN `can0`, these are Linux commands, not PowerShell commands:

```bash
sudo ip link set can0 down
sudo ip link set can0 type can bitrate 500000 dbitrate 2000000 fd on
sudo ip link set can0 up
ip -details -statistics link show can0
candump -x can0
```

In a second Linux terminal, send the known 12-byte BRS test frame:

```bash
cansend can0 123##100112233445566778899AABB
```

The MCU must be programmed to receive it and supply independent logs. Use the actual interface name; these commands do not make a serial SLCAN adapter FD-capable. Driver timing constraints may require documented sample-point or other settings.

Windows users can keep the MCU IDE on Windows. For these SocketCAN commands, prefer native Linux or a Linux VM with supported USB passthrough. WSL2 requires compatible kernel modules plus explicit USB forwarding, such as a supported usbipd-win setup; USB adapters do not automatically appear as `can0`. A native Windows application needs an adapter backend explicitly supporting FD instead.

Pass: 12/32/64-byte patterns verified in both directions, including BRS, with recorded timing and controller status. No hardware test is claimed complete by merely displaying the configuration.

### Stage 4: Prove ISO-TP, Then UDS

1. Configure normal 11-bit physical addressing: tester TX/RX `0x7E0`/`0x7E8`, ECU reversed.
2. Match Classical/FD mode, ISO-TP frame data length, padding, flow-control settings, and timeouts.
3. For the documented Python transport, FD settings include `can_fd=True`, `bitrate_switch=True`, and a suitable `tx_data_length`, such as 64. Configure the underlying adapter too; these flags do not set physical bitrates.
4. Test short payload delivery, then a 100-byte segmented exchange and receiver-removal timeout.
5. Add UDS `0x22` for the synthetic VIN and confirm the decoded value.
6. Add session control and TesterPresent, then the training setpoint with bounded validation.
7. Repeat negative-response and timeout tests. Keep downloads in RAM and all resets confined to your training firmware.

Pass: correct application bytes AND transport traces, including flow control for a long message. Record separate results for Classical UDS and FD UDS; success on one does not prove the other.

## 8. Evidence and Next Milestone

| Test | Required evidence |
| --- | --- |
| Physical setup | Board/adapter revision, schematic references, wiring photo, termination and supply checks |
| Raw CAN / FD | Pattern lengths, received bytes, timing, BRS status, error counters |
| ISO-TP | Transport version/backend, settings, 100-byte reassembly, controlled timeout |
| UDS / OBD training | Request/response bytes, decoded VIN/RPM, expected rejection behavior |
| Repeatability | Firmware commit, host versions, startup steps, actual pass/fail/not-run results |

First milestone: two-node raw FD communication. Second milestone: a real-hardware synthetic VIN read with a verified transport. Do not buy a vehicle ECU or attempt a production bootloader until these steps are independently understood and tested.