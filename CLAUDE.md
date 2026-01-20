# CLAUDE.md - QGroundControl Codebase Guide

This document helps AI assistants understand how QGroundControl communicates with drones via MAVLink and how to trace message flows for any workflow.

## Project Overview

**QGroundControl (QGC)** is a ground control station for MAVLink-enabled UAVs. It communicates with PX4 and ArduPilot autopilots using the MAVLink protocol.

- **Language**: C++20 with Qt 6.10+
- **UI Framework**: Qt Quick/QML
- **Build System**: CMake 3.25+
- **Protocol**: MAVLink v2

## Code Organization

### Directory Structure

```
src/
├── Comms/                    # Communication layer
│   ├── MAVLinkProtocol.cc    # MAVLink parsing & dispatch (START HERE for message flow)
│   ├── LinkInterface.h       # Abstract link (Serial, UDP, TCP)
│   └── LinkManager.cc        # Link lifecycle management
│
├── Vehicle/                  # Vehicle state & commands
│   ├── Vehicle.cc            # Main vehicle class - ALL message handlers here
│   ├── Vehicle.h             # ~1200 lines - core vehicle interface
│   ├── VehicleLinkManager.cc # Per-vehicle link management
│   ├── MultiVehicleManager.cc# Vehicle discovery via HEARTBEAT
│   └── InitialConnectStateMachine.cc # Connection sequence
│
├── FactSystem/               # Parameter system
│   ├── ParameterManager.cc   # PARAM_* message handling
│   ├── Fact.h                # Single parameter with metadata
│   └── FactGroup.h           # Telemetry data groups
│
├── MissionManager/           # Mission/waypoint protocol
│   ├── PlanManager.cc        # MISSION_* message protocol
│   ├── MissionManager.cc     # Mission-specific handling
│   ├── GeoFenceManager.cc    # Geofence protocol
│   └── RallyPointManager.cc  # Rally point protocol
│
├── FirmwarePlugin/           # Firmware-specific behavior
│   ├── FirmwarePlugin.h      # Base class - virtual methods
│   ├── APM/                   # ArduPilot implementations
│   │   └── APMFirmwarePlugin.cc
│   └── PX4/                   # PX4 implementations
│       └── PX4FirmwarePlugin.cc
│
├── AutoPilotPlugins/         # Vehicle setup & calibration
│   ├── APM/
│   │   └── APMSensorsComponentController.cc  # APM calibration
│   └── PX4/
│       └── SensorsComponentController.cc     # PX4 calibration
│
└── FlyView/                  # Flight UI
    └── GuidedActionsController.qml  # Takeoff, Land, RTL triggers
```

## MAVLink Message Flow Architecture

### Message Reception Path

```
Network/Serial Data
    ↓
MAVLinkProtocol::receiveBytes()        [src/Comms/MAVLinkProtocol.cc:88]
    ↓ mavlink_parse_char()
emit messageReceived(link, message)    [src/Comms/MAVLinkProtocol.cc:254]
    ↓
Vehicle::_mavlinkMessageReceived()     [src/Vehicle/Vehicle.cc:424]
    ↓ Dispatch by message ID
Specific handlers (see below)
```

### Message Dispatch in Vehicle.cc (lines 424-626)

The `_mavlinkMessageReceived()` method dispatches to:

1. **VehicleLinkManager** (line 434) - Heartbeat tracking
2. **FirmwarePlugin** (line 463) - Firmware-specific filtering
3. **ParameterManager** (line 476) - PARAM_VALUE handling
4. **FTPManager** (line 475) - File transfer protocol
5. **FactGroups** (lines 487-491) - Telemetry data
6. **Switch statement** (lines 493-621) - Message-specific handlers

### Message Sending Path

```
Vehicle::sendMavCommand()              [src/Vehicle/Vehicle.cc:2280]
    ↓
Vehicle::_sendMavCommandWorker()       [src/Vehicle/Vehicle.cc:2496]
    ↓
Vehicle::_sendMavCommandFromList()     [src/Vehicle/Vehicle.cc:2564]
    ↓ mavlink_msg_command_long_encode_chan()
Vehicle::sendMessageOnLinkThreadSafe() [src/Vehicle/Vehicle.cc:1425]
    ↓
LinkInterface::writeBytesThreadSafe()
```

---

## Workflow: Vehicle Connection

### Discovery via HEARTBEAT

```
Vehicle sends HEARTBEAT (1 Hz)
    ↓
MAVLinkProtocol::receiveBytes()                    [MAVLinkProtocol.cc:88]
    ↓
emit vehicleHeartbeatInfo()                        [MAVLinkProtocol.cc:224]
    ↓
MultiVehicleManager::_vehicleHeartbeatInfo()       [MultiVehicleManager.cc:68]
    ↓ Creates Vehicle object
Vehicle::_handleHeartbeat()                        [Vehicle.cc:1257]
    ↓ Extracts armed state, flight mode
```

### Initial Connection State Machine

**File**: `src/Vehicle/InitialConnectStateMachine.cc`

```
State 1: _stateRequestAutopilotVersion    → REQUEST_MESSAGE(AUTOPILOT_VERSION)
State 2: _stateRequestStandardModes       → Request flight modes
State 3: _stateRequestCompInfo            → Component information
State 4: _stateRequestParameters          → PARAM_REQUEST_LIST
State 5: _stateRequestMission             → MISSION_REQUEST_LIST
State 6: _stateRequestGeoFence            → MISSION_REQUEST_LIST (fence)
State 7: _stateRequestRallyPoints         → MISSION_REQUEST_LIST (rally)
State 8: _stateSignalInitialConnectComplete
```

**Key Files**:
- `src/Vehicle/InitialConnectStateMachine.cc` - State definitions
- `src/Vehicle/Vehicle.cc:152-158` - State machine initialization

---

## Workflow: Parameter Read/Write

### Initial Parameter Download

**File**: `src/FactSystem/ParameterManager.cc`

```
GCS → Vehicle: PARAM_REQUEST_LIST                  [ParameterManager.cc:590-597]
Vehicle → GCS: PARAM_VALUE (repeated for each param)
    ↓
ParameterManager::mavlinkMessageReceived()         [ParameterManager.cc:85]
    ↓
ParameterManager::_handleParamValue()              [ParameterManager.cc:112]
    ↓ Creates Fact objects, updates _mapCompId2FactMap
emit parametersReadyChanged(true)                  [ParameterManager.cc:1190]
```

### Write Single Parameter

```
User changes value in UI
    ↓
Fact::setCookedValue()
    ↓
ParameterManager::_factRawValueUpdated()           [ParameterManager.cc:474]
    ↓
ParameterManager::_mavlinkParamSet()               [ParameterManager.cc:277]
    ↓
GCS → Vehicle: PARAM_SET                           [ParameterManager.cc:290]
Vehicle → GCS: PARAM_VALUE (echo back)
    ↓
State machine validates response                   [ParameterManager.cc:339-402]
```

**MAVLink Messages**:
- `PARAM_REQUEST_LIST` - Request all parameters
- `PARAM_VALUE` - Parameter value (response)
- `PARAM_SET` - Set parameter value
- `PARAM_REQUEST_READ` - Request single parameter

**Key Files**:
- `src/FactSystem/ParameterManager.cc` - All parameter protocol
- `src/FactSystem/Fact.h` - Parameter data structure

---

## Workflow: Mission Upload/Download

### Mission Download

**File**: `src/MissionManager/PlanManager.cc`

```
GCS → Vehicle: MISSION_REQUEST_LIST                [PlanManager.cc:152-158]
Vehicle → GCS: MISSION_COUNT (count=N)
    ↓
PlanManager::_handleMissionCount()                 [PlanManager.cc:312]
    ↓
GCS → Vehicle: MISSION_REQUEST_INT (seq=0)         [PlanManager.cc:358-365]
Vehicle → GCS: MISSION_ITEM_INT (seq=0)
    ↓
PlanManager::_handleMissionItem()                  [PlanManager.cc:371]
    ↓ (repeat for seq=1 to N-1)
GCS → Vehicle: MISSION_ACK (ACCEPTED)              [PlanManager.cc:294-304]
```

### Mission Upload

```
User clicks "Upload" in Plan View
    ↓
PlanManager::writeMissionItems()                   [PlanManager.cc:56]
    ↓
GCS → Vehicle: MISSION_COUNT (count=N)             [PlanManager.cc:105-115]
Vehicle → GCS: MISSION_REQUEST_INT (seq=0)
    ↓
PlanManager::_handleMissionRequest()               [PlanManager.cc:482]
    ↓
GCS → Vehicle: MISSION_ITEM_INT (seq=0)            [PlanManager.cc:527-545]
    ↓ (repeat for each item vehicle requests)
Vehicle → GCS: MISSION_ACK (ACCEPTED)
    ↓
PlanManager::_handleMissionAck()                   [PlanManager.cc:551]
```

**MAVLink Messages**:
- `MISSION_REQUEST_LIST` - Start download
- `MISSION_COUNT` - Number of items
- `MISSION_REQUEST_INT` - Request specific item
- `MISSION_ITEM_INT` - Mission item data
- `MISSION_ACK` - Success/failure acknowledgment
- `MISSION_CLEAR_ALL` - Delete all items

**Key Files**:
- `src/MissionManager/PlanManager.cc` - Core protocol (963 lines)
- `src/MissionManager/MissionManager.cc` - Mission-specific logic
- `src/Comms/MockLink/MockLinkMissionItemHandler.cc` - Test simulation

---

## Workflow: Arming/Disarming

**File**: `src/Vehicle/Vehicle.cc`

```
QML: vehicle.armed = true
    ↓
Vehicle::setArmed(true)                            [Vehicle.cc:1475]
    ↓
Vehicle::sendMavCommand(MAV_CMD_COMPONENT_ARM_DISARM, 1.0f)  [Vehicle.cc:1480-1484]
    ↓
GCS → Vehicle: COMMAND_LONG (cmd=400, p1=1)        [Vehicle.cc:2638-2642]
Vehicle → GCS: COMMAND_ACK (result)
    ↓
Vehicle::_handleCommandAck()                       [Vehicle.cc:2695]
    ↓
Vehicle::_handleHeartbeat() detects armed flag     [Vehicle.cc:1272]
    ↓
emit armedChanged(true)                            [Vehicle.cc:1059]
```

**MAVLink Messages**:
- `COMMAND_LONG` with `MAV_CMD_COMPONENT_ARM_DISARM`
  - param1 = 1.0 (arm) or 0.0 (disarm)
  - param2 = 2989 (force arm, optional)
- `COMMAND_ACK` - Command result

**Key Files**:
- `src/Vehicle/Vehicle.cc:1475-1491` - Arm/disarm methods

---

## Workflow: Flight Mode Change

**File**: `src/Vehicle/Vehicle.cc`

```
QML: vehicle.flightMode = "Stabilize"
    ↓
Vehicle::setFlightMode()                           [Vehicle.cc:1514]
    ↓
FirmwarePlugin::setFlightMode() → base_mode, custom_mode
    ↓
Vehicle::sendMavCommand(MAV_CMD_DO_SET_MODE)       [Vehicle.cc:1532-1537]
    ↓
GCS → Vehicle: COMMAND_LONG (cmd=176)
Vehicle → GCS: COMMAND_ACK
Vehicle → GCS: HEARTBEAT (with new mode)
    ↓
Vehicle::_handleHeartbeat()                        [Vehicle.cc:1282-1294]
    ↓
emit flightModeChanged()
```

**Alternative**: `SET_MODE` message (legacy, line 1539-1547)

**Key Files**:
- `src/Vehicle/Vehicle.cc:1514-1552` - Flight mode change
- `src/FirmwarePlugin/*/` - Mode name to ID mapping

---

## Workflow: Guided Flight Commands

### Takeoff

**Files**: `src/Vehicle/Vehicle.cc`, `src/FirmwarePlugin/*/`

```
QML: GuidedActionsController actionTakeoff         [GuidedActionsController.qml:562]
    ↓
Vehicle::guidedModeTakeoff(altitude)               [Vehicle.cc:1967]
    ↓
FirmwarePlugin::guidedModeTakeoff()
    ↓
APM: Sets GUIDED mode, arms, then:
GCS → Vehicle: COMMAND_LONG (MAV_CMD_NAV_TAKEOFF)  [APMFirmwarePlugin.cc:1016-1022]

PX4: Sends takeoff command, arms on ACK:
GCS → Vehicle: COMMAND_LONG (MAV_CMD_NAV_TAKEOFF)  [PX4FirmwarePlugin.cc:323-342]
```

### Land / RTL

```
Vehicle::guidedModeLand()                          [Vehicle.cc:1958]
Vehicle::guidedModeRTL()                           [Vehicle.cc:1949]
    ↓
FirmwarePlugin sets flight mode to LAND or RTL
```

### Go-To Location

```
Vehicle::guidedModeGotoLocation(coord)             [Vehicle.cc:2014]
    ↓
FirmwarePlugin::guidedModeGotoLocation()
    ↓
GCS → Vehicle: COMMAND_INT (MAV_CMD_DO_REPOSITION) [APMFirmwarePlugin.cc:809-827]
  - Latitude, Longitude (in param5, param6 as int32)
  - Altitude (param7)
```

### Pause

```
Vehicle::pauseVehicle()                            [Vehicle.cc:2181]
    ↓
APM: Sets BRAKE/LOITER flight mode                 [APMFirmwarePlugin.cc:767-770]
PX4: Sends MAV_CMD_DO_REPOSITION with NAN coords   [PX4FirmwarePlugin.cc:277-289]
```

**Key Files**:
- `src/FlyView/GuidedActionsController.qml` - UI triggers
- `src/FirmwarePlugin/APM/APMFirmwarePlugin.cc:795-907` - APM guided
- `src/FirmwarePlugin/PX4/PX4FirmwarePlugin.cc:277-423` - PX4 guided

---

## Workflow: Sensor Calibration

### Calibration Command

**File**: `src/Vehicle/Vehicle.cc`

```
Controller calls Vehicle::startCalibration(type)   [Vehicle.cc:2985]
    ↓
GCS → Vehicle: COMMAND_LONG (MAV_CMD_PREFLIGHT_CALIBRATION)
    - param1=1: Gyro
    - param2=1: Magnetometer
    - param3=1: Pressure (PX4)
    - param5=1: Accelerometer
    - param5=2: Level horizon
    - param6=1: Airspeed
    - param7=1: ESC
```

### Progress Monitoring

**PX4** (text-based): `src/AutoPilotPlugins/PX4/SensorsComponentController.cc`
```
Vehicle → GCS: STATUSTEXT messages
    ↓
_handleUASTextMessage()                            [SensorsComponentController.cc:215]
    ↓
Parses: "[cal] calibration started", "progress <N>", "orientation detected"
```

**APM** (binary): `src/AutoPilotPlugins/APM/APMSensorsComponentController.cc`
```
Compass calibration:
GCS → Vehicle: MAV_CMD_DO_START_MAG_CAL            [APMSensorsComponentController.cc:216]
Vehicle → GCS: MAG_CAL_PROGRESS                    [APMSensorsComponentController.cc:489]
Vehicle → GCS: MAG_CAL_REPORT                      [APMSensorsComponentController.cc:521]

Accelerometer calibration:
Vehicle → GCS: COMMAND_LONG (MAV_CMD_ACCELCAL_VEHICLE_POS)
    ↓
_mavlinkMessageReceived()                          [APMSensorsComponentController.cc:641]
```

**Key Files**:
- `src/Vehicle/Vehicle.cc:2985-3046` - Calibration command encoding
- `src/AutoPilotPlugins/PX4/SensorsComponentController.cc` - PX4 calibration
- `src/AutoPilotPlugins/APM/APMSensorsComponentController.cc` - APM calibration

---

## How to Trace Any Workflow

### 1. Find the Entry Point

- **QML UI action**: Search `src/FlyView/` or `src/QmlControls/` for button/action
- **Vehicle method**: Check `src/Vehicle/Vehicle.h` for Q_INVOKABLE methods
- **C++ command**: Search for `sendMavCommand` or `sendMessageOnLinkThreadSafe`

### 2. Follow the Message Chain

```cpp
// Sending: Look for these patterns
vehicle->sendMavCommand(compId, MAV_CMD_*, ...)
vehicle->sendMavCommandInt(compId, MAV_CMD_*, ...)
mavlink_msg_*_pack_chan(..., &msg, ...)
sendMessageOnLinkThreadSafe(link, msg)

// Receiving: Check Vehicle::_mavlinkMessageReceived()
case MAVLINK_MSG_ID_*:
    _handle*();
    break;
```

### 3. Check Firmware-Specific Behavior

Many commands differ between PX4 and APM:
```cpp
// In Vehicle.cc - delegates to firmware plugin
_firmwarePlugin->guidedModeTakeoff(this, altitude);

// Check both:
// src/FirmwarePlugin/APM/APMFirmwarePlugin.cc
// src/FirmwarePlugin/PX4/PX4FirmwarePlugin.cc
```

### 4. Understand the State Machine Pattern

Many protocols use state machines with timeouts:
```cpp
// Common pattern in ParameterManager, PlanManager
_startAckTimeout(AckType);           // Start waiting
_waitingParamTimeout();              // Timeout handler
_handleSomeMessage();                // Message received
```

---

## Key MAVLink Message Reference

| Workflow | GCS → Vehicle | Vehicle → GCS |
|----------|---------------|---------------|
| **Connect** | - | HEARTBEAT (1 Hz) |
| **Sync Info** | REQUEST_MESSAGE | AUTOPILOT_VERSION |
| **Parameters** | PARAM_REQUEST_LIST | PARAM_VALUE |
| **Set Param** | PARAM_SET | PARAM_VALUE |
| **Mission Download** | MISSION_REQUEST_LIST → MISSION_REQUEST_INT | MISSION_COUNT → MISSION_ITEM_INT |
| **Mission Upload** | MISSION_COUNT → MISSION_ITEM_INT | MISSION_REQUEST_INT → MISSION_ACK |
| **Arm/Disarm** | COMMAND_LONG (400) | COMMAND_ACK |
| **Flight Mode** | COMMAND_LONG (176) or SET_MODE | COMMAND_ACK |
| **Takeoff** | COMMAND_LONG (22) | COMMAND_ACK |
| **Go-To** | COMMAND_INT (192) | COMMAND_ACK |
| **Calibration** | COMMAND_LONG (241) | COMMAND_ACK, STATUSTEXT |

---

## Build Commands

```bash
make configure   # Configure CMake build
make build       # Build the project
make test        # Run unit tests
make lint        # Run pre-commit checks
make run         # Launch QGroundControl
```

## Essential Files for Understanding Message Flow

| Purpose | File | Key Lines |
|---------|------|-----------|
| **Message parsing** | `src/Comms/MAVLinkProtocol.cc` | 88-124 |
| **Message dispatch** | `src/Vehicle/Vehicle.cc` | 424-626 |
| **Command sending** | `src/Vehicle/Vehicle.cc` | 2280-2646 |
| **Parameter protocol** | `src/FactSystem/ParameterManager.cc` | 85-402 |
| **Mission protocol** | `src/MissionManager/PlanManager.cc` | 56-606 |
| **Vehicle discovery** | `src/Vehicle/MultiVehicleManager.cc` | 68-138 |
| **APM specifics** | `src/FirmwarePlugin/APM/APMFirmwarePlugin.cc` | - |
| **PX4 specifics** | `src/FirmwarePlugin/PX4/PX4FirmwarePlugin.cc` | - |

## Coding Conventions

- **Private members**: `_leadingUnderscore`
- **Classes**: `PascalCase`
- **Methods**: `camelCase`
- **Always null-check**: `activeVehicle()` before use
- **Wait for**: `parametersReady` signal before accessing Facts
- **Use FirmwarePlugin**: for firmware-specific behavior

## Additional Resources

- **MAVLink Protocol**: https://mavlink.io/
- **Developer Guide**: https://dev.qgroundcontrol.com/en/
- **Qt 6 Documentation**: https://doc.qt.io/qt-6/
