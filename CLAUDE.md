# CLAUDE.md - QGroundControl AI Assistant Guide

This document provides essential context for AI assistants working with the QGroundControl codebase.

## Project Overview

**QGroundControl (QGC)** is a ground control station for MAVLink-enabled UAVs (drones). It provides flight control, mission planning, and vehicle configuration for PX4 and ArduPilot autopilot systems.

- **Language**: C++20 with Qt 6.10+
- **UI Framework**: Qt Quick/QML
- **Build System**: CMake 3.25+
- **Protocol**: MAVLink
- **Platforms**: Windows, macOS, Linux, Android, iOS
- **License**: Dual-licensed (Apache 2.0 AND GPL v3)

## Repository Structure

```
qgroundcontrol/
├── src/                      # Main source code
│   ├── Vehicle/              # Vehicle state, communication, telemetry
│   ├── FactSystem/           # Parameter system (CRITICAL - see below)
│   ├── FirmwarePlugin/       # PX4/ArduPilot abstraction layer
│   ├── AutoPilotPlugins/     # Vehicle setup UI per firmware
│   ├── MissionManager/       # Mission planning and execution
│   ├── MAVLink/              # MAVLink protocol handling
│   ├── QmlControls/          # Reusable QML UI components
│   ├── Comms/                # Communication links (Serial, UDP, TCP)
│   ├── Settings/             # Application settings
│   ├── FlyView/              # Flight view UI
│   ├── Camera/               # Camera control
│   ├── Gimbal/               # Gimbal control
│   ├── GPS/                  # GPS handling
│   ├── Joystick/             # Joystick input
│   ├── Terrain/              # Terrain data
│   ├── UI/                   # Main window and top-level UI
│   ├── Utilities/            # Helper classes
│   └── QGCApplication.cc     # Application entry point
├── test/                     # Unit tests (mirrors src/ structure)
├── tools/                    # Development scripts and utilities
├── cmake/                    # CMake modules and platform configs
├── resources/                # Images, icons, QML resources
├── translations/             # Internationalization files
├── docs/                     # Documentation source
├── deploy/                   # Deployment scripts and Docker
└── custom-example/           # Example custom build template
```

## Critical Architecture Patterns

### 1. Fact System (MOST IMPORTANT)

ALL vehicle parameters use the Fact System. **Never create custom parameter storage.**

```cpp
// Access parameters - ALWAYS null-check!
Fact* param = vehicle->parameterManager()->getParameter(-1, "PARAM_NAME");
if (param && param->validate(newValue, false).isEmpty()) {
    param->setCookedValue(newValue);  // Use cookedValue for UI (with units)
    // param->rawValue() for MAVLink/storage
}
```

**Key classes:**
- `Fact` - Single parameter with validation, units, metadata (`src/FactSystem/Fact.h`)
- `FactGroup` - Hierarchical container for telemetry (handles MAVLink via `handleMessage()`)
- `FactMetaData` - JSON-based metadata (min/max, enums, descriptions)

**Rules:**
- Wait for `parametersReady` signal before accessing parameters
- Use `cookedValue` for display (user units), `rawValue` for storage/MAVLink
- Metadata defined in `*.FactMetaData.json` files

### 2. Multi-Vehicle Support

**ALWAYS null-check the active vehicle:**

```cpp
Vehicle* vehicle = MultiVehicleManager::instance()->activeVehicle();
if (!vehicle) {
    qCWarning(MyLog) << "No active vehicle";
    return;
}
```

```qml
// QML vehicle access
property var _activeVehicle: QGroundControl.multiVehicleManager.activeVehicle
enabled: _activeVehicle && _activeVehicle.armed
```

### 3. Firmware Plugin System

Use FirmwarePlugin for ALL firmware-specific behavior:

```cpp
// Get firmware-specific capabilities
vehicle->firmwarePlugin()->flightModes();
vehicle->firmwarePlugin()->isCapable(capability);

// Key classes:
// FirmwarePlugin - Base class for firmware behavior
// PX4FirmwarePlugin - PX4-specific implementation
// ArduCopterFirmwarePlugin - ArduPilot Copter implementation
```

### 4. QML/C++ Integration

```cpp
// Expose C++ to QML using Qt6 macros
class MyClass : public QObject
{
    Q_OBJECT
    QML_ELEMENT                    // Creatable in QML
    QML_SINGLETON                  // Singleton pattern
    QML_UNCREATABLE("")            // C++-only instantiation

    Q_MOC_INCLUDE("Vehicle.h")     // For forward-declared types

    Q_PROPERTY(int value READ value WRITE setValue NOTIFY valueChanged)
    Q_INVOKABLE void doSomething();
    Q_ENUM(EnumType)
};
```

## Build Commands

### Using Make (recommended)

```bash
make help        # Show all commands
make deps        # Install system dependencies (Debian/Ubuntu)
make configure   # Configure CMake build (Debug)
make build       # Build the project
make test        # Run unit tests
make lint        # Run pre-commit checks
make check       # Run lint + test
make run         # Launch QGroundControl
make clean       # Remove build directory
```

### Using just (alternative)

```bash
just             # Show all commands
just setup       # Full setup: deps, submodules, configure, build
just check       # Run lint + test
```

### Direct CMake

```bash
# Configure
~/Qt/6.10.1/gcc_64/bin/qt-cmake -B build -G Ninja \
    -DCMAKE_BUILD_TYPE=Debug \
    -DQGC_BUILD_TESTING=ON

# Build
cmake --build build --parallel

# Run tests
cd build && ctest --output-on-failure
```

## Testing

### Run Unit Tests

```bash
# Via make/just
make test
just test

# Via ctest
cd build && ctest --output-on-failure

# Run specific test
./build/staging/QGroundControl --unittest:FactSystemTestGeneric

# Run all tests
./build/staging/QGroundControl --unittest
```

### Test Structure

Tests are in `test/` directory, mirroring `src/` structure. Use the Qt Test framework with `UnitTest` base class.

## Coding Conventions

### Naming

| Element | Convention | Example |
|---------|------------|---------|
| Classes | PascalCase | `VehicleManager` |
| Methods/Functions | camelCase | `getActiveVehicle()` |
| Variables | camelCase | `activeVehicle` |
| Private members | _leadingUnderscore | `_vehicleList` |
| Constants | UPPER_SNAKE_CASE | `MAX_RETRY_COUNT` |
| Files | ClassName.h/.cc | `Vehicle.h`, `Vehicle.cc` |

### Formatting

- **Indentation**: 4 spaces (no tabs)
- **Line length**: 120 characters max
- **Braces**: Allman style for functions/classes, K&R for control statements
- Run `clang-format` before committing

### QML Guidelines

- **No hardcoded sizes**: Use `ScreenTools.defaultFontPixelHeight/Width`
- **No hardcoded colors**: Use `QGCPalette` for theming
- **Use QGC controls**: `QGCButton`, `QGCLabel`, `QGCTextField`
- **Translations**: Wrap user-visible strings with `qsTr()`
- **Connections**: Use Qt6 function syntax:

```qml
// CORRECT - Qt6 function syntax
Connections {
    target: vehicle
    function onArmedChanged() { }
}

// DEPRECATED - Old Qt5 syntax
Connections {
    target: vehicle
    onArmedChanged: { }  // Don't use this
}
```

### Logging

```cpp
// Declare in header
Q_DECLARE_LOGGING_CATEGORY(MyComponentLog)

// Define in source
QGC_LOGGING_CATEGORY(MyComponentLog, "qgc.component.name")

// Use categorized logging
qCDebug(MyComponentLog) << "Debug message";
qCWarning(MyComponentLog) << "Warning message";
```

## Key Files to Read First

1. **`src/FactSystem/Fact.h`** - Parameter system foundation
2. **`src/Vehicle/Vehicle.h`** - Core vehicle model
3. **`src/FirmwarePlugin/FirmwarePlugin.h`** - Firmware abstraction
4. **`CODING_STYLE.md`** - Complete coding style guide
5. **`tools/coding-style/`** - Reference implementations

## Common Pitfalls

1. **Assuming single vehicle** - Always null-check `activeVehicle()`
2. **Accessing Facts before ready** - Wait for `parametersReady` signal
3. **Bypassing FirmwarePlugin** - Always use plugin for firmware-specific behavior
4. **Using Q_ASSERT in production** - Use defensive checks instead (Q_ASSERT is removed in release builds)
5. **Mixing cookedValue/rawValue** - `cookedValue` for UI, `rawValue` for MAVLink
6. **Hardcoded QML sizes/colors** - Use ScreenTools and QGCPalette
7. **Old Connections syntax** - Use Qt6 function syntax in QML

## Pre-commit Hooks

The repository uses pre-commit for code quality. Install and run:

```bash
# Install pre-commit
pip install pre-commit
pre-commit install

# Run all checks
pre-commit run --all-files

# Or use make/just
make lint
```

### Checks include:
- clang-format (C++ formatting)
- clang-tidy (C++ static analysis)
- qmllint (QML linting)
- vehicle-null-check (QGC-specific null safety)
- typos (spell checking)
- shellcheck (shell scripts)

## Development Tools

See `tools/README.md` for comprehensive documentation. Key tools:

| Tool | Purpose |
|------|---------|
| `tools/analyze.sh` | Static analysis (clang-tidy, cppcheck) |
| `tools/coverage.sh` | Code coverage reports |
| `tools/simulation/mock_vehicle.py` | Lightweight MAVLink simulator |
| `tools/generators/factgroup/` | Generate FactGroup boilerplate |
| `tools/analyzers/vehicle_null_check.py` | Detect unsafe vehicle access |

## Configuration Files

| File | Purpose |
|------|---------|
| `.clang-format` | C++ code formatting rules |
| `.clang-tidy` | C++ static analysis config |
| `.qmlformat.ini` | QML formatting rules |
| `.qmllint.ini` | QML linting config |
| `.editorconfig` | Editor settings |
| `.pre-commit-config.yaml` | Pre-commit hook configuration |
| `.github/build-config.json` | Centralized version numbers |

## Additional Resources

- **User Manual**: https://docs.qgroundcontrol.com/en/
- **Developer Guide**: https://dev.qgroundcontrol.com/en/
- **MAVLink Protocol**: https://mavlink.io/
- **Qt 6 Documentation**: https://doc.qt.io/qt-6/
- **Discussion Forum**: https://discuss.px4.io/c/qgroundcontrol
- **Discord**: https://discord.gg/dronecode

## Quick Reference

```cpp
// Get active vehicle (ALWAYS null-check!)
Vehicle* vehicle = MultiVehicleManager::instance()->activeVehicle();
if (!vehicle) return;

// Access parameters
ParameterManager* pm = vehicle->parameterManager();
Fact* param = pm->getParameter(-1, "PARAM_NAME");
if (param) {
    QVariant value = param->cookedValue();
    param->setCookedValue(newValue);
}

// Get firmware plugin
FirmwarePlugin* plugin = vehicle->firmwarePlugin();
QStringList modes = plugin->flightModes(vehicle);

// Access managers
SettingsManager::instance()->appSettings()->...
LinkManager::instance()->...
MissionManager* mm = vehicle->missionManager();
```

```qml
// QML quick patterns
import QGroundControl
import QGroundControl.Controls

Item {
    property var _activeVehicle: QGroundControl.multiVehicleManager.activeVehicle

    QGCButton {
        text: qsTr("Arm")
        enabled: _activeVehicle && !_activeVehicle.armed
        onClicked: _activeVehicle.armed = true
    }
}
```
