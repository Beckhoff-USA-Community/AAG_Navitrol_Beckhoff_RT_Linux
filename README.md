# Navitrol Ethernet Supervisor Interface - TwinCAT 3 Demo

This repository contains a complete TwinCAT 3 PLC example implementation demonstrating TCP/IP communication with the Navitec Navitrol navigation system using the **Ethernet Supervisor Interface Protocol Version 4**.

## Overview

![System Topology](Topology.svg)

## Table of Contents

- [Overview](#overview)
- [Test Environment Specifications](#test-environment-specifications)
- [Implemented Messages](#implemented-messages)
- [Protocol Implementation](#protocol-implementation)
  - [Message Flow](#message-flow)
  - [How Received Messages Are Handled](#how-received-messages-are-handled)
- [Customization Guide](#customization-guide)
  - [Adding New Message Types](#adding-new-message-types)
  - [Adjusting Cycle Times](#adjusting-cycle-times)
- [Additional Resources](#additional-resources)
- [Support](#support)

## Test Environment Specifications

**Hardware:**
- Test fixture: Beckhoff C6032-0080
  - *Note: Please work with your local Beckhoff team to understand what IPC makes sense for your application*

**Beckhoff RT Linux®:**
- Version: 13
- Build: 262191
- Update: 19.12.2025

**Beckhoff RT Linux® Packages:**
- `tf6310-tcp-ip` version 2.0.22-1
- `tc31-xar-um` version 4026.21.107-1

**Navitec Software:**
- Navitrol 7.1.0-alpha.1.deb
- NavitrolMonitor_1.70.00_Navitrol_7.1.0-alpha.1.zip
- Ethernet Supervisor Documentation dated 17.10.2025

**Development PC:**
- TwinCAT XAE 3.1.4026.20
- TF6310 3.3.23

**Important parameters for Ethernet Supervisor interface:**

```
I,supervisor_eth_port,                  2000       # Supervisor communication port (1-65535)
I,supervisor_eth_server,                0          # Server mode: 0=client, 1=server
I,supervisor_eth_endianness,            0          # Endianness: 0=little endian, 1=big endian
I,supervisor_eth_enabled,               1          # Enable Supervisor interface
I,control_interval_us,                  30000      # min 100, max 100000
```

**Note:** A copy of the complete parameter file used for testing is stored in this repository at [`Navitrol Param File/params.txt`](Navitrol%20Param%20File/params.txt).

## Implemented Messages

| ID   | Direction               | Name                                 | Purpose                                   | Cycle Time |
|------|-------------------------|------------------------------------- |-------------------------------------------|------------|
| 1001 | Supervisor → Navitrol   | Initialize Position                  | Set AGV position on map                   | On demand  |
| 1026 | Supervisor → Navitrol   | Measurement Update (Generic Omni)    | Send odometry, receive control refs       | 30ms       |
| 3002 | Supervisor → Navitrol   | Supervisor Status                    | Send supervisor state & requests          | 100ms      |
| 1100 | Navitrol → Supervisor   | Position Initialization Started      | Acknowledge init request                  | Response   |
| 1101 | Navitrol → Supervisor   | Position Initialized                 | Init result with confidence               | Response   |
| 1126 | Navitrol → Supervisor   | Motor Control (Generic Omni)         | Wheel speed & angle references            | Response   |
| 3102 | Navitrol → Supervisor   | Navitrol Status                      | Navitrol state, errors, warnings          | Response   |

## Protocol Implementation

### Message Flow

**Sequence:**
1. Supervisor sends **3002 (Supervisor Status)** → Navitrol responds with **3102 (Navitrol Status)**
2. Supervisor sends **1026 (Measurement Update)** → Navitrol responds with **1126 (Motor Control)**
3. When needed, Supervisor sends **1001 (Initialize Position)** → Navitrol responds with **1100**, then **1101**

**Cyclic Operation:**
- **3002** sent every 100ms (configurable via `FB_Supervisor.CycleTime3002`)
  - Maximum interval: 1 second
- **1026** sent every 30ms (configurable via `FB_Supervisor.CycleTime1026`)
  - Must match Navitrol parameter `control_interval_us` in params.txt
  - Refer to "Measurement update message" in the glossary section of Navitrol documentation Section 11.1
- Responses received asynchronously via continuous socket receive

### How Received Messages Are Handled

Incoming messages are received into a union buffer that acts as a raw byte array:

```iecst
// Bytes bucket to buffer received messages.
// Header info is parsed before being copied into the correct message structure.
// PLC logic is built expecting to receive the entire TCP message payload with one call to FB_SocketReceive.
// FB_SocketReceive will fragment the received data if the receive buffer is not large enough.
// Make sure this is large enough to fit the biggest received message.
TYPE U_Received_bytes_from_Navitrol : UNION
    Header : ST_SupervisorHeader;
    Bytes  : ARRAY[0..Param_Supervisor_response.ReceivedBytesBufferSize] OF UINT;
END_UNION
END_TYPE
```

**How it works:**
1. Raw TCP data is received into the `Bytes` array
2. The `Header` overlay allows reading the message type without copying
3. Message is identified by `Header.Message_ID`
4. Full message is copied to the appropriate typed output variable using `MEMCPY`

This pattern ensures the receive buffer is large enough for any message type while maintaining type safety for processed messages.

## Customization Guide

### Adding New Message Types

Follow the established pattern to add new messages:

1. **Create the struct** in the appropriate folder:
   - Outgoing: `Supervisor to Navitrol message/<MessageID>/`
   - Incoming: `Navitrol to Supervisor response/<MessageID>/`

2. **Define the struct:**
   ```iecst
   {attribute 'pack_mode' := '1'}  // Byte-aligned
   TYPE ST_<MessageID>_<Description> EXTENDS ST_SupervisorHeader : STRUCT
       // Then: Payload fields
       Field1 : UINT;
       Field2 : REAL;
       // ...
   END_STRUCT
   END_TYPE
   ```

3. **For outgoing messages:**
   - Call `F_SetHeaderInfo(message)` and `F_SetSectionHeaderInfo()` (in the case of msg 3002) before sending
   - Add send logic in `FB_Supervisor` COMS_ACTIVE state (or create new internal METHOD following the pattern of `SendMsg3002()`, `SendMsg1026()`, `SendMsg1001()`)
   - Create timer if cyclic, or event trigger if on-demand

4. **For incoming messages:**
   - Add CASE handler in `ReceiveAllMsgsAndParse()` METHOD in FB_Supervisor
   - Add output variable to FB_Supervisor for storing the received message
   - Use `MEMCPY` to copy from `ReceiveBuffer` union to the typed output variable

   Example pattern:
   ```iecst
   // In ReceiveAllMsgsAndParse() method:
   CASE ReceiveBuffer.Header.Message_ID OF
       // ... existing cases ...
       2050:
           NumOf2050MsgReceived := NumOf2050MsgReceived + 1;
           MEMCPY(destAddr := ADR(st2050_New_Message),
                  srcAddr := ADR(ReceiveBuffer),
                  n := SIZEOF(st2050_New_Message));
   ```

### Adjusting Cycle Times

Modify cycle times based on your system requirements:

```iecst
Supervisor : FB_Supervisor := (
    CycleTime3002 := T#100MS,  // Supervisor status (default: 100ms)
    CycleTime1026 := T#30MS    // Measurement update (default: 30ms)
);
```

**Important:** Measurement update interval must match Navitrol's `control_interval_us` parameter (default: 30ms ±20% tolerance). Please refer to Navitrol documentation Section 11.1 "Ethernet Supervisor" - Glossary - Measurement update message.

## Additional Resources

### Official Documentation
Refer to your Navitrol documentation package for:
- Complete message specifications (Section 11.1)
- Parameter reference (`supervisor_eth_*` parameters)
- Error code descriptions
- Timing requirements and best practices

### TwinCAT Resources
- [TF6310 TCP/IP function documentation](https://infosys.beckhoff.com/content/1033/tf6310_tc3_tcpip/index.html?id=9025637582166106076)
- [Beckhoff RT Linux®](https://infosys.beckhoff.com/content/1033/beckhoff_rt_linux/index.html?id=1171886970310160181)

### Important Notes
- The Navitrol Ethernet Supervisor Interface protocol is proprietary to Navitec Systems Oy
- Contact Navitec Systems for licensing and commercial use of Navitrol

### Disclaimer

All sample code provided by Beckhoff Automation LLC are for illustrative purposes only and are provided "as is" and without any warranties, express or implied. Actual implementations in applications will vary significantly. Beckhoff Automation LLC shall have no liability for, and does not waive any rights in relation to, any code samples that it provides or the use of such code samples for any purpose.

## Support

For issues related to:
- **This example code:** Open an issue in this repository
- **Navitrol software:** Contact Navitec Systems support
- **TwinCAT/Beckhoff:** Contact Beckhoff Automation support
