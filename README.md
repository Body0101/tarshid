# ESP32 Smart Home Automation (2 Relays + 3 PIR)

ESP32 smart home controller (2 relays, 3 PIR inputs) that safely coordinates relay actions using a priority-based engine. Motion (PIR), manual toggles, and timed actions are reconciled per-relay so outputs remain conflict-safe and predictable. The optional **Night Lock** safety mode (force-off at night) is now opt-in — toggle it from the System Settings panel, and your choice is remembered across reboots.

Key capabilities:

- Wi‑Fi captive portal (AP) + optional STA network
- WebSocket live updates for relay, timer, PIR, energy, and system events
- Conflict-safe relay control:
  - PIR motion extends an AUTO “hold” window while the debounced PIR level remains active
  - Timers drive relay outputs and restore AUTO at end
  - Night Lock forces all relays OFF and cancels active timers
- Optional access control (MAC-based auth + restricted/unauthorized pages)
- Persistent configuration in Preferences/NVS; LittleFS is used for web UI assets only
- Activity History storage and the Activity Log UI have been removed to reduce flash writes and save space
- FreeRTOS task split + watchdog safety

## 1) Folder Layout

```
smart-home-automation-esp32/
  README.md
  platformio.ini
  SmartHomeAutomation/
    src/
      main.cpp              # PlatformIO / Arduino entry point
      Config.h
      SystemTypes.h
      Utils.h
      TimeKeeper.h/.cpp
      StorageLayer.h/.cpp
      ControlEngine.h/.cpp
      WebPortal.h/.cpp
    data/                   # LittleFS image (upload to flash)
      index.html
      restricted.html
      unauthorized.html
```

The project uses **PlatformIO**. The Arduino entry point is `src/main.cpp`; upload the `data/` folder as a LittleFS image so `/index.html` and the auth pages are available at runtime.

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

1. Open the project root (`platformio.ini`) in PlatformIO (CLI or VS Code extension).
2. Build + upload the LittleFS image first so `/index.html` and the HTML pages are available:
   - `pio run -t uploadfs`
3. Build + upload the firmware:
   - `pio run -t upload`
4. Connect to the device AP:
   - **AP SSID**: `tarshid`
   - **AP password**: `12345678`
   - (Access portal URL is the device softAP IP; captive portal redirects you.)
5. Open the device portal URL from the softAP IP returned/used by the captive redirect.

## 5) Core Logic

### Operating Modes (what actually drives outputs)

Each relay’s applied output is decided from these sources (highest wins):

`MANUAL > TIMER > PIR > NONE`

Key behaviors:

- **PIR motion**:
  - PIR events extend an “auto hold” window **only when the relay is in `AUTO` mode** and a timer is not active on that relay.
  - The hold window is level-triggered: as long as the debounced PIR input remains active, the controller extends the hold time. Motion events are still published only on the inactive-to-active edge to avoid event spam.
  - If a relay is `MANUAL ON/OFF`, PIR does not override it.
- **Timer**:
  - When a timer is active, the relay output follows the timer’s `targetState`.
  - When the timer ends, the relay returns to **`AUTO`** (restores previous relay state via `previousState`).
- **Night Lock** (safety freeze, opt-in):
  - The Night Lock feature is controlled by a user-facing master switch in the System Settings panel of the web UI (label: **“Enable Night Lock (قفل الليل)”**).
  - When the option is **ON** and the time phase is **NIGHT** (06:00–18:00 is DAY), the controller forces **all relays OFF** and cancels active timers. Manual ON, timer-ON, and PIR motion are all blocked during the night phase.
  - When the option is **OFF** (default after first boot or firmware upgrade), the controller ignores the night phase entirely. Manual ON, timers, and PIR motion work the same way at night as they do during the day.
  - The choice is persisted to NVS under the key `night_lock_en` and survives a reboot.
  - Flipping the option OFF while the lock is currently active releases it on the next control tick (within ~50 ms) — no need to wait for sunrise.
  - Certain operations that mutate persistent settings (changing PIR mapping, resetting consumption, updating rated power, toggling energy tracking) are still blocked *only* when the lock is currently active — i.e. only when the option is ON and the phase is NIGHT.

### Timer Lifecycle / Time alignment

- Timers store: `active`, `startEpoch`, `endEpoch`, `durationMinutes`, `targetState`, plus restore bookkeeping.
- Remaining time is always derived from absolute epoch timestamps once time is synced.

### Persistence

- **Preferences** (`PREF_NAMESPACE = "smart_home"`):
  - relay manual mode
  - timer plans
  - relay state/source
  - energy tracking enabled flag (`energy_en`)
  - **Night Lock option / user master switch** (`night_lock_en`, default `false`)
  - PIR mapping
  - rated power (and whether it is locked)
  - periodic housekeeping markers (daily cleanup)
  - user activity log controls (login inactivity handling)
- **LittleFS**:
  - web interface files such as `/index.html`, `/restricted.html`, and `/unauthorized.html`
  - no activity history, event log, or pending notification files are written at runtime
- Daily log cleanup is now a no-op because file-backed activity history has been removed.

### Runtime memory / flash-wear notes

- `ControlEngine::tickFast()` uses a fixed-size stack array for relay decisions, avoiding heap allocation in the high-frequency control loop.
- Real-time event generation remains active for WebSocket clients.
- Activity history persistence is disabled: `StorageLayer` log/pending methods are no-ops and `/api/logs` returns an empty JSON array (`[]`) for frontend compatibility.
- The frontend no longer renders the Activity Log panel or fetches `/api/logs`.

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
- `set_night_lock_option`
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

#### 6) `set_night_lock_option`
Toggle the Night Lock master switch. Persists to NVS (`night_lock_en`).
- `{"type":"set_night_lock_option","enabled":true}`   — enable forced-off at night
- `{"type":"set_night_lock_option","enabled":false}`  — disable (default)

Notes:
- The controller replies with `command_ack` (`ok: true/false`).
- On success a `state_snapshot` is broadcast to all connected clients.
- This command is intentionally **not** blocked while Night Lock is currently active — the whole point is to be able to turn the lock off mid-night.

#### 7) `get_state`
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
  - `night_lock.activated` / `night_lock.released`  *(the lock being currently active)*
  - `night_lock_option.changed`  *(the user toggled the Night Lock master switch)*
  - `energy_update` (energy tracking updates)
  - plus various connectivity/storage events (client connected/disconnected)

The `state_snapshot` payload exposes both the **current lock state** and the **user option** as separate fields, so the UI can render the toggle independently of the day phase:

- `state.nightLock`         — `true` while the lock is actively forcing relays OFF (option ON *and* phase NIGHT)
- `state.nightLockOption`   — `true` if the user has enabled the Night Lock master switch in Settings
- `state.energyTrackingEnabled` — `true` if energy tracking is enabled

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
