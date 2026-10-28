# CCSDS 235.1 Space Communications Session Control (SCSC) Specification

## Technical Architecture, Finite State Machine & Protocol Data Units

### 1\. Purpose & Strategic Scope

The **Space Communications Session Control (SCSC)** standard, designated **CCSDS 235.1-R.1 (Red Book, May 2026\)**, defines how space flight missions establish, manage, and conclude communications sessions between spacecraft and ground stations, or between cross-link spacecraft nodes (e.g. Lunar Gateway, rovers, orbiters, and deep space probes).

Historically, space communications relied on static, schedule-driven contact windows where ground transmitters were blindly energized based on pre-calculated orbital passes without an explicit bilateral session establishment handshake. This paradigm causes severe data loss during pointing errors, orbital insertion maneuvers, occultations, or multi-hop routing disruptions.

CCSDS 235.1 introduces a lightweight, robust, deterministic session layer sitting above transport and data link protocols (Space Packet, CFDP, DTN Bundle Protocol, and Telecommand). It provides:

1. **Explicit Session Establishment**: Bidirectional parameter negotiation and readiness verification.  
2. **Occultation & Link Outage Management**: Seamless transition into `SUSPENDED` state without tearing down cryptographic associations or draining flight software buffers.  
3. **Graceful Teardown Handshake**: Synchronized clean release preventing hanging transmitter power or lost in-flight frames.  
4. **Light-Time Delay Resilience**: Built-in awareness of deep-space One-Way Light Time (OWLT).

---

### 2\. SCSC Finite State Machine (FSM)

                       \+----------------+

                       \|      IDLE      \|\<-----------------------+

                       \+----------------+                        \|

                                \|                                \|

                     OPEN\_REQ   \|                                \|

                                v                                \|

                       \+----------------+                        \|

                       \|   INITIATING   \|                        \|

                       \+----------------+                        \|

                                \|                                \|

                  OPEN\_CONFIRM  \|                                \|

                                v                                \|

                       \+----------------+                        \|

          \+-----------\>\|     ACTIVE     \|\<-----------+           \|

          \|            \+----------------+            \|           \|

          \|               \|          \|               \|           \|

RESUME\_REQ\|    SUSPEND\_REQ\|          \|CLOSE\_REQ      \|           \|

          \|               v          v               \|           \|

          \|      \+------------+  \+------------+      \|           \|

          \+------\|  SUSPENDED \|  \|  CLOSING   \|------+           \|

                 \+------------+  \+------------+ CLOSE\_CONFIRM   \|

                                       \|                         \|

                                       \+-------------------------+

                                                ABORT / TIMEOUT

#### 2.1 State Definitions

- **`IDLE`**: No active session. Ground transmitter off or listening for incoming contact sweeps.  
- **`INITIATING`**: `OPEN_REQ` transmitted; waiting for remote peer confirmation or rejection.  
- **`ACTIVE`**: Full bidirectional communications established. Application PDUs (Space Packets, CFDP, DTN bundles) flow normally. Heartbeat keepalives exchanged.  
- **`SUSPENDED`**: Temporary link drop (planetary occultation, solar flare, attitude off-pointing). State and buffers preserved.  
- **`CLOSING`**: Graceful teardown initiated via `CLOSE_REQ`. In-flight buffers flushed.  
- **`ABORTED`**: Emergency teardown due to unrecoverable fault or security breach.

---

### 3\. Protocol Data Unit (PDU) Header & Types

SCSC utilizes a standardized 16-byte fixed header:

| Field Name | Bit Width | Data Type | Description |
| :---- | :---: | :---: | :---- |
| **Protocol ID** | 16 bits | `uint16` | Standardized SCSC identifier: `0x2351` (Decimal 9041\) |
| **Version** | 8 bits | `uint8` | Protocol version (`1` for initial release) |
| **PDU Type** | 8 bits | `uint8` | Control message identifier (`0x01` through `0x09`) |
| **Session ID** | 32 bits | `uint32` | Unique pseudorandom session identifier |
| **Sequence Number** | 16 bits | `uint16` | Strictly monotonic packet sequence counter |
| **Reserved / Flags** | 16 bits | `uint16` | Reserved for security association flags |
| **Timestamp** | 32 bits | `uint32` | CCSDS CDS coarse time epoch |
| **Payload Length** | 16 bits | `uint16` | Length of parameter payload in octets |
| **CRC-32** | 32 bits | `uint32` | Frame integrity check polynomial (\$0x04C11DB7\$) |

#### 3.1 PDU Types

- `OPEN_REQ` (`0x01`): Initiator requests session setup with service class and suggested timeout.  
- `OPEN_CONFIRM` (`0x02`): Responder acknowledges and accepts session parameters.  
- `OPEN_REJECT` (`0x03`): Responder declines session (station busy, maintenance, incompatible profile).  
- `KEEP_ALIVE` (`0x04`): Periodic heartbeat confirming continuous line-of-sight and receiver lock.  
- `SUSPEND_REQ` (`0x05`): Informs peer of upcoming temporary link loss (occultation).  
- `RESUME_REQ` (`0x06`): Requests restoration of suspended session.  
- `CLOSE_REQ` (`0x07`): Initiates graceful session termination.  
- `CLOSE_CONFIRM` (`0x08`): Acknowledges termination; endpoints return to `IDLE`.  
- `ABORT` (`0x09`): Immediate ungraceful termination.

---

### 4\. Deep-Space Delay Compensation

In interplanetary links, round-trip time (\$2 \\times \\text{OWLT}\$) can reach tens of minutes:

\$\$T\_{\\text{timeout}} \\ge 2 \\cdot t\_{\\text{prop}} \+ t\_{\\text{proc}} \+ t\_{\\text{margin}}\$\$

Where:

- \$t\_{\\text{prop}}\$: One-Way Light Time (\$\\approx 1.28\\text{s}\$ for Moon, \$180\\text{s}\$ to \$1300\\text{s}\$ for Mars).  
- \$t\_{\\text{proc}}\$: Ground station baseband processing delay (\$\\approx 1.0\\text{s}\$).  
- \$t\_{\\text{margin}}\$: Safety margin (\$\\ge 3.0\\text{s}\$).

SCSC parameter negotiation enforces that timers adapt dynamically to the ephemeris-derived propagation delay.