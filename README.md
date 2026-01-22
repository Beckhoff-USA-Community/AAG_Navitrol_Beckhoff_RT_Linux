# Navitrol Ethernet Supervisor Interface - TwinCAT 3 Demo

This repository contains a complete TwinCAT 3 PLC example implementation demonstrating TCP/IP communication between an AGV (Automated Guided Vehicle) Supervisor system and the Navitrol navigation system using the **Ethernet Supervisor Interface Protocol Version 4**.

## Overview

This example shows how to implement a Beckhoff PLC-based supervisor that communicates with Navitrol over Ethernet using TwinCAT's TF6310 TCP/IP function. The supervisor acts as a TCP client that connects to Navitrol's positioning and control system, exchanges status information, sends measurement updates, and receives motor control commands for an omni-wheel drive AGV.

### What This Example Demonstrates

- ✅ TCP/IP client connection to Navitrol (TF6310 function blocks)
- ✅ Cyclic status updates (Message 3002 - Supervisor Status)
- ✅ Cyclic measurement updates (Message 1026 - Generic Omni Drive)
- ✅ Position initialization commands (Message 1001)
- ✅ Receiving motor control references (Message 1126)
- ✅ Receiving Navitrol status (Message 3102)
- ✅ Message parsing using union buffers

## Table of Contents

- [Overview](#overview)
- [Repository Structure](#repository-structure)
- [Complete Setup Guide](#complete-setup-guide)
  - [Test Environment Specifications](#test-environment-specifications)
  - [Beckhoff RT Linux Setup](#beckhoff-rt-linux-setup)
  - [Navitrol Monitor Setup (Windows)](#navitrol-monitor-setup-windows)
  - [Opening the TwinCAT Project](#opening-the-twincat-project)
  - [Configuration](#configuration)
  - [Building and Running](#building-and-running)
  - [Debugging and Monitoring](#debugging-and-monitoring)
- [Protocol Implementation](#protocol-implementation)
  - [Message Flow](#message-flow)
  - [Message Structure](#message-structure)
  - [Implemented Messages](#implemented-messages)
  - [Automatic Header Population](#automatic-header-population)
- [Key Features](#key-features)
  - [State Machine Architecture](#1-state-machine-architecture)
  - [Union Buffer Pattern](#2-union-buffer-pattern)
  - [Section-Based Status Messages](#3-section-based-status-messages)
  - [Real-Time Visualization](#4-real-time-visualization)
  - [Event Logging](#5-event-logging)
- [Customization Guide](#customization-guide)
  - [Adding New Message Types](#adding-new-message-types)
  - [Adjusting Cycle Times](#adjusting-cycle-times)
- [Protocol Compliance](#protocol-compliance)
- [Additional Resources](#additional-resources)
- [Support](#support)

## Repository Structure

```
── TF6310_Supervisor_Demo/
   ├── TF6310_Supervisor_Demo.sln          # Visual Studio solution
   └── TF6310_Supervisor_Demo/
       ├── TF6310_Supervisor_Demo.tsproj   # TwinCAT project
       ├── SuperviserPLC/                  # PLC implementation
       │   └── Navitrol Supervisor/
       │       ├── FB_Supervisor.TcPOU     # Main supervisor logic with internal methods
       │       ├── Common types/           # Protocol headers
       │       ├── Supervisor to Navitrol message/  # Outgoing messages
       │       │   ├── F_SetHeaderInfo.TcPOU       # Auto header population
       │       │   ├── 1001/              # Initialize position
       │       │   ├── 1026/              # Measurement update (omni)
       │       │   └── 3002/              # Supervisor status
       │       │           └── Components/Sections/
       │       │               └── F_SetSectionHeaderInfo.TcPOU  # Auto section header
       │       └── Navitrol to Supervisor response/  # Incoming messages
       │           ├── 1100, 1101/        # Position init responses
       │           ├── 1126/              # Motor control commands
       │           └── 3102/              # Navitrol status
       └── Scope Project/                 # Real-time visualization
           └── YT Scope Project.tcscopex  # Scope configuration
── Navitrol Param File/
   └── params.txt                          # Example Navitrol parameters for testing

```

## Complete Setup Guide

This guide provides step-by-step instructions for setting up a complete test environment with Navitrol running on Beckhoff RT Linux and communicating with the TwinCAT supervisor.

**Important Notes:**
- This example implements the **Ethernet Supervisor Interface Protocol Version 4**
- Refer to your Navitrol documentation **Section 11.1 "Ethernet Supervisor"** for complete protocol specifications

### Test Environment Specifications

**Hardware:**
- Test fixture: Beckhoff C6032-0080
  - *Note: Please work with your local Beckhoff team to understand what IPC makes sense for your application*

**Beckhoff RT Linux:**
- Version: 13
- Build: 262191
- Update: 19.12.2025

**Beckhoff Packages (from Beckhoff package server):**
- `tf6310-tcp-ip` version 2.0.22-1
- `tc31-xar-um` version 4026.21.107-1

**Navitec Software (Please contact Navitec to obtain) :** 
- Navitrol 7.1.0-alpha.1.deb (Docker package)
- NavitrolMonitor_1.70.00_Navitrol_7.1.0-alpha.1.zip
- Ethernet Supervisor Documentation dated 17.10.2025

**Development PC:**
- TwinCAT XAE 3.1.4026.20
- TF6310 3.3.23

### Beckhoff RT Linux Setup

#### 1. Install Docker Runtime

Install Docker on Beckhoff RT Linux following the official Docker documentation:

```bash
# Follow Docker documentation: https://docs.docker.com/engine/install/debian/
# Required sections:
# - Set up Docker's apt repository
# - Install the Docker packages
# - Verify installation with hello-world image
```

#### 2. Install TwinCAT Runtime and TF6310

Configure the Beckhoff RT Linux package server and install required packages:

1. [Configure Beckhoff RT Linux Package Server](https://infosys.beckhoff.com/content/1033/beckhoff_rt_linux/17350408843.html)
2. [Install TwinCAT Runtime for Linux](https://infosys.beckhoff.com/content/1033/beckhoff_rt_linux/17350412299.html)

Install the following packages:
```bash
sudo apt-get update
sudo apt-get install tc31-xar-um tf6310-tcp-ip
```

#### 3. Install Navitrol Docker Package

Transfer the Navitrol package from your Windows PC to the Beckhoff RT Linux target (via SSH/SCP or USB), then install:

```bash
# On the Beckhoff RT Linux target:
sudo dpkg -i Navitrol_7.1.0-alpha.1_docker.deb
```

#### 4. Fix Docker Compose File

The container will not start due to file mapping issues. Edit the docker-compose file on the target:

```bash
# On the Beckhoff RT Linux target:
sudo nano /usr/share/navitrol-docker/docker-compose.yml
```

Comment out these two lines:
```yaml
#- /etc/timezone:/etc/timezone:ro
#- /etc/localtime:/etc/localtime:ro
```

#### 5. Install Navitrol License

Transfer the license file `LicenseDetails.json` (obtained from Navitec) to the target, and place it into the license directory:

```bash
/home/navitec/license/
```

#### 6. Configure Navitrol Parameters

Transfer the `params.txt` file from to the target, and place it into the Navitrol directory:

```bash
 /home/navitec/
```

**Required parameters for Ethernet Supervisor interface:**

```
I,supervisor_eth_port,                  2000       # Supervisor communication port (1-65535)
I,supervisor_eth_server,                0          # Server mode: 0=client, 1=server
I,supervisor_eth_endianness,            0          # Endianness: 0=little endian, 1=big endian
I,supervisor_eth_enabled,               1          # Enable Supervisor interface
I,control_interval_us,                  30000      # min 100, max 100000
```

**Note:** A copy of the complete parameter file used for testing is stored in this repository at [`Navitrol Param File/params.txt`](Navitrol%20Param%20File/params.txt).

#### 7. Reboot System

Reboot the Beckhoff RT Linux target to apply all changes:

```bash
# On the Beckhoff RT Linux target:
sudo reboot
```

### Navitrol Monitor Setup (Windows)

Navitrol Monitor is a diagnostic tool for connecting to and monitoring Navitrol systems.

#### Installation Steps

1. **Install Visual C++ Runtime**

   Download and install the latest VC++ redistributable:
   - [Microsoft VC++ Downloads](https://learn.microsoft.com/en-us/cpp/windows/latest-supported-vc-redist?view=msvc-170)
   - Direct download: https://aka.ms/vs/17/release/vc_redist.x64.exe

2. **Extract Navitrol Monitor**

   Extract the zip file to your preferred location:
   ```
   NavitrolMonitor_1.70.00_Navitrol_7.1.0-alpha.1.zip → C:\NavitrolMonitor
   ```

3. **Launch Navitrol Monitor**

   Run `NavitrolMonitor.exe` from the extracted directory.

   - Default password: `ntnavitec`

#### Configure Target Firewall

On the Beckhoff RT Linux target, open firewall port 1234 for Navitrol Monitor:

```bash
# Option 1: Open port 1234 (recommended for production)
sudo firewall-cmd --permanent --add-port=1234/tcp
sudo firewall-cmd --reload

# Option 2: Temporarily disable firewall (testing only)
sudo systemctl stop nftables
sudo systemctl disable nftables
```

### Opening the TwinCAT Project

1. **Install TwinCAT XAE** on your development PC
   - Version 3.1.4026.20 or newer
   - TwinCAT TF6310 (TCP/IP function) version 3.3.23 or newer

2. **Open the solution:**
   - Navigate to: `TF6310_Supervisor_Demo/TF6310_Supervisor_Demo.sln`
   - The PLC project loads automatically from the solution

### Configuration

The default configuration connects to Navitrol running on the same machine (localhost):

```iecst
// In MAIN.TcPOU
Supervisor : FB_Supervisor := (
    RemoteHost := '127.0.0.1',    // Navitrol IPC IP address
    RemotePort := 2000,           // Default Navitrol supervisor port
    CycleTime3002 := T#100MS,     // Supervisor status cycle
    CycleTime1026 := T#30MS       // Measurement update cycle
);
```

### Building and Running

1. **Build the project:**
   - Visual Studio → Build → Build Solution (Ctrl+Shift+B)
   - Verify no compilation errors

2. **Activate configuration:**
   - TwinCAT menu → Activate Configuration
   - System restarts in Run mode

3. **Start the PLC:**
   - TwinCAT menu → PLC → Start

4. **Monitor communication:**
   - Check TwinCAT Event Logger for connection status
   - Use Online View to monitor `FB_Supervisor` variables
   - Open Scope Project to visualize motor control data


### Debugging and Monitoring

**View Navitrol Logs:**

- Using Navitrol Monitor (GUI) - Connect to target and view logs in real-time
- Or connect to target via SSH and use Docker CLI:

```bash
# View last 100 log entries
docker logs navitrol_inside_docker --tail 100

# Follow logs in real-time
docker logs navitrol_inside_docker --follow
```

**Check Docker Container Status:**

```bash
# List running containers
docker ps

# Check Navitrol container health
docker inspect navitrol_inside_docker
```

**Navitrol Web Interface Manual Control UI:**

Open the TwinCAT measurement scope project and start recording data.

Open your web browser and navigate to: `http://[ip address]:8080/`
- Turn on Manual control, and use the mouse to drag around the virtual joystick 

## Protocol Implementation

### Message Flow

**Startup Sequence:**
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

### Message Structure

All messages follow Protocol Version 4 structure:

```
[Header: 8 bytes]
├─ Protocol_version (UINT, 2 bytes): Always 4
├─ Message_ID (UINT, 2 bytes): Message type identifier
└─ Message_length (UDINT, 4 bytes): Total message size in bytes
[Payload: Variable length]
```

**Naming Convention:** Message structs are named `ST_<MessageID>_<Description>`

Example: `ST_3002_Supervisor_Status`, `ST_1026_Measurement_update_generic_omni_drive`

### Implemented Messages

| ID   | Direction          | Name                                  | Purpose                                   | Cycle Time |
|------|--------------------|------------------------------------- |-------------------------------------------|------------|
| 1001 | Supervisor → Nav   | Initialize Position                  | Set AGV position on map                   | On demand  |
| 1026 | Supervisor → Nav   | Measurement Update (Generic Omni)    | Send odometry, receive control refs       | 30ms       |
| 3002 | Supervisor → Nav   | Supervisor Status                    | Send supervisor state & requests          | 100ms      |
| 1100 | Nav → Supervisor   | Position Initialization Started      | Acknowledge init request                  | Response   |
| 1101 | Nav → Supervisor   | Position Initialized                 | Init result with confidence               | Response   |
| 1126 | Nav → Supervisor   | Motor Control (Generic Omni)         | Wheel speed & angle references            | Response   |
| 3102 | Nav → Supervisor   | Navitrol Status                      | Navitrol state, errors, warnings          | Response   |

### Automatic Header Population

The `F_SetHeaderInfo()` function automatically populates message headers:

```iecst
// Usage example:
F_SetHeaderInfo(st3002_Supervisor_Status);
// Automatically sets:
//   - Protocol_version := 4
//   - Message_ID := 3002 (parsed from struct name "ST_3002_...")
//   - Message_length := SIZEOF(st3002_Supervisor_Status)
```

The `F_SetSectionHeaderInfo()` function automatically populates section headers for message 3002:

```iecst
// Usage example for 3002 message sections:
F_SetSectionHeaderInfo(st3002_2_Manual_section);
// Automatically sets:
//   - Section_ID := 2 (parsed from struct name "ST_3002_2_...")
//   - Section_length := SIZEOF(st3002_2_Manual_section)
```

These functions eliminate manual header management and reduce errors.

## Key Features

### 1. State Machine Architecture

`FB_Supervisor` implements a robust 5-state connection manager:

```
INIT_FBs → CLOSE_OLD_SOCKETS → CONNECT_TO_Server → COMS_ACTIVE ⇄ CLOSE_CONNECTION
                                                          ↓
                                                      (on error)
```

- **INIT_FBs**: Initialize function blocks, wait for Execute
- **CLOSE_OLD_SOCKETS**: Clean up previous connections
- **CONNECT_TO_Server**: Establish TCP connection (4s timeout)
- **COMS_ACTIVE**: Main communication loop, organized into internal methods:
  - `SendMsg3002()`: Cyclic supervisor status transmission (timer-based)
  - `SendMsg1026()`: Cyclic measurement updates (timer-based, increments Message_number)
  - `SendMsg1001()`: Position initialization on rising edge trigger
  - `ReciveAllMsgsAndParse()`: Continuous receive and message parsing (non-blocking)
- **CLOSE_CONNECTION**: Graceful disconnect

### 2. Union Buffer Pattern

Incoming messages are received into a union buffer that acts as a raw byte array:

```iecst
// Bytes bucket to buffer received messages.
// Header info is parsed before being copied into the correct message structure.
// PLC logic is built expecting to receive the entire TCP message payload with one call to FB_SocketReceive.
// FB_SocketReceive will fragment the received data if the receive buffer is not large enough.
// Make sure this is large enough to fit the biggest received message.
TYPE U_Recived_bytes_from_Navitrol : UNION
    Header : ST_SupervisorHeader;
    Bytes  : ARRAY[0..Param_Supervisor_response.RecivedBytesBufferSize] OF UINT;
END_UNION
END_TYPE
```

**How it works:**
1. Raw TCP data is received into the `Bytes` array
2. The `Header` overlay allows reading the message type without copying
3. Message is identified by `Header.Message_ID`
4. Full message is copied to the appropriate typed output variable using `MEMCPY`

This pattern ensures the receive buffer is large enough for any message type while maintaining type safety for processed messages.

### 3. Section-Based Status Messages

Large status messages (3002, 3102) use a sectioned architecture:
- Each section has an `ST_SectionHeader` with `Section_ID` and `Section_length`
- Sections can be requested/omitted as needed
- Example sections in 3002: Manual (ID 2), Remote control (ID 6), Battery (ID 8)
- Section naming convention: `ST_3002_<SectionID>_<Description>_section`
- Use `F_SetSectionHeaderInfo()` to automatically populate section headers

### 4. Real-Time Visualization

The included Scope Project monitors motor control commands in real-time:
- Wheel 0 & 1 steering angles
- Wheel 0 & 1 speed references
- 10ms sampling synchronized with PLC task
- Useful for debugging control loops

### 5. Event Logging

Built-in diagnostic logging via TwinCAT events:
- **ErrorMsg**: Connection failures, socket errors, receive buffer overflow
- **VerboseMsg**: Successful connections, position init events
- **WarnMsg**: Unknown messages received (with message ID and hex dump)

View logs in TwinCAT Event Logger.

## Customization Guide

### Adding New Message Types

Follow the established pattern to add new Navitrol messages:

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
   - Call `F_SetHeaderInfo(message)` before sending
   - Add send logic in `FB_Supervisor` COMS_ACTIVE state (or create new internal METHOD following the pattern of `SendMsg3002()`, `SendMsg1026()`, `SendMsg1001()`)
   - Create timer if cyclic, or event trigger if on-demand

4. **For incoming messages:**
   - Add CASE handler in `ReciveAllMsgsAndParse()` METHOD in FB_Supervisor
   - Add output variable to FB_Supervisor for storing the received message
   - Use `MEMCPY` to copy from `ReceiveBuffer` union to the typed output variable
   - Note: The union buffer is just a byte array with header overlay - you don't add message types to it

   Example pattern:
   ```iecst
   // In ReciveAllMsgsAndParse() method:
   CASE ReceiveBuffer.Header.Message_ID OF
       // ... existing cases ...
       2050:
           NumOf2050MsgRecived := NumOf2050MsgRecived + 1;
           MEMCPY(destAddr := ADR(st2050_New_Message),
                  srcAddr := ADR(ReceiveBuffer),
                  n := SIZEOF(st2050_New_Message));
   ```

### Adjusting Cycle Times

Modify cycle times based on your system requirements:

```iecst
Supervisor : FB_Supervisor := (
    CycleTime3002 := T#100MS,  // Supervisor status (default: 100ms)
    CycleTime1026 := T#30MS    // Measurement update (default: 30ms, recommended: 30±10ms)
);
```

**Important:** Measurement update interval must match Navitrol's `control_interval_us` parameter (default: 30ms ±20% tolerance). Please refer to Navitrol documentation Section 11.1 "Ethernet Supervisor" - Glossary - Measurement update message.


## Protocol Compliance

This implementation follows the **Ethernet Supervisor Interface Protocol Version 4** as specified in:
- **Section 11.1 "Ethernet Supervisor"** of the Navitrol User Instructions


## Additional Resources

### Official Documentation
Refer to your Navitrol documentation package for:
- Complete message specifications (Section 11.1)
- Parameter reference (`supervisor_eth_*` parameters)
- Error code descriptions
- Timing requirements and best practices

### TwinCAT Resources
- [Beckhoff Information System](https://infosys.beckhoff.com/)
- [TF6310 TCP/IP function documentation](https://infosys.beckhoff.com/content/1033/tf6310_tc3_tcpip/index.html?id=9025637582166106076)

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
