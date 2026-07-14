# Repository Map for Agent Navigation

This map is a system-oriented index of responsibilities, interfaces, state, and data flow.
It is designed for autonomous agents that decide investigation paths using their own reasoning.

## 1) Repository Topology

- `General_Driver/`
  - Main ESP32 firmware (Arduino sketch + header-implemented modules).
  - Contains runtime orchestration, control-plane command dispatch, hardware subsystems, networking, storage, and telemetry.
- `SCServo/`
  - Servo communication library used by the arm/gimbal stack.
  - Layered protocol/transport/model API plus isolated examples.
- `README_footage/`
  - Documentation media only (non-runtime assets).
- `.github/`
  - Repository metadata and support docs (this map lives here).

## 2) Runtime Entrypoints and Execution Model

### Primary firmware entrypoint
- `General_Driver/General_Driver.ino`
  - `setup()` initializes hardware interfaces and feature modules.
  - `loop()` executes recurring control-plane and data-plane work.

### Loop-level responsibilities (conceptual)
- Command ingress handling (UART, HTTP bridge, ESP-NOW bridge).
- Motion and module control updates.
- IMU and actuator feedback refresh.
- Safety/heartbeat enforcement.
- Periodic telemetry/output publication.

## 3) Architectural Layers

### Layer A: Transport ingress/egress
- `General_Driver/uart_ctrl.h`
- `General_Driver/http_server.h`
- `General_Driver/esp_now_ctrl.h`

Responsibilities:
- Receive serialized command payloads.
- Parse JSON and route into shared command handling.
- Emit transport-specific responses/acknowledgements.

### Layer B: Command schema and dispatch
- `General_Driver/json_cmd.h`
- `General_Driver/uart_ctrl.h` (dispatch implementation is centered here)

Responsibilities:
- Command identifiers and payload-key conventions.
- Mapping command IDs to subsystem operations.

### Layer C: Domain subsystems
- Mobility: `General_Driver/movtion_module.h`
- Arm: `General_Driver/RoArm-M2_module.h`
- Gimbal: `General_Driver/gimbal_module.h`
- Mission/workflow: `General_Driver/ugv_advance.h`
- IMU wrapper: `General_Driver/IMU_ctrl.h`
- Wi-Fi/network mode: `General_Driver/wifi_ctrl.h`
- Filesystem abstraction: `General_Driver/files_ctrl.h`
- UI/status peripherals: `General_Driver/oled_ctrl.h`, `General_Driver/battery_ctrl.h`, `General_Driver/ugv_led_ctrl.h`

### Layer D: Hardware drivers and external library
- IMU backend:
  - `General_Driver/IMU.cpp`, `General_Driver/IMU.h`
  - `General_Driver/QMI8658.cpp`, `General_Driver/QMI8658.h`, `General_Driver/QMI8658reg.h`
  - `General_Driver/AK09918.cpp`, `General_Driver/AK09918.h`
- Servo library:
  - `SCServo/SCS.h`, `SCServo/SCS.cpp` (protocol primitives)
  - `SCServo/SCSerial.h`, `SCServo/SCSerial.cpp` (bus serial transport)
  - `SCServo/SCSCL.h`, `SCServo/SCSCL.cpp` (SCSCL model API)
  - `SCServo/SMS_STS.h`, `SCServo/SMS_STS.cpp` (SMS/STS model API)

### Layer E: Shared configuration and global state
- `General_Driver/ugv_config.h`

Responsibilities:
- Pin maps, geometry constants, feature flags, control defaults.
- Cross-module globals used as shared runtime state.

## 4) Key Control and Data Flows

### Command flow
1. Input arrives through UART, HTTP endpoint, or ESP-NOW callback.
2. Payload is deserialized into shared JSON command document(s).
3. Dispatcher resolves command type/id.
4. Subsystem handlers mutate control state and trigger hardware actions.
5. Feedback is serialized for one or more output channels.

Primary files involved:
- `General_Driver/uart_ctrl.h`
- `General_Driver/http_server.h`
- `General_Driver/esp_now_ctrl.h`
- `General_Driver/json_cmd.h`
- Target subsystem module(s)

### Persistent configuration flow
1. LittleFS mount and file access setup.
2. Config load/create for device and network settings.
3. Runtime state hydrated from persisted JSON.
4. Updates flushed back to storage by subsystem logic.

Primary files involved:
- `General_Driver/files_ctrl.h`
- `General_Driver/wifi_ctrl.h`
- `General_Driver/ugv_advance.h`
- `General_Driver/data/devConfig.json`
- `General_Driver/data/wifiConfig.json`

### Motion control flow
1. Velocity/motion command updates target setpoints.
2. Encoder sampling computes observed wheel speeds.
3. Optional PID compute derives motor outputs.
4. PWM/direction control is applied to motor drivers.
5. Heartbeat/safety logic can override outputs.

Primary files involved:
- `General_Driver/movtion_module.h`
- `General_Driver/ugv_config.h`

### Arm/servo control flow
1. High-level arm/gimbal command arrives.
2. Kinematic/joint transformation produces actuator targets.
3. Servo API writes commands over bus serial.
4. Feedback polling updates telemetry and control logic.

Primary files involved:
- `General_Driver/RoArm-M2_module.h`
- `General_Driver/gimbal_module.h`
- `SCServo/*` (library layers)
- `General_Driver/ugv_config.h`

### IMU sensing flow
1. IMU wrapper requests sensor update.
2. Backend drivers read accelerometer/gyro/magnetometer.
3. Fusion/orientation calculations update state.
4. Orientation values feed control/telemetry consumers.

Primary files involved:
- `General_Driver/IMU_ctrl.h`
- `General_Driver/IMU.cpp`, `General_Driver/IMU.h`
- `General_Driver/QMI8658*`
- `General_Driver/AK09918*`

## 5) Module Ownership Index

### Core orchestration
- `General_Driver/General_Driver.ino`
  - Owns startup ordering and periodic task ordering.

### Command/control plane
- `General_Driver/uart_ctrl.h`
  - Owns serial intake and central command dispatch logic.
- `General_Driver/json_cmd.h`
  - Owns command IDs and naming conventions.
- `General_Driver/http_server.h`
  - Owns web endpoint command bridge.
- `General_Driver/esp_now_ctrl.h`
  - Owns ESP-NOW transport mode, peer interactions, and callback bridge.

### Mobility
- `General_Driver/movtion_module.h`
  - Owns motor IO control, encoder readback, PID, and heartbeat interaction.

### Manipulator and mission
- `General_Driver/RoArm-M2_module.h`
  - Owns arm servo initialization, movement primitives, and feedback capture.
- `General_Driver/ugv_advance.h`
  - Owns mission format/playback and selected advanced behavior orchestration.
- `General_Driver/gimbal_module.h`
  - Owns gimbal axis control and steady-mode behavior.

### Sensing and telemetry peripherals
- `General_Driver/IMU_ctrl.h`
  - Owns firmware-facing IMU state update/packaging.
- `General_Driver/battery_ctrl.h`
  - Owns battery/INA219 measurement integration.
- `General_Driver/oled_ctrl.h`
  - Owns on-device display rendering/state projection.
- `General_Driver/ugv_led_ctrl.h`
  - Owns auxiliary LED/PWM outputs.

### Networking and storage
- `General_Driver/wifi_ctrl.h`
  - Owns AP/STA mode transitions and Wi-Fi config persistence behavior.
- `General_Driver/files_ctrl.h`
  - Owns LittleFS mount lifecycle and file utility functions.

### Shared constants and globals
- `General_Driver/ugv_config.h`
  - Owns compile-time constants and mutable globals consumed by many modules.

## 6) Shared Runtime State Surfaces

This codebase is header-heavy and relies on shared globals.
Cross-module behavior is often mediated by values declared in `General_Driver/ugv_config.h` and referenced elsewhere.

Common state categories:
- Device/mode state (example categories: module selection, transport mode, steady/feedback modes).
- Motion state (setpoints, PID toggles, scaling factors, heartbeat timing/flags).
- Arm geometry and constraints (link lengths, IDs, angle/range limits, offsets).
- Telemetry JSON documents and output buffers.

Agent note:
- When investigating behavior, treat global-variable writers/readers as part of the effective call graph, even if direct function calls are absent.

## 7) Filesystem and Data Artifacts

### Source-controlled data defaults
- `General_Driver/data/devConfig.json`
- `General_Driver/data/wifiConfig.json`

### Runtime-managed content (through LittleFS APIs)
- Wi-Fi config payloads.
- Mission files (for mission playback/edit operations).
- Device configuration snapshots.

### Generated/build artifacts
- `General_Driver/build/`
  - Build output for ESP32 toolchain; not authoritative source for logic.

## 8) External Library Boundary: SCServo

Use `SCServo/` as an independent boundary with its own API stack.

Library structure:
- Protocol base: `SCS*`
- Serial transport: `SCSerial*`
- Device families: `SCSCL*`, `SMS_STS*`
- Usage references: `SCServo/examples/`

Integration boundary with firmware:
- Firmware modules call library model APIs.
- Library handles packet formatting, transport, and register-level command semantics.

## 9) Quick Navigation Table

- Startup + loop orchestration: `General_Driver/General_Driver.ino`
- Command schema: `General_Driver/json_cmd.h`
- Command dispatch: `General_Driver/uart_ctrl.h`
- HTTP bridge: `General_Driver/http_server.h`
- ESP-NOW bridge: `General_Driver/esp_now_ctrl.h`
- Motion stack: `General_Driver/movtion_module.h`
- Arm stack: `General_Driver/RoArm-M2_module.h`
- Gimbal stack: `General_Driver/gimbal_module.h`
- Mission stack: `General_Driver/ugv_advance.h`
- Wi-Fi + persistence: `General_Driver/wifi_ctrl.h`
- LittleFS/file utilities: `General_Driver/files_ctrl.h`
- IMU wrapper/backend: `General_Driver/IMU_ctrl.h`, `General_Driver/IMU.cpp`, `General_Driver/IMU.h`
- Shared globals/constants: `General_Driver/ugv_config.h`
- Servo library entry files: `SCServo/SCS.h`, `SCServo/SCSerial.h`, `SCServo/SCSCL.h`, `SCServo/SMS_STS.h`

## 10) Agent Usage Notes

- Prefer tracing by data flow (input -> dispatch -> subsystem -> shared state -> output) rather than by file proximity.
- Account for both direct call edges and indirect shared-state coupling.
- Distinguish firmware-layer issues (`General_Driver/`) from servo-library-layer issues (`SCServo/`) before deep dives.
