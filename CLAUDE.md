# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build Commands

```bash
make deps                              # Create .venv and install ESPHome
make compile BOARD=m5stack-nanoc6      # Validate config and compile binary
make upload BOARD=m5stack-nanoc6       # Compile, flash via serial, stream logs
make upload HOST_SUFFIX=-<hash>        # OTA upload to specific device
make logs HOST_SUFFIX=-<hash>          # Stream logs from device
make clean                             # Remove .venv and .esphome/
make compile_docker                    # Compile via Docker (no venv needed)
```

The Makefile derives `TARGET=$(PROJECT)-$(BOARD).yml` (e.g. `tesla-ble-m5stack-nanoc6.yml`). To build a custom-named yaml (like `carlita.yaml`), run ESPHome directly:

```bash
. .venv/bin/activate && esphome compile carlita.yaml
. .venv/bin/activate && esphome run carlita.yaml
```

There is no test suite. Build validation (`make compile`) is the primary correctness check.

## Architecture

This is an ESPHome external component project that bridges Tesla vehicles into Home Assistant via BLE. It is composed of:

### Package composition system

User config files (like `tesla-ble.example.yml` or `carlita.yaml`) pull in YAML packages via ESPHome's `packages:` mechanism. Packages can be loaded from the local `packages/` directory (for local/CLI builds) or from GitHub via `remote_package_files:` (for HA ESPHome add-on builds). The recommended approach for most users is remote packages.

**Package load order matters for substitution precedence** — the user's top-level yaml overrides package defaults. Packages loaded earlier in the list can be overridden by later ones:

| Package | Role |
|---------|------|
| `boards/m5stack-nanoc6.yml` | Board-specific: sets correct `board`/`variant` substitutions and sdkconfig, plus GPIO for LEDs and button. Load this **first** when targeting that board. |
| `packages/base.yml` | Defines all default substitutions (`board`, `variant`, `flash_size`, library URLs, poll timeouts). Sets ESP32 framework to `esp-idf`. Configures WiFi, OTA, API, logger. |
| `packages/common.yml` | Standard sensors: WiFi signal, uptime, BLE presence timeout, IP address, restart button, HA time sync. |
| `packages/project.yml` | Configures `esp32_ble_tracker` scan parameters; hooks to start/stop BLE on WiFi connect/disconnect. |
| `packages/client.yml` | The main integration: declares the `tesla_ble_vehicle` component with all 40+ sensors, switches, covers, buttons, locks, and number entities. |
| `packages/listener.yml` | Optional — scan for Tesla BLE beacon to discover the car's MAC address. Only needed during initial setup. |
| `packages/board.yml` | Pins ESP-IDF and PlatformIO versions. Used only for local builds. |

### Custom components (`components/`)

Two custom ESPHome C++ components:

**`tesla_ble_vehicle/`** — The core component. Extends `PollingComponent` and `BLEClientNode`. Manages the full BLE lifecycle: connect → GATT exchange → protobuf parse → sensor update → send commands. Pairing keys are persisted to ESP32 NVS flash so pairing survives reboots. The polling state machine differentiates between sleep/awake/charging states to balance HA responsiveness against battery drain.

**`tesla_ble_listener/`** — Lightweight BLE advertisement scanner. Parses Tesla's BLE beacon name format (e.g. `S1a87a5a75f3df858C`) to identify a car by VIN. Used only during initial setup.

Both components depend on the external C++ library at `https://github.com/PedroKTFC/tesla-ble` (set via `BLE_library` substitution in `base.yml`), which handles Tesla's protobuf spec and BLE crypto key derivation.

### Board configuration

The `boards/` directory is a legacy artifact from a prior fork and is not used. Board settings go inline as substitutions in your user yaml. For the M5Stack Nano C6:

```yaml
substitutions:
  board: "esp32-c6-devkitm-1"
  variant: "esp32c6"
  flash_size: "4MB"
```

Do **not** use `board: m5stack-nano-c6` — that is not a valid ESPHome board ID. See the [wiki](https://github.com/PedroKTFC/esphome-tesla-ble/wiki/How-to-build-for-different-board-types) for other supported boards.

### Required secrets

Before building, copy `secrets.yaml.example` → `secrets.yaml` and populate:

- `ble_mac_address` — the car's BLE MAC (found via BLE scanner app or `ble-scanner.yml`)
- `tesla_vin` — the vehicle VIN from the windscreen or Tesla app (e.g. `LRW3012...`), **not** the BLE beacon name
- `wifi_ssid`, `wifi_password`, `api_encryption_key`, `ota_password`, `wifi_hotspot_password`
