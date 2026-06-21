# ESP32 Smart Home Automation (2 Relays + 3 PIR)

ESP32 smart home controller (2 relays, 3 PIR inputs) that safely coordinates relay actions using a priority-based engine and an optional “Night Lock” safety mode. Motion (PIR), manual toggles, and timed actions are reconciled per-relay so outputs remain conflict-safe and predictable.

Key capabilities:

- Wi‑Fi captive portal (AP) + optional STA network
- WebSocket live updates + offline notification buffering (pending queue)
- Conflict-safe relay control:
  - PIR motion extends an AUTO “hold” window (only when relay is in AUTO and no timer is active)
  - Timers drive relay outputs and restore AUTO at end
  - Night Lock forces all relays OFF and cancels active timers
- Optional access control (MAC-based auth + restricted/unauthorized pages)
- Persistent configuration (Preferences) + event logs + pending notifications (LittleFS)
- FreeRTOS task split + watchdog safety

## 1) Folder Layout

```
smart-home-automation-esp32/
  README.md
  SmartHomeAutomation/
    SmartHomeAutomation.ino
    Config.h
    SystemTypes.h
    Utils.h
    TimeKeeper.h/.cpp
    StorageLayer.h/.cpp
    ControlEngine.h/.cpp
    WebPortal.h/.cpp
    data/
      index.html
```

## 2) Dependencies

Install these Arduino libraries:

- `ArduinoJson`
- `WebSockets` by Markus Sattler
- `LittleFS` (ships with ESP32 core)
- `Preferences` (ships with ESP32 core)

Board package: `esp32` (Arduino core for ESP32).

## 3) Pin Mapping (default)

Set in `SmartHomeAutomation/Config.h`:

- Relays:
  - Relay A -> GPIO26
  - Relay B -> GPIO27
- PIR:
  - PIR A -> GPIO32 (Relay A)
  - PIR B -> GPIO33 (Relay B)
  - PIR C -> GPIO25 (Relay A + Relay B)

Adjust pin numbers and sensor mapping as needed.

## 4) Build + Upload Steps

1. Open `SmartHomeAutomation/SmartHomeAutomation.ino` in Arduino IDE.
2. Choose your ESP32 board and COM port.
3. Upload filesystem first (LittleFS upload tool) so `/index.html` and the HTML pages are available.
4. Upload firmware.
5. Connect to the device AP:
   - **AP SSID**: `tarshid`
   - **AP password**: `12345678`
   - (Access portal URL is the device softAP IP; captive portal redirects you.)
6. Open the device portal URL from the softAP IP returned/used by the captive redirect.

## 5) Core Logic

### Operating Modes (what actually drives outputs)

Each relay’s applied output is decided from these sources (highest wins):

`MANUAL > TIMER > PIR > NONE`

Key behaviors:

- **PIR motion**:
  - PIR events extend an “auto hold” window **only when the relay is in `AUTO` mode** and a timer is not active on that relay.
  - If a relay is `MANUAL ON/OFF`, PIR does not override it.
- **Timer**:
  - When a timer is active, the relay output follows the timer’s `targetState`.
  - When the timer ends, the relay returns to **`AUTO`** (restores previous relay state via `previousState`).
- **Night Lock** (safety freeze):
  - When enabled and time phase is **NIGHT** (06:00–18:00 is DAY), the controller forces **all relays OFF** and cancels active timers.
  - Certain operations (starting timers, PIR mapping changes, resets, rated power/energy tracking updates) are blocked while Night Lock is active.

### Timer Lifecycle / Time alignment

- Timers store: `active`, `startEpoch`, `endEpoch`, `durationMinutes`, `targetState`, plus restore bookkeeping.
- Remaining time is always derived from absolute epoch timestamps once time is synced.

### Persistence

- **Preferences** (`PREF_NAMESPACE = "smart_home"`):
  - relay manual mode
  - timer plans
  - relay state/source
  - energy tracking enabled flag
  - PIR mapping
  - rated power (and whether it is locked)
  - periodic housekeeping markers (daily cleanup)
  - user activity log controls (login inactivity handling)
- **LittleFS**:
  - event log file (`/logs.jsonl`)
  - pending notifications buffer (`/pending.jsonl`)
- Daily cleanup is controlled by `LOG_RETENTION_DAYS` (and additional size caps).

## 6) Time Synchronization

Time sources:

1. NTP (when STA is connected)
2. Client `time_sync` WebSocket packet on each connect

When time updates, timer remaining time naturally re-aligns because end timestamps are absolute.

## 7) WebSocket Contract

Only the WebSocket message types listed below are accepted:

- `time_sync`
- `set_manual`
- `set_timer`
- `cancel_timer`
- `set_energy_tracking`
- `get_state`

### Client -> ESP32 (request payloads)

#### 1) `time_sync`
Send one of:
- epoch form:
  - `{"type":"time_sync","epoch":1710000000}`
- or date/time fields:
  - `{"type":"time_sync","year":2026,"month":6,"day":21,"hours":12,"minutes":34,"seconds":56,"tzOffsetMinutes":0}`

#### 2) `set_manual`
- `{"type":"set_manual","channel":0,"mode":"ON|OFF|AUTO"}`

#### 3) `set_timer`
Time fields are required by the controller for validation:
- `{"type":"set_timer","channel":1,"durationMinutes":5,"target":"ON|OFF","epoch":1710000000,"year":2026,"month":6,"day":21,"hours":12,"minutes":34,"seconds":56,"tzOffsetMinutes":0}`

Notes:
- `durationMinutes` is the primary field.
- If you provide `durationSec`, the controller converts it to minutes.

#### 4) `cancel_timer`
- `{"type":"cancel_timer","channel":1}`

#### 5) `set_energy_tracking`
- `{"type":"set_energy_tracking","enabled":true}`

#### 6) `get_state`
- `{"type":"get_state"}`

### ESP32 -> Client (response/event payloads)

Controller sends JSON messages of types like:

- `state_snapshot` (full state)
- `command_ack` (ack for commands)
- event notifications such as:
  - `relay.changed`
  - `timer.started`
  - `timer.ended`
  - `timer.canceled`
  - `pir.motion`
  - `pir.idle`
  - `night_lock.activated` / `night_lock.released`
  - `energy_update` (energy tracking updates)
  - plus various connectivity/storage events (client connected/disconnected)

(Each event includes `ts` epoch and may include `channel`/`relay` and `mac` depending on the event.)

## 8) FreeRTOS + Watchdog

- **Core 1 task**: PIR processing + control evaluation + relay actuation
- **Core 0 task**: Web server + WebSocket + queue handling + housekeeping

Task watchdog is enabled to auto-reset on hangs.

## 9) Notes for Real Deployment

- Use opto-isolated relay module and proper power rails.
- Validate relay active HIGH/LOW logic for your hardware:
  - default config assumes most relay boards are **active-low** (`RELAY_ACTIVE_LOW = true`)
- Access control is enabled by default (`ENABLE_ACCESS_CONTROL = true`):
  - UI routes can show restricted/unauthorized pages based on registered MACs.
- Replace default AP credentials in `SmartHomeAutomation/src/Config.h` before deployment:
  - AP SSID: `tarshid`
  - AP password: `12345678`
- Captive portal redirects to the device softAP IP.
