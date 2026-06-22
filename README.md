# Tarshid — ESP32 Smart Home Automation Controller

> **Platform:** ESP32 (Xtensa LX6 Dual-Core)  
> **Framework:** Arduino / PlatformIO / FreeRTOS  
> **Version:** 1.0.0  
> **Project Name:** HomeCore (Tarshid)

An ESP32-based smart home controller that safely coordinates **2 relay outputs** and **3 PIR motion sensor inputs** using a priority-based control engine. Motion detection, manual toggles, and timed actions are reconciled per-relay so outputs remain **conflict-safe** and **predictable** at all times.

---

## Table of Contents

- [Overview](#overview)
- [Key Features](#key-features)
- [Architecture](#architecture)
- [Project Structure](#project-structure)
- [Hardware Setup](#hardware-setup)
- [Software Dependencies](#software-dependencies)
- [Configuration](#configuration)
- [Build & Upload](#build--upload)
- [Core Logic](#core-logic)
- [WebSocket API](#websocket-api)
- [FreeRTOS Task Model](#freertos-task-model)
- [Persistence & Storage](#persistence--storage)
- [Deployment Notes](#deployment-notes)
- [Troubleshooting](#troubleshooting)

---

## Overview

Tarshid (HomeCore) is a production-grade embedded firmware for ESP32 that implements a complete home automation controller. The system uses a **layered event-driven architecture** to manage relay outputs based on three independent input sources — PIR motion sensors, manual web commands, and time-based timers — while guaranteeing conflict-free operation through a strict priority ladder: **MANUAL > TIMER > PIR**.

The controller exposes a full-featured **web portal** served directly from the ESP32 via a captive portal (Access Point mode), with real-time bidirectional communication through **WebSockets**. No external server or cloud service is required.

### Design Philosophy

- **Separation of Concerns** — Presentation (WebPortal), Business Logic (ControlEngine), and Infrastructure (StorageLayer) are isolated into independent layers, each modifiable without touching the others.
- **Event-Driven Reactivity** — Hardware interrupts (PIR), WebSocket messages, and timer expirations are all treated as events dispatched to the ControlEngine, eliminating busy-polling and reducing CPU load.
- **Persistence Isolation** — All flash storage is abstracted through StorageLayer, so the control and web layers never deal with flash directly.

---

## Key Features

| Feature | Description |
|---|---|
| **Wi-Fi Captive Portal** | AP mode with SSID `tarshid` + optional STA network connection; DNS-based captive portal redirects all browser requests to the device UI |
| **WebSocket Live Updates** | Real-time bidirectional JSON communication for relay state, timer progress, PIR events, energy tracking, and system events |
| **Conflict-Safe Relay Control** | Priority ladder (MANUAL > TIMER > PIR) ensures outputs remain predictable; level-triggered PIR hold windows extend while motion is detected |
| **Timer System** | Absolute epoch-based timers that survive reboots; automatic mode restoration (AUTO) when timer expires |
| **Night Lock (Opt-in)** | Safety mode that forces all relays OFF during night hours (18:00–06:00); user toggle persisted across reboots via NVS |
| **Energy Tracking** | Per-relay power consumption estimation using configurable rated power; accumulated on-time and energy statistics persisted to NVS |
| **Access Control** | MAC-based authentication with restricted/unauthorized HTML pages; configurable max auth attempts and lockout duration |
| **FreeRTOS Concurrency** | Dual-core task split: Core 1 for control/PIR, Core 0 for web/network; mutex-protected shared state; watchdog auto-reset on hangs |
| **Persistent Configuration** | All settings (relay modes, timers, PIR mapping, WiFi credentials, Night Lock option) stored in NVS (Preferences) and survive reboots |
| **LittleFS Web UI** | Web interface assets (`index.html`, `restricted.html`, `unauthorized.html`) served from LittleFS filesystem partition |

---

## Architecture

```
+-------------------+       WebSocket / HTTP       +--------------------+
|    Web Browser    | <--------------------------> |    WebPortal       |
+-------------------+                              | (HTTP + WS Server) |
                                                   +--------------------+
                                                           |
                                                    Command / Query
                                                           |
                                                           v
+----------------+   PIR Events    +--------------------+   Persist/Load   +-------------------+
|  PIR Sensors   | --------------> |   ControlEngine    | <--------------> |   StorageLayer    |
| (GPIO ISR)     |                 | (State machine,    |                  | (Preferences +    |
+----------------+                 |  Priority logic,   |                  |  LittleFS)        |
                                   |  Timer management) |                  +-------------------+
                                   +--------------------+
                                           |
                                    Relay actuation
                                           |
                                    +------+------+
                                    |             |
                               +--------+    +--------+
                               | Relay A |    | Relay B |
                               +--------+    +--------+
```

### Component Responsibilities

| Component | Role |
|---|---|
| **ControlEngine** | Heart of the firmware — maintains authoritative relay state, resolves control conflicts via priority ladder, evaluates PIR-to-relay mappings, manages timer lifecycle, emits state-change events |
| **StorageLayer** | Abstracts all non-volatile storage (Preferences/NVS for key-value config, LittleFS for web UI files and event logs) into a single persistence interface |
| **WebPortal** | Owns all network-facing responsibilities — WiFi management, captive portal DNS, HTTP server, WebSocket server, JSON message parsing/dispatch, state-change broadcasting |
| **Config.h** | Static compile-time configuration (GPIO pins, timing values, WiFi defaults, feature flags); no runtime state |
| **TimeKeeper** | Manages time synchronization from NTP and WebSocket `time_sync` messages; provides epoch timestamps, day-phase classification, and timer alignment |
| **SystemTypes.h** | Defines all shared data structures, enums (`RelayState`, `RelayMode`, `DayPhase`), and configuration structs (`RelayConfig`, `PirConfig`, `TimerPlan`) |
| **Utils.h** | Utility functions and helper macros used across components |

---

## Project Structure

```
tarshid/
├── README.md                    # This file
├── platformio.ini               # PlatformIO build configuration
├── documentation.md             # Complete technical documentation (21.4 KB)
├── SmartHomeAutomation/
│   ├── src/
│   │   ├── main.cpp             # Arduino / PlatformIO entry point
│   │   ├── Config.h             # Hardware pin mapping, constants, compile-time config (79 lines)
│   │   ├── SystemTypes.h        # Shared types: enums, structs, relay/PIR/timer configs
│   │   ├── Utils.h              # Helper functions and macros
│   │   ├── TimeKeeper.h          # Time synchronization interface
│   │   ├── TimeKeeper.cpp        # NTP + WebSocket time sync implementation
│   │   ├── StorageLayer.h        # Persistence abstraction interface
│   │   ├── StorageLayer.cpp      # NVS (Preferences) + LittleFS implementation
│   │   ├── ControlEngine.h       # Relay control engine interface
│   │   ├── ControlEngine.cpp     # Priority resolution, state machine, timer management (1210 lines)
│   │   ├── WebPortal.h           # Network/web interface
│   │   └── WebPortal.cpp         # HTTP server, WebSocket, captive portal implementation
│   └── data/                     # LittleFS filesystem image (uploaded to flash)
│       ├── index.html            # Main web UI (served at device IP)
│       ├── restricted.html       # Shown for authorized-but-restricted MAC addresses
│       └── unauthorized.html      # Shown for unregistered MAC addresses
└── .pio/                         # PlatformIO build artifacts and library dependencies
    └── libdeps/
        └── esp32dev/
            ├── ArduinoJson/       # JSON parsing/serialization library (v7.1.0+)
            └── WebSockets/        # WebSocket server/client library (v2.6.1+)
```

**Total:** 251 files, ~39,384 lines of code (including third-party libraries)

---

## Hardware Setup

### Required Components

| Component | Quantity | Description |
|---|---|---|
| ESP32 DevKit | 1 | ESP32-WROOM-32 development board |
| Relay Module | 2 | Opto-isolated relay boards (active-low recommended) |
| PIR Sensor | 3 | HC-SR501 or equivalent passive infrared motion sensors |
| Power Supply | 1 | 5V regulated (for ESP32 + relay coils) |

### Default Pin Mapping

Configured in `SmartHomeAutomation/src/Config.h`:

| Component | GPIO Pin | Notes |
|---|---|---|
| Relay A | GPIO 26 | Active-low (LOW = energized) |
| Relay B | GPIO 27 | Active-low (LOW = energized) |
| PIR A | GPIO 32 | Triggers Relay A |
| PIR B | GPIO 33 | Triggers Relay B |
| PIR C | GPIO 25 | Shared sensor — triggers both Relay A and Relay B |

### Wiring Notes

- Use **opto-isolated relay modules** and proper power rails for safety.
- The default configuration assumes **active-low** relay boards (`RELAY_ACTIVE_LOW = true`). Set to `false` if your relay module is active-high.
- PIR sensors should be powered from the ESP32 3.3V rail. Adjust debounce timing in `Config.h` if needed (`PIR_DEBOUNCE_MS = 180`).
- Adjust GPIO pin numbers in `Config.h` to match your specific wiring.

---

## Software Dependencies

Managed by PlatformIO — declared in `platformio.ini`:

| Library | Version | Purpose |
|---|---|---|
| [ArduinoJson](https://github.com/bblanchon/ArduinoJson) | ^7.1.0 | JSON parsing/serialization for WebSocket payloads, state snapshots, and NVS log data |
| [WebSockets](https://github.com/Links2004/arduinoWebSockets) | ^2.6.1 | WebSocket server (`WebSocketsServer.h`) for real-time browser communication |
| LittleFS | Built-in (ESP32 core) | Filesystem for serving web UI HTML files from flash |
| Preferences (NVS) | Built-in (ESP32 core) | Key-value persistent storage for relay modes, timers, WiFi credentials, and settings |

**Board Package:** `espressif32` (Arduino core for ESP32)

---

## Configuration

All static configuration is centralized in `SmartHomeAutomation/src/Config.h`. Key settings:

### Network

```cpp
constexpr char AP_SSID[] = "tarshid";         // Access Point SSID
constexpr char AP_PASSWORD[] = "12345678";     // AP password (min 8 chars)
constexpr char STA_SSID[] = "";                // Infrastructure WiFi SSID (blank = AP-only)
constexpr char STA_PASSWORD[] = "";            // Infrastructure WiFi password
```

### Relay Configuration

```cpp
constexpr RelayConfig RELAY_CONFIG[] = {
    {26, "Relay A", 60.0f},    // GPIO 26, 60W rated power
    {27, "Relay B", 100.0f},   // GPIO 27, 100W rated power
};
constexpr bool RELAY_ACTIVE_LOW = true;         // Set false for active-high relay boards
```

### PIR Configuration

```cpp
constexpr PirConfig PIR_CONFIG[] = {
    {32, relayMaskForRelay(0), "PIR A"},                               // Drives Relay A only
    {33, relayMaskForRelay(1), "PIR B"},                               // Drives Relay B only
    {25, relayMaskForRelay(0) | relayMaskForRelay(1), "PIR C"},        // Drives both relays
};
constexpr uint32_t PIR_DEBOUNCE_MS = 180;        // Debounce interval for PIR inputs
```

### Timing

```cpp
constexpr uint32_t CONTROL_TASK_PERIOD_MS = 50;   // Control loop tick interval
constexpr uint32_t WEB_TASK_PERIOD_MS = 10;       // Web loop tick interval
constexpr uint32_t HOUSEKEEPING_PERIOD_MS = 1500;  // Housekeeping/maintenance interval
constexpr uint8_t DAY_START_HOUR = 6;             // Day phase starts at 06:00
constexpr uint8_t NIGHT_START_HOUR = 18;          // Night phase starts at 18:00
```

### Access Control

```cpp
constexpr bool ENABLE_ACCESS_CONTROL = true;      // MAC-based auth enabled
constexpr uint8_t MAX_AUTH_ATTEMPTS = 5;          // Max failed auth attempts before lockout
constexpr uint32_t AUTH_LOCKOUT_SECONDS = 300;   // Lockout duration (5 minutes)
```

### Ports

```cpp
constexpr uint16_t HTTP_PORT = 80;                // HTTP server port
constexpr uint16_t WS_PORT = 81;                   // WebSocket server port
constexpr uint8_t WS_MAX_CLIENTS = 8;             // Max simultaneous WebSocket connections
```

---

## Build & Upload

### Prerequisites

1. [PlatformIO](https://platformio.org/) (CLI or VS Code extension)
2. ESP32 board package (`espressif32`)

### Steps

```bash
# 1. Clone or copy the project to your workspace

# 2. Upload LittleFS filesystem image first (web UI assets)
pio run -t uploadfs

# 3. Build and upload firmware
pio run -t upload

# 4. Open serial monitor (optional, for debugging)
pio device monitor -b 115200
```

### Connect to the Device

1. Connect to the ESP32 Access Point:
   - **SSID:** `tarshid`
   - **Password:** `12345678`
2. Open a browser — the captive portal will redirect you to the device's web UI.
3. The web UI provides real-time control of relays, timers, PIR settings, energy tracking, and system configuration.

---

## Core Logic

### Operating Modes

Each relay independently operates in one of three modes. The applied output is decided from these sources (highest priority wins):

```
MANUAL > TIMER > PIR > NONE
```

| Mode | Behavior |
|---|---|
| **AUTO** (default) | Relay is controlled by PIR motion sensors according to the configured mapping. PIR events extend a "hold" window while the debounced PIR level remains active. |
| **MANUAL** | Relay is locked to user-specified ON or OFF. PIR events and timer expirations are completely ignored. Highest priority. |
| **TIMER** | Relay follows the timer's target state (ON/OFF) for a specified duration. When the timer ends, the relay returns to AUTO mode with its previous state. |

### PIR Motion Behavior

- PIR events **only** actuate relays in **AUTO** mode when no timer is active.
- The hold window is **level-triggered**: as long as the debounced PIR input remains active, the hold time is extended.
- Motion events are published only on the **inactive-to-active edge** to prevent event spam.
- PIR C (GPIO 25) is a shared sensor that drives **both** Relay A and Relay B simultaneously.

### Night Lock (Opt-in Safety Mode)

- Controlled by a user-facing toggle in the System Settings panel.
- When **enabled** and the current time phase is **NIGHT** (18:00–06:00), the controller forces **all relays OFF** and cancels active timers.
- Manual ON, timer-ON, and PIR motion are all **blocked** during the active night lock phase.
- When **disabled** (default), the controller ignores the night phase — all automation works identically day and night.
- The user's choice is **persisted to NVS** (`night_lock_en`) and survives reboots.
- Toggling the option OFF while the lock is active releases it on the next control tick (~50 ms).

### Timer Lifecycle

Timers use **absolute epoch timestamps** (not countdowns) for robustness:

- **Reboot resilience:** Persisted end-timestamps survive reboots. Valid timers are reinstated after restart; expired/stale timers are discarded.
- **No drift:** Absolute timestamps are immune to task preemption and variable polling intervals.
- **Accurate remaining time:** WebSocket broadcasts always show `remaining = timerEnd - currentTime`, correct regardless of when a client connects.

### Conflict Resolution Examples

| Scenario | Active Mode | Incoming Event | Result |
|---|---|---|---|
| User turns relay ON manually, then PIR fires | MANUAL | PIR trigger | PIR ignored. Relay stays ON. |
| Timer running (5 min), PIR fires | TIMER | PIR trigger | PIR ignored. Timer continues. |
| PIR auto-on active, user sets manual OFF | AUTO (PIR-ON) | MANUAL OFF | Relay turns OFF. Mode becomes MANUAL. |
| Timer running, user sets manual ON | TIMER | MANUAL ON | Timer cancelled. Relay stays ON. Mode becomes MANUAL. |
| Relay in AUTO, timer command arrives | AUTO | TIMER set | Timer starts. Mode becomes TIMER. |

---

## WebSocket API

The controller communicates with the web UI through a JSON-based WebSocket protocol on port **81**.

### Client → ESP32 (Request Payloads)

#### `time_sync`
Synchronize the device clock from the browser:
```json
{"type": "time_sync", "epoch": 1710000000}
```
Or with date/time fields:
```json
{"type": "time_sync", "year": 2026, "month": 6, "day": 21, "hours": 12, "minutes": 34, "seconds": 56, "tzOffsetMinutes": 0}
```

#### `set_manual`
Set a relay to manual mode:
```json
{"type": "set_manual", "channel": 0, "mode": "ON"}
```
Mode options: `"ON"`, `"OFF"`, `"AUTO"`

#### `set_timer`
Start a timed relay action:
```json
{"type": "set_timer", "channel": 1, "durationMinutes": 5, "target": "ON", "epoch": 1710000000, "year": 2026, "month": 6, "day": 21, "hours": 12, "minutes": 34, "seconds": 56, "tzOffsetMinutes": 0}
```

#### `cancel_timer`
Cancel an active timer:
```json
{"type": "cancel_timer", "channel": 1}
```

#### `set_energy_tracking`
Enable/disable energy tracking:
```json
{"type": "set_energy_tracking", "enabled": true}
```

#### `set_night_lock_option`
Toggle the Night Lock master switch:
```json
{"type": "set_night_lock_option", "enabled": true}
```

#### `get_state`
Request a full state snapshot:
```json
{"type": "get_state"}
```

### ESP32 → Client (Response/Event Payloads)

| Message Type | Description |
|---|---|
| `state_snapshot` | Full system state (relay states, modes, timers, PIR, energy, Night Lock) |
| `command_ack` | Acknowledgment for commands (`ok: true/false`) |
| `relay.changed` | Relay state changed |
| `timer.started` | Timer began |
| `timer.ended` | Timer expired |
| `timer.canceled` | Timer was cancelled |
| `pir.motion` | PIR sensor detected motion |
| `pir.idle` | PIR sensor returned to idle |
| `night_lock.activated` | Night Lock is now actively forcing relays OFF |
| `night_lock.released` | Night Lock deactivated |
| `night_lock_option.changed` | User toggled the Night Lock master switch |
| `energy_update` | Energy tracking statistics update |

**State Snapshot Fields:**

| Field | Type | Description |
|---|---|---|
| `state.nightLock` | `bool` | `true` while lock is actively forcing relays OFF (option ON + phase NIGHT) |
| `state.nightLockOption` | `bool` | `true` if user has enabled the Night Lock master switch |
| `state.energyTrackingEnabled` | `bool` | `true` if energy tracking is active |

---

## FreeRTOS Task Model

The firmware distributes work across multiple FreeRTOS tasks to achieve true concurrency on the ESP32's dual-core processor:

| Task | Core | Priority | Stack Size | Responsibility |
|---|---|---|---|---|
| **WebTask** | Core 0 | 5 | 8192 bytes | HTTP server, WebSocket loop, WiFi management, captive portal DNS |
| **ControlTask** | Core 1 | 10 | 4096 bytes | PIR processing, control evaluation, timer management, relay actuation |
| **LogTask** | Core 0 | 2 | 4096 bytes | Periodic flush of in-RAM event buffer to LittleFS |
| **Arduino loop()** | Core 1 | 1 | 8192 bytes | Minimal work — delegates to FreeRTOS tasks |

### Synchronization

- **RelayState array** — protected by `stateMutex`. ControlTask writes; WebTask reads for broadcast serialization.
- **Event log ring-buffer** — protected by `logMutex`. Both tasks can enqueue events; LogTask drains and writes to LittleFS.
- **WebSocket client list** — protected internally by the WebSockets library's own locking.
- **PIR ISRs** use `xQueueSendFromISR()` — they never touch shared data directly, avoiding race conditions.

### Watchdog

Task watchdog is enabled with a 12-second timeout (`WATCHDOG_TIMEOUT_SECONDS = 12`) to auto-reset on hangs.

---

## Persistence & Storage

### What Goes Where

| Data | Storage | Write Trigger | Rationale |
|---|---|---|---|
| WiFi SSID / Password | Preferences (NVS) | On WiFi config save | Small, infrequently written, fast boot read |
| Relay mode & state | Preferences (NVS) | Every mode/state change | Survives reboot without re-intervention |
| Timer end-timestamps | Preferences (NVS) | On timer set/cancel | Timer resume after unexpected reboot |
| PIR-to-relay mapping | Preferences (NVS) | On mapping change from UI | User config, rarely changes |
| Night Lock option | Preferences (NVS) | On toggle | Persists user preference across reboots |
| Energy tracking flag | Preferences (NVS) | On toggle | Persists feature state |
| Web UI (HTML files) | LittleFS | Initial flash / OTA | Large files, filesystem addressing |
| Event logs | LittleFS (JSON lines) | Buffer flush (periodic) | Batching reduces flash wear |

### Flash Wear Considerations

- NVS uses wear-leveling page rotation (~4 KB sectors), supporting ~10,000–100,000 writes per key. At 100 state changes/day, this equates to 27+ years of operation.
- Event logs use a **RAM ring-buffer with batch flush** to minimize LittleFS write frequency.
- Activity history persistence is currently disabled to reduce flash writes and save space. The `/api/logs` endpoint returns an empty array for frontend compatibility.

### Boot-Time Reconciliation

On startup, StorageLayer reads all persisted state and compares timer end-timestamps against current RTC/NTP time. Expired timers are discarded; valid timers are reinstated. This prevents stale "zombie timers" from firing after a long power-cut reboot.

---

## Deployment Notes

- **Replace default credentials** in `Config.h` before deploying to production:
  - AP SSID: `tarshid`
  - AP password: `12345678`
- **Validate relay polarity** — most ESP32 relay boards are active-low. Set `RELAY_ACTIVE_LOW` accordingly.
- **Use opto-isolated relay modules** and proper power rails for electrical safety.
- **Access control** is enabled by default (`ENABLE_ACCESS_CONTROL = true`). Configure MAC addresses for authorized devices.
- The captive portal automatically redirects all DNS queries to the device IP, providing a seamless setup experience.

---

## Troubleshooting

| Issue | Solution |
|---|---|
| Device not appearing in WiFi scan | Verify the ESP32 is powered and the firmware is flashed correctly. Check serial monitor output. |
| Web UI not loading | Ensure LittleFS image was uploaded (`pio run -t uploadfs`). Verify `index.html` exists in the `data/` directory. |
| Relays not responding | Check GPIO wiring matches `Config.h`. Verify `RELAY_ACTIVE_LOW` matches your relay board polarity. |
| PIR sensors not triggering | Confirm PIR wiring (3.3V, GND, OUT). Adjust `PIR_DEBOUNCE_MS` if needed. Ensure relay is in AUTO mode. |
| PIR sensors trigger while disconnected | Configure PIR GPIOs with pull-down/pull-up so they don’t float. Firmware sets PIR pins to `INPUT_PULLDOWN` by default in `ControlEngine::begin()` to prevent phantom triggers when sensors are unplugged. |
| Timer not surviving reboot | Verify NTP or `time_sync` WebSocket message has been received before setting a timer. Check serial monitor for "timer.blocked" errors. |
| Night Lock not activating | Confirm the Night Lock option is enabled in System Settings. Verify time is synchronized (NTP or browser `time_sync`). Check `DAY_START_HOUR` and `NIGHT_START_HOUR` in Config.h. |
| WebSocket connection drops | Check WiFi signal strength. Reduce `WS_MAX_CLIENTS` if too many browsers are connected. Monitor serial output for stack overflow warnings. |
| Overheat cooldown mode (temperature feature) | Firmware monitors ESP32 chip temperature and can suspend relays when a configured threshold is exceeded. In the UI, configure the threshold (°C) and cooldown duration in **System Settings**. If overheat happens too frequently, increase the threshold or reduce load/airflow constraints. |



---

## License

This project is provided as-is for educational and personal use. See individual library licenses for third-party dependencies (ArduinoJson, WebSockets).
