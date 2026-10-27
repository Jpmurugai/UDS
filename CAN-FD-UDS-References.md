# CAN, CAN FD, UDS, and OBD: Reference Documents

## 1. Scope and Access

This document maps the standards relevant to the [lessons](CAN-FD-UDS-Lessons.md) and [hardware guide](CAN-FD-UDS-Hardware-Guide.md). It covers the core families and related extensions, not every ISO standard in existence or a reproduction of their full text.

ISO/SAE standards are generally copyrighted and may require purchase or institutional access. Use official catalogue summaries to identify a document, then obtain a licensed copy through your college, employer, ISO, SAE, or national standards body. Public tutorials and open-source code are useful learning aids, not normative specifications.

Reference review date: 2026-10-27. The ISO catalogue returned published entries for ISO 11898-1:2024, ISO 11898-2:2026, ISO 15765-2:2024, ISO 14229-1:2026, ISO 14229-2:2021, and ISO 14229-3:2022. These are catalogue observations, not a requirement to migrate an existing project. Check amendments, publication status, and the project's required edition before implementation; library support can lag newer editions.

## 2. Which Standard Answers My Question?

| Question | Start here |
| --- | --- |
| How are Classical CAN and CAN FD frames encoded? | ISO 11898-1 |
| What electrical characteristics must the transceiver meet? | ISO 11898-2 |
| How is a long diagnostic message split and reassembled? | ISO 15765-2 |
| What does service 0x22, 0x19, or 0x34 mean? | ISO 14229-1 |
| Which diagnostic session-layer timing rules apply? | ISO 14229-2 plus the applicable service/profile |
| How is UDS applied to CAN? | ISO 14229-3 |
| What extra profile applies to emissions diagnostics over CAN? | ISO 15765-4 and relevant application/regulatory requirements |
| What are legacy OBD modes and PIDs? | ISO 15031-5 / SAE J1979 and applicable digital annex |
| Where are standardized OBD DTC definitions? | ISO 15031-6 / SAE J2012 and applicable digital annex |
| What defines the diagnostic connector? | ISO 15031-3 / SAE J1962 |
| How does OBD use UDS? | SAE J1979-2; distinguish it from ISO 27145 WWH-OBD |
| What if diagnostics use Ethernet/IP? | ISO 13400 plus ISO 14229-5 |

## 3. CAN and CAN FD: ISO 11898 and ISO 16845

Official catalogues: [ISO 11898 search](https://www.iso.org/search.html?q=ISO%2011898) and [ISO 16845 search](https://www.iso.org/search.html?q=ISO%2016845).

| Document | What it covers | Course relevance |
| --- | --- | --- |
| ISO 11898-1 | CAN data link layer and physical coding sublayer | Core: identifiers, frame formats, arbitration, error handling, Classical CAN and FD |
| ISO 11898-2 | High-speed physical medium attachment | Core: transceiver behavior and electrical compatibility |
| ISO 11898-3 | Low-speed fault-tolerant CAN physical interface | Background only; not the high-speed FD bench used here |
| ISO 11898-4 | Time-triggered CAN communication | Optional background; not required for these labs |
| ISO 11898-5 / -6, historical references | High-speed low-power/selective-wake material in older document structures | Check catalogue status and how requirements are incorporated into the applicable ISO 11898-2 edition |
| ISO 16845-1 | CAN data-link/physical-signaling conformance test plan | Advanced: formal protocol testing, not just successful message exchange |
| ISO 16845-2 | High-speed CAN physical-layer conformance tests | Advanced hardware qualification |

Newer ISO 11898-1 editions also cover CAN XL. That does not change FD's 64-byte maximum or make an FD-only controller support XL. Ensure the hardware supports **ISO CAN FD**, not only an early non-ISO FD implementation.

## 4. Transport and DoCAN: ISO 15765

Official catalogue: [ISO 15765 search](https://www.iso.org/search.html?q=ISO%2015765).

| Document | What it covers | Course relevance |
| --- | --- | --- |
| ISO 15765-2 | Transport protocol and network-layer services, ISO-TP | Core: addressing, SF/FF/CF/FC, segmentation, flow control, sequence counters, timing, applicable FD encodings |
| ISO 15765-4 | Requirements for emissions-related systems | Core OBD reading: regulated CAN communication profile, addressing, initialization, and compatibility requirements |
| ISO 15765-5 | In-vehicle network connected to the diagnostic link connector | Extended reading: requirements for the connection from external test equipment to the vehicle network |
| ISO 15765-1 / -3, older references | Older general-information and UDS-on-CAN document organization | Historical only: do not substitute an old part for the applicable ISO 14229-3 and current transport/profile requirements |

ISO-TP does not decide which DIDs exist. It delivers application messages. Conversely, an application library accepting a 100-byte buffer does not prove its chosen transport can carry it over CAN FD.

## 5. UDS: ISO 14229 Parts

Official catalogue: [ISO 14229 search](https://www.iso.org/search.html?q=ISO%2014229).

| Part | Subject | What to study |
| --- | --- | --- |
| ISO 14229-1 | Application layer | Service requests/responses, NRCs, sessions, data access, routines, DTC services, transfers, access-related services |
| ISO 14229-2 | Session-layer services | Common session-layer services and timing relationships independent of the underlying transport |
| ISO 14229-3 | UDS on CAN, UDSonCAN | Application profile and implementation requirements for CAN-based UDS |
| ISO 14229-4 | UDS on FlexRay | Awareness only; not a CAN FD requirement |
| ISO 14229-5 | UDS on Internet Protocol, UDSonIP | Later Ethernet/DoIP learning path |
| ISO 14229-6 | UDS on K-Line | Awareness only; do not confuse with legacy KWP2000 application services |
| ISO 14229-7 | UDS on LIN | Awareness only; separate network and transport constraints |
| ISO 14229-8 | UDS on CXPI | Awareness only; not required for this bench |

The ECU's diagnostic specification must additionally define supported services, DIDs, routines, DTCs, codecs, access rules, and state transitions. ISO 14229 is not a universal list of all manufacturer-specific ECU data.

## 6. Legacy OBD: ISO 15031 and SAE Documents

Official catalogue: [ISO 15031 search](https://www.iso.org/search.html?q=ISO%2015031). Find SAE documents by identifier in the [SAE standards catalogue](https://www.sae.org/standards).

| ISO document | Related SAE reference where applicable | Purpose |
| --- | --- | --- |
| ISO 15031-1 | Consult applicable profile | General information and use-case definition |
| ISO 15031-2 | SAE J1930 | Terms, definitions, abbreviations, and acronyms |
| ISO 15031-3 | SAE J1962 | Diagnostic connector and related electrical circuits |
| ISO 15031-4 | SAE J1978 | External emissions diagnostic test equipment |
| ISO 15031-5 | SAE J1979; J1979-DA where applicable | Emissions diagnostic services and standardized data definitions |
| ISO 15031-6 | SAE J2012; J2012-DA where applicable | Diagnostic trouble code definitions |
| ISO 15031-7 | Consult applicable edition | Data-link security for emissions-related diagnostics |

Related ISO and SAE documents are not automatically identical in every revision. Match editions and regulations. Here, SAE J1930 concerns terminology; SAE J1939 is a different heavy-duty network family.

Older OBD communication may use ISO 9141-2, ISO 14230-4 (KWP2000 emissions profile), or SAE J1850 rather than CAN. The ISO 14230 family also separates physical, data-link, and application requirements in Parts 1, 2, and 3. These are background references, not reasons to buy a K-Line adapter for CAN FD lessons.

## 7. OBD-on-UDS and WWH-OBD

| Reference | Scope | Important distinction |
| --- | --- | --- |
| SAE J1979-2 | OBD communication using UDS | Regulated OBD profile/data requirements using UDS service mechanisms |
| ISO 27145-1 | WWH-OBD general information and use cases | Worldwide harmonized OBD overview |
| ISO 27145-2 | WWH-OBD common data dictionary | Defines the data elements |
| ISO 27145-3 | WWH-OBD common message dictionary | Uses a specified subset of UDS services to communicate the data |
| ISO 27145-4 | WWH-OBD vehicle/test-equipment connection | Establishing and maintaining the relevant connection |

Official catalogue: [ISO 27145 search](https://www.iso.org/search.html?q=ISO%2027145).

OBD-on-UDS and WWH-OBD both use UDS concepts, but are not names for an interchangeable single profile. Do not infer the mandated protocol, adoption date, or data support from a connector shape or a generic claim of "UDS support." Check the applicable market, model year, vehicle category, and legislation.

## 8. Related Standards for Later Study

| Family | What it adds | Needed for the initial bench? |
| --- | --- | --- |
| ISO 13400, especially -1, -2, -3, -4 | DoIP overview, transport/network services, wired interface, and connector requirements | No; later Ethernet diagnostics |
| ISO 22901-1 / -2 | ODX diagnostic data exchange model and emissions-related diagnostic data | No; useful for machine-readable diagnostic descriptions |
| ISO 22900 family | Modular vehicle communication interface, including D-PDU API in Part 2 | No; useful when integrating professional tester interfaces |
| ISO 26262 family | Functional safety lifecycle for relevant road-vehicle electrical/electronic systems | Not a claim established by these labs |
| ISO/SAE 21434 | Automotive cybersecurity engineering lifecycle | Later engineering context; a toy seed/key exercise does not satisfy it |

Find these by their exact numbers using the [ISO catalogue](https://www.iso.org/standards.html) or [ISO Online Browsing Platform](https://www.iso.org/obp/ui/). Some catalogue pages expose only summaries; full normative requirements still require the appropriate access.

## 9. Recommended Reading Order

1. Lessons 3-10, then the relevant Classical/FD and physical sections of ISO 11898-1/-2.
2. ISO 15765-2 while inspecting a short and a segmented transport capture.
3. ISO 14229-1 service definitions alongside the actual UDS exercises.
4. ISO 14229-2 and -3 to connect timing and CAN-specific profile requirements.
5. Lesson 21, ISO 15031-5/-6, ISO 15765-4, and the relevant SAE data annexes.
6. SAE J1979-2 or ISO 27145 when studying that specific OBD profile.
7. Conformance, ODX, DoIP, safety, and cybersecurity references only when the project needs them.

## 10. Free Implementation and Tutorial References

- [Linux SocketCAN](https://docs.kernel.org/networking/can.html): interface configuration, frames, driver behavior, and error handling.
- [Linux ISO-TP](https://docs.kernel.org/networking/iso15765-2.html): kernel transport API and CAN FD link-layer options.
- [can-utils](https://github.com/linux-can/can-utils): capture, transmit, and transport utilities.
- [iso14229](https://github.com/driftregion/iso14229): open-source C UDS server/client; verify the selected transport backend separately.
- [udsoncan](https://udsoncan.readthedocs.io/): Python UDS tester/client and payload handling.
- [can-isotp](https://can-isotp.readthedocs.io/): ISO-TP transport options and FD parameters.
- [python-can](https://python-can.readthedocs.io/): adapter APIs; backend capabilities differ.
- [OBD tutorial](https://www.csselectronics.com/pages/obd2-explained-simple-intro): secondary explanation and worked examples, not normative requirements.

For reproducibility, record the standard edition, library release/commit, board revision, adapter firmware, and driver/kernel version with each test report.