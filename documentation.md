# HomeCore — Complete Technical Documentation

> **Platform:** ESP32 (Xtensa LX6 Dual-Core) | **Framework:** Arduino / FreeRTOS | **Version:** 1.0.0
> **Document Type:** Engineering Reference | **Audience:** Embedded / IoT Engineers

---

## Table of Contents

- [System Analysis (Critical Section)](#system-analysis)
  - [1. Architectural Design Pattern](#1-architectural-design-pattern)
  - [2. Core Components Responsibilities](#2-core-components-responsibilities)
  - [3. Data Flow](#3-data-flow)
  - [4. State Management Model](#4-state-management-model)
  - [5. Control Logic Analysis](#5-control-logic-analysis)
  - [6. Timing & Event Model](#6-timing--event-model)
  - [7. Persistence Strategy](#7-persistence-strategy)
  - [8. Concurrency Model](#8-concurrency-model)
  - [9. System Limitations](#9-system-limitations)
  - [10. Scalability Analysis](#10-scalability-analysis)
- [Section 01 — Introduction](#introduction)
- [Section 02 — System Architecture](#system-architecture)
- [Section 03 — Folder & File Structure](#folder--file-structure)
- [Section 04 — Hardware Setup](#hardware-setup)
- [Section 05 — Software Setup](#software-setup)
- [Section 06 — Configuration (Config.h)](#configuration-configh-deep-explanation)
- [Section 07 — Core Logic](#core-logic-deep-dive)
- [Section 08 — PIR System](#pir-system)
- [Section 09 — Relay Control](#relay-control)
- [Section 10 — Timer System](#timer-system)
- [Section 11 — Web System](#web-system)
- [Section 12 — Captive Portal](#captive-portal)
- [Section 13 — Storage System](#storage-system)
- [Section 14 — FreeRTOS & Performance](#freertos--performance)
- [Section 15 — Power & Deployment](#power--deployment)
- [Section 16 — Troubleshooting](#troubleshooting)
- [Section 17 — Testing Guide](#testing-guide)
- [Section 18 — Extending the Project](#extending-the-project)
- [Section 19 — Best Practices](#best-practices)

---

Complete production-level reference for architecture, firmware design,
hardware wiring, WebSocket API, FreeRTOS concurrency, storage strategy,
and real-world deployment of the HomeCore ESP32-based home automation controller.

## System Analysis

This section presents a deep engineering analysis of the HomeCore system. It examines
the architectural decisions, data pathways, concurrency model, state management, and design tradeoffs.
Reading this section first enables a thorough understanding of every other part of the documentation.

### 1. Architectural Design Pattern

#### 1.1 Identified Style: Layered Event-Driven Architecture

The system uses a **layered + event-driven hybrid architecture**. At a macro level,
responsibilities are separated into three horizontal layers:

- **Presentation Layer** — WebPortal (HTTP server, WebSocket, Captive Portal, index.html)
- **Business Logic Layer** — ControlEngine (priority resolution, state machine, relay actuation)
- **Infrastructure Layer** — StorageLayer (Preferences, LittleFS), Config.h (static constants), FreeRTOS tasks

Within the business logic layer, execution is **event-driven**: hardware interrupts (PIR sensors),
WebSocket messages, and timer expirations are all treated as events dispatched to the ControlEngine for processing.
This prevents busy-polling and makes the firmware reactive and power-friendly.

#### 1.2 Why This Design?

- **Separation of concerns** — each layer can be modified independently. Adding a new relay type does not touch the web code.
- **Testability** — ControlEngine can be unit-tested in isolation by injecting synthetic events.
- **Reactivity** — event-driven dispatch eliminates polling loops, reducing CPU load and improving responsiveness.
- **Persistence isolation** — by routing all storage through StorageLayer, the rest of the code never deals with flash directly, making storage backend swaps straightforward.

#### 1.3 Component Interaction Map

```
+-------------------+       WebSocket / HTTP       +--------------------+
|    Web Browser    | <---------------------------> |    WebPortal       |
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
                               | Relay1 |    | Relay2 |
                               +--------+    +--------+
```

_Figure 0-1: High-level component interaction diagram_

### 2. Core Components Responsibilities

#### 2.1 ControlEngine

The ControlEngine is the heart of the firmware. Its responsibilities are:

- Maintain the authoritative in-RAM state for all relays (current mode, active timer end-timestamp)
- Resolve control conflicts using the priority ladder: MANUAL > TIMER > PIR
- Evaluate PIR-to-relay mappings and decide whether a PIR event should actuate a relay
- Manage timer lifecycle: creation, expiry detection, cancellation
- Emit state-change events back to WebPortal so connected clients receive real-time updates
- Write critical state transitions to StorageLayer for persistence across reboots

Boundaries: ControlEngine does **not** perform any network operations,
does **not** directly manipulate flash, and does **not** serve HTML.
It consumes events and produces actuator commands and state-update callbacks.

#### 2.2 StorageLayer

StorageLayer abstracts all non-volatile storage into a single interface:

- **Preferences (NVS)** — stores small key-value config: WiFi credentials, relay modes, PIR mapping, timer settings
- **LittleFS** — stores the web interface (index.html) and event log files (JSON line-delimited)
- Provides atomic read/write helpers to ensure partial writes do not corrupt state
- Implements a buffering strategy for event logs to avoid write amplification on flash
- Handles filesystem mounting, error recovery, and format-on-first-boot

Boundaries: StorageLayer never makes decisions about relay behavior or networking. It is a pure persistence service.

#### 2.3 WebPortal

WebPortal owns all network-facing responsibilities:

- AP + STA dual WiFi mode management (connect, reconnect, fallback logic)
- Captive Portal: DNS responder that redirects all DNS queries to the device IP
- HTTP server: serves `index.html` and static assets from LittleFS
- WebSocket server: handles real-time bidirectional communication with the browser
- Parses inbound WebSocket JSON messages and dispatches commands to ControlEngine
- Subscribes to ControlEngine state-change callbacks and broadcasts updates to all WS clients
- Implements offline buffering: if no client is connected, state changes are queued and replayed on reconnect

Boundaries: WebPortal does not make any control decisions — it passes all commands directly to ControlEngine.

#### 2.4 Config.h

Config.h is a purely static compile-time configuration header. It contains:
`#define` constants for GPIO pins, timing values, WiFi defaults, WebSocket port, and feature flags.
It has no runtime state and no dependencies on other modules.
Changing a value in Config.h requires a firmware recompile.

#### 2.5 FreeRTOS Tasks

The firmware distributes work across multiple FreeRTOS tasks to achieve true concurrency on the dual-core ESP32:

- **WebTask (Core 0)** — runs the HTTP and WebSocket loop, WiFi management, Captive Portal DNS
- **ControlTask (Core 1)** — runs the ControlEngine loop: timer evaluation, PIR debounce, relay state enforcement
- **LogTask (Core 0 or 1)** — periodically flushes the in-RAM event buffer to LittleFS
- Inter-task communication uses FreeRTOS queues and mutexes to prevent data races on shared state

### 3. Data Flow

#### 3.1 PIR Trigger Data Flow

```
PIR sensor detects motion
        |
        | (GPIO level change)
        v
GPIO ISR fires (runs on the core that registered the interrupt)
        |
        | xQueueSendFromISR()  [non-blocking]
        v
FreeRTOS PIR Event Queue
        |
        | ControlTask dequeues event
        v
ControlEngine.handlePirEvent(pirIndex)
        |
        |-- Is this relay in MANUAL mode? --> YES --> Ignore event, log suppression
        |                                 \
        |                                  NO
        |-- Is this relay in TIMER mode?  --> YES --> Ignore (timer takes priority over PIR)
        |                                 \
        |                                  NO (relay is in AUTO/PIR mode)
        |
        |-- Look up PIR-to-relay mapping
        |   pirMapping[pirIndex] --> relayMask (bitmask of controlled relays)
        |
        |-- For each relay in relayMask:
        |       setRelayState(relayIndex, ON)
        |       startAutoOffTimer(relayIndex, PIR_AUTO_OFF_DELAY)
        |       updateStateCache(relayIndex, ON, MODE_PIR)
        |       emitStateChange(relayIndex)   --> notifies WebPortal
        |
        v
WebPortal receives state-change callback
        |
        | Serialize to JSON {"type":"state_update","relay":N,"state":1,"mode":"PIR"}
        v
WebSocket broadcast to all connected browsers
        |
        v
Browser UI updates indicator in real-time
```

_Figure 0-2: PIR trigger data flow_

#### 3.2 Web Command Data Flow

```
User clicks button in browser (e.g., "Turn Relay 1 ON manually")
        |
        | WebSocket send: {"type":"set_manual","relay":0,"state":1}
        v
ESP32 WebSocket server receives frame
        |
        | WebPortal.onWebSocketMessage(clientId, data)
        v
Parse JSON, validate fields
        |
        | Dispatch to ControlEngine.applyManualCommand(relayIndex, state)
        v
ControlEngine.applyManualCommand()
        |
        |-- Set relay mode to MANUAL
        |-- Cancel any active PIR or TIMER override
        |-- Drive GPIO to requested state
        |-- Update in-RAM state cache
        |-- Persist mode + state to Preferences (NVS)
        |-- Call emitStateChange(relayIndex)
        v
WebPortal broadcasts updated state JSON to ALL connected clients
        v
All browser tabs update simultaneously in real-time
```

_Figure 0-3: Web command data flow_

#### 3.3 Timer Lifecycle Data Flow

```
User sends: {"type":"set_timer","relay":0,"duration_s":300}
        |
        v
ControlEngine.applyTimerCommand(relayIndex, durationSeconds)
        |
        |-- Compute: endTimestamp = currentTime + durationSeconds
        |-- Set relay mode = TIMER
        |-- Drive relay GPIO ON
        |-- Store timerEnd[relayIndex] = endTimestamp  (in RAM)
        |-- Persist to Preferences
        |-- emitStateChange with remaining seconds
        v
... (300 seconds pass) ...
        |
ControlTask periodic loop runs every TIMER_CHECK_INTERVAL_MS
        |
        |-- For each relay: is (currentTime >= timerEnd[relay]) AND mode == TIMER?
        |       YES --> timerExpired(relayIndex)
        |                   Drive relay GPIO OFF
        |                   Set mode = AUTO
        |                   emitStateChange
        |                   Clear timerEnd in RAM and Preferences
        v
Browser receives: {"type":"state_update","relay":0,"state":0,"mode":"AUTO","remaining_s":0}
```

_Figure 0-4: Timer lifecycle data flow_

### 4. State Management Model

#### 4.1 Sources of Truth

| State Item                  | Primary Store      | Secondary Store    | Notes                                                  |
| --------------------------- | ------------------ | ------------------ | ------------------------------------------------------ |
| Relay ON/OFF state          | RAM (GPIO + cache) | Preferences (NVS)  | GPIO is the real-time truth; NVS recovers after reboot |
| Relay control mode          | RAM                | Preferences (NVS)  | MANUAL / TIMER / AUTO                                  |
| Timer end-timestamp         | RAM                | Preferences (NVS)  | Persisted so timers survive soft reboots               |
| PIR-to-relay mapping        | RAM                | Preferences (NVS)  | Loaded at boot, updated on web command                 |
| WiFi credentials            | Preferences (NVS)  | —                  | Only in NVS, never in RAM long-term                    |
| Web interface (HTML/JS/CSS) | LittleFS           | —                  | Read-only at runtime, updated via OTA or USB           |
| Event log                   | RAM ring-buffer    | LittleFS (flushed) | Buffer prevents excessive flash writes                 |

#### 4.2 Consistency Maintenance

The system maintains consistency through two mechanisms:

- **Write-through on mode changes** — whenever ControlEngine changes a relay mode or state, it immediately writes to Preferences. This means NVS always reflects the current control intent within milliseconds of a change.
- **Boot-time reconciliation** — on startup, StorageLayer reads all persisted state and compares timer end-timestamps against the current RTC/NTP time. Expired timers are discarded; valid timers are reinstated. This prevents "zombie timers" from firing on stale data after a long power-cut reboot.
- **Mutex protection** — the shared state struct in RAM is guarded by a FreeRTOS mutex. ControlTask holds the mutex during state updates; WebTask holds it only briefly during state reads for broadcast serialization.

### 5. Control Logic Analysis

#### 5.1 Mode Switching

Each relay independently operates in one of three modes:

- **AUTO mode** — relay is controlled by the PIR sensor according to the configured mapping. This is the default "smart" mode.
- **MANUAL mode** — relay is locked to a user-specified ON or OFF state. PIR events and timer expirations are completely ignored.
- **TIMER mode** — relay is ON for a user-specified duration, then auto-OFF. PIR events are suppressed during the timer run.

#### 5.2 Priority System: MANUAL > TIMER > PIR

The priority ladder is enforced in ControlEngine every time an actuation request arrives.
The logic is implemented as a guard-clause cascade:

```
function applyActuation(relay, source, requestedState):
    if relay.mode == MANUAL:
        return IGNORED     // Highest priority: MANUAL blocks everything

    if source == PIR:
        if relay.mode == TIMER:
            return IGNORED // TIMER beats PIR
        // PIR is allowed only in AUTO mode
        executeActuation(relay, requestedState, source=PIR)

    if source == TIMER:
        if relay.mode == MANUAL:
            return IGNORED // Already caught above
        executeActuation(relay, requestedState, source=TIMER)

    if source == MANUAL:
        cancelTimer(relay)         // Manual overrides active timer
        executeActuation(relay, requestedState, source=MANUAL)
```

#### 5.3 Conflict Resolution Examples

| Scenario                                     | Active Mode   | Incoming Event | Result                                                            |
| -------------------------------------------- | ------------- | -------------- | ----------------------------------------------------------------- |
| User turns relay ON manually, then PIR fires | MANUAL        | PIR trigger    | PIR ignored. Relay stays ON as set manually.                      |
| Timer is running (5 min), PIR fires          | TIMER         | PIR trigger    | PIR ignored. Timer continues undisturbed.                         |
| PIR auto-on is active, user sets manual OFF  | AUTO (PIR-ON) | MANUAL OFF     | Relay turns OFF immediately. Mode becomes MANUAL. PIR suppressed. |
| Timer running, user sets manual ON           | TIMER         | MANUAL ON      | Timer cancelled. Relay stays ON. Mode becomes MANUAL.             |
| Relay in AUTO, timer command arrives         | AUTO          | TIMER set      | Timer starts. Mode becomes TIMER. PIR suppressed for duration.    |

### 6. Timing & Event Model

#### 6.1 End-Timestamp vs. Countdown

Timers are stored as **absolute end-timestamps** (Unix epoch seconds) rather than countdown values.
This is a deliberate and important design choice. The ControlTask evaluates each timer by comparing
`currentEpochTime >= timerEnd[i]`. There are several advantages:

- **Reboot resilience** — if the device reboots mid-timer, the persisted end-timestamp is still valid. The countdown approach would lose all elapsed time since the counter resets to zero on reboot.
- **No drift** — countdown timers implemented via `millis()` drift if the polling interval is variable or if the task is preempted. Absolute timestamps are immune to this.
- **Cheap evaluation** — comparing two integers is O(1) and requires no state mutation on each tick.
- **Remaining-time broadcast** — the WebSocket can always broadcast accurate remaining seconds: `remaining = timerEnd - currentTime`, which stays correct regardless of when the client connects.

#### 6.2 Time Synchronization

When the device has STA WiFi connectivity, it syncs time via NTP (SNTP). Until a valid NTP sync is obtained,
the internal `millis()` offset is used as a fallback timer base, anchored to boot time.
The `time_sync` WebSocket message allows the browser to push the current Unix timestamp to the device,
providing a fallback time source when NTP is unavailable (e.g., purely AP mode with no internet).
Timer end-timestamps are recalibrated on each successful time_sync to maintain accuracy.

### 7. Persistence Strategy

#### 7.1 What Goes Where

| Data                       | Storage               | Write Trigger             | Rationale                                            |
| -------------------------- | --------------------- | ------------------------- | ---------------------------------------------------- |
| WiFi SSID / Password       | Preferences (NVS)     | On WiFi config save       | Small, infrequently written, needs fast read at boot |
| Relay mode & state         | Preferences (NVS)     | Every mode/state change   | Survives reboot without user re-intervention         |
| Timer end-timestamps       | Preferences (NVS)     | On timer set / cancel     | Allows timer resume after unexpected reboot          |
| PIR-to-relay mapping       | Preferences (NVS)     | On mapping change from UI | User config, rarely changes                          |
| Web interface (index.html) | LittleFS              | Initial flash / OTA       | Large binary, needs filesystem addressing            |
| Event logs                 | LittleFS (JSON lines) | Buffer flush (periodic)   | Batching reduces flash wear                          |

#### 7.2 Flash Wear Tradeoffs

The ESP32's NVS (Non-Volatile Storage) uses a **wear-leveling page-rotation** scheme. Each NVS
partition sector is ~4 KB. NVS rotates writes across pages, so a single key can be updated
approximately **10,000–100,000 times before wear affects that sector**.
For relay state and timer, this means even at 100 state changes per day, the NVS partition
would last over 27 years — well beyond hardware MTBF.

Event logs use a **RAM ring-buffer with batch flush** to LittleFS. LittleFS itself
implements wear leveling across the filesystem partition. Log entries are buffered in RAM until
either the buffer is full or a flush interval elapses (configurable, default 60 seconds).
This keeps flash write frequency low while maintaining reasonable log durability.

### 8. Concurrency Model

#### 8.1 Core Assignment

| Task           | Core   | Priority | Stack Size | Rationale                                                             |
| -------------- | ------ | -------- | ---------- | --------------------------------------------------------------------- |
| WebTask        | Core 0 | 5        | 8192 bytes | Network stack runs on Core 0 by default in Arduino ESP32              |
| ControlTask    | Core 1 | 10       | 4096 bytes | Higher priority ensures relay actuation is not delayed by web traffic |
| LogTask        | Core 0 | 2        | 4096 bytes | Low priority; logging must not starve control or web                  |
| Arduino loop() | Core 1 | 1        | 8192 bytes | Minimal work — delegates to FreeRTOS tasks                            |

#### 8.2 Shared Data and Synchronization

Three shared data structures require mutual exclusion:

- **RelayState array** — protected by `stateMutex`. ControlTask writes; WebTask reads for broadcast serialization.
- **Event log ring-buffer** — protected by `logMutex`. Both tasks can enqueue events; LogTask drains and writes to LittleFS.
- **WebSocket client list** — protected internally by the AsyncWebSocket library's own locking mechanism.

#### 8.3 Race Condition Mitigation

- PIR ISRs use `xQueueSendFromISR()` — they never touch shared d
