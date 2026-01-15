# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Beckhoff TwinCAT 3 PLC project demonstrating TCP/IP communication between an AGV (Automated Guided Vehicle) supervisor system and a control system. The project uses TwinCAT's TF6310 function (TCP/IP communication) to exchange measurement data and motor control commands.

## Technology Stack

- **TwinCAT 3**: Version 3.1.4026.19
- **Target Platform**: TwinCAT OS (x64-E) running on RT Linux
- **Programming Language**: IEC 61131-3 Structured Text (ST)
- **Key Libraries**:
  - Tc2_Standard (v3.3.3.0 / v3.4.5.0)
  - Tc2_System (v3.4.26.0 / v3.9.1.0)
  - Tc2_TcpIp (v3.3.6.0 / v3.4.2.0)
  - Tc2_Utilities
  - Tc3_Module

## Project Structure

```
TF6310_Supervisor_Demo/
└── TF6310_Supervisor_Demo/
    ├── TF6310_Supervisor_Demo.tsproj    # TwinCAT project file
    ├── SuperviserPLC/                   # PLC project
    │   ├── SuperviserPLC.plcproj        # PLC project definition
    │   ├── POUs/                        # Program Organization Units
    │   │   ├── MAIN.TcPOU               # Main program (entry point)
    │   │   ├── FB_AGVSupervisor.TcPOU   # AGV supervisor logic
    │   │   └── FB_TcpIpComms.TcPOU      # TCP/IP communication handler
    │   ├── DUTs/                        # Data type definitions
    │   │   ├── E_MessageIds.TcDUT       # Message ID enumeration
    │   │   └── AGV SupervisorMessages/  # Protocol message structures
    │   └── GVLs/                        # Global Variable Lists
    │       └── TcpIpCommVars.TcGVL      # TCP/IP communication variables
    ├── _Boot/                           # Boot configurations
    └── _Config/                         # Configuration files
```

## Architecture

### Communication Protocol

The system implements a custom binary protocol over TCP/IP with the following message types:

- **Message ID 1023** (`MEASUREMENT_UPDATE_COMBINED`): Sent from PLC to remote system containing sensor data, wheel speeds, drive mode, and PLC status
- **Message ID 1123** (`MOTOR_CONTROL_COMBINED`): Received from remote system with motor control commands

### Message Structure

All messages extend `ST_SupervisorHeader`:
- `nPVersion` (UINT): Protocol version (default: 4)
- `nMessageID` (UINT): Message type identifier
- `nMessageLength` (UDINT): Total message length in bytes

### Core Components

1. **FB_AGVSupervisor**: Main supervisor function block
   - Manages TCP/IP connection lifecycle
   - Orchestrates measurement data transmission (10ms cycle time)
   - Processes received motor control data
   - Instantiated in MAIN with default IP 127.0.0.1:2000

2. **FB_TcpIpComms**: Reusable TCP/IP communication handler
   - State machine (0→10→20→30) for connect, send, receive, disconnect
   - Uses Beckhoff's socket function blocks (FB_SocketConnect, FB_SocketSend, FB_SocketReceive, FB_SocketClose)
   - Supports cyclic message transmission with configurable timing
   - Properties for RemoteHost and RemotePort configuration

3. **Data Structures**: Organized in DUTs/AGV SupervisorMessages/
   - Measurement data with wheel speeds, angles, drive mode, timestamp
   - Status sections for actuators, battery, drive control, safety scanners, errors
   - Motor control commands

## Working with TwinCAT Projects

### Opening the Project

1. Open TwinCAT XAE (Visual Studio with TwinCAT extension)
2. Open solution: `TF6310_Supervisor_Demo/TF6310_Supervisor_Demo.sln`
3. The PLC project loads automatically from `SuperviserPLC.xti`

### Building the Project

- **Build PLC**: Right-click on `SuperviserPLC` project → Build
- **Activate Configuration**: TwinCAT → Activate Configuration
- **Download to Target**: Right-click PLC instance → Download

### File Types

- `.tsproj`: TwinCAT System Manager project (XML-based)
- `.plcproj`: PLC project definition (MSBuild format)
- `.TcPOU`: Program Organization Unit (Function Block, Program, Function)
- `.TcDUT`: Data Unit Type (struct, enum, union definitions)
- `.TcGVL`: Global Variable List
- `.xti`: PLC instance configuration
- `.tmc`: TMC (TwinCAT Module Class) file - auto-generated type information

### System Configuration

- **Target NetId**: 39.63.81.110.1.1
- **CPU Assignment**: Real-time core on CPU 3
- **PLC Task**: PlcTask (Priority 20, Cycle Time 10ms, AMS Port 350)
- **Licenses Required**:
  - TF6310 (TCP/IP function)
  - Additional license GUID: {3EBB9639-5FF3-42B6-8847-35C70DC013C8}

## Development Notes

### Modifying Communication Parameters

To change IP address/port, modify the FB_init call in MAIN.TcPOU:
```iecst
fbSupervisor	: FB_AGVSupervisor(sIPaddr := '192.168.0.18', nPort := 2000);
```

### Adding New Message Types

1. Add enum value to `E_MessageIds.TcDUT`
2. Create new struct in `DUTs/AGV SupervisorMessages/` extending `ST_SupervisorHeader`
3. Add message handling logic in `FB_AGVSupervisor` methods
4. Update message length in `InitTcpIP()` method

### State Machine Flow

FB_TcpIpComms operates with states:
- **0**: Initialize
- **10**: Connect to server
- **20**: Send/Receive messages
- **30**: Disconnect
- **999**: Error/Reset state

### Timing Considerations

- Measurement updates: 10ms cycle time (configurable via `tCyclicTime`)
- Socket operations: 100ms timeout
- Connection timeout: 4 seconds
