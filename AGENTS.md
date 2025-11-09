# Repository Guidelines

## Project Structure & Module Organization
`VWBMSV2/` holds the active Teensy 3.2 firmware: `VWBMSV2.ino` bootstraps the system, `BMSModule*` manage CMU data, `SerialConsole.*` handles CLI diagnostics, and `Logger.*` persists pack state. Support sketches live in `Teensy_can/` and `VWbms/` for quick experiments, while `VW GTE E-Golf Can/` stores reference DBC files and capture logs. Hardware-specific defaults and EEPROM mirrors are centralized in `VWBMSV2/CONFIG.H`; keep board profiles and pin maps there. Generated HEX files land in `VWBMSV2/build/`.

## Build, Test, and Development Commands
Use the Teensy/Arduino toolchain locally; install the Teensy core once via `arduino-cli core install teensy:avr`. Common commands:
```
arduino-cli compile --fqbn teensy:avr:teensy31 VWBMSV2        # build for Teensy 3.2
arduino-cli upload --fqbn teensy:avr:teensy31 -p /dev/ttyACM0 VWBMSV2
teensy_loader_cli --mcu=TEENSY32 -w VWBMSV2/build/VWBMSV2.ino.hex
arduino-cli monitor -p /dev/ttyACM0 -c baudrate=115200        # watch SerialConsole output
```
Rebuild whenever `CONFIG.H` or any module changes; cached artifacts will otherwise keep stale calibrations.

## Coding Style & Naming Conventions
Match the existing two-space indentation, braces on the same line as declarations, and `UpperCamelCase` for classes (`BMSModuleManager`) with lower camelCase for methods/variables (`decodecan`, `lowpassFilter`). Keep constants `ALL_CAPS` and prefer `constexpr` over macros where possible. When adding files, route includes through project-relative headers, and update `CONFIG.H` for any new pin, CAN ID, or threshold so downstream agents can audit changes quickly.

## Testing Guidelines
There is no automated unit test harness yet; rely on bench validation. Before flashing, enable verbose logging by toggling the relevant `DEBUG_LEVEL` flags in `Logger.cpp`, then exercise charge, drive, and balance modes via `SerialConsole` commands. Capture CAN traffic with the `Teensy_can` sketch to confirm frame layouts whenever you touch `decodecan`/`decodebal` logic. Document measured pack voltages and temperatures in your PR if they vary from configured values.

## Commit & Pull Request Guidelines
Commits follow short, imperative subjects (`Update balancing check`, `SOH to reflect cell gap`). Reference the touched subsystem (`BMSModuleManager`, `SerialConsole`, `CONFIG`). For PRs, include: problem statement, summary of firmware behavior changes, hardware/firmware versions used for validation, console logs or CAN captures that prove success, and any remaining risks. Link related issues and call out EEPROM layout changes so other contributors can plan migrations.

## Configuration & Safety Tips
Treat `CONFIG.H`, EEPROM schema, and `EEPROMSettings` structures as the single source of truth for pack geometry, sensor type, and charger profiles. Validate over-current/over-temp thresholds against the real pack before merging. Never commit customer-specific CAN keys or VIN data; store those in local `*.local.h` files ignored by Git if you must keep templates.

## E-Golf Integration Checklist
- Safety: e-Golf packs stay above 350 V and hide a DC-link capacitor inside the contactor box; discharge it with a resistor before loosening any HV hardware.
- Harness: half-modules expose connector TE 1-1670990-1 (alias 6R0 972 930). Wire pin 1=GND, 3=Enable 12 V, 5=+12 V, 6=CAN-H, 7=CAN-L, then daisy-chain CAN to the Teensy.
- CAN protocol: firmware already transmits the required `0x0BA` poll (`VWBMSV2.ino:190`) and listens for module replies on `0x1CC–0x1D4` inside `BMSModuleManager::decodecan`. Verify IDs against `VW GTE E-Golf Can/VWtest.dbc` when sniffing traffic.
- Configuration: set the pack series count, charger thresholds, and sensor options in `VWBMSV2/CONFIG.H`, then document your measured volt/temperature limits in PRs.
- More detail lives in `e-golf-notes.md`; keep it updated with wiring changes, contactor control expectations, and EEPROM migration steps.
