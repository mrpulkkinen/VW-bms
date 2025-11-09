# E-Golf Integration Notes

## Safety
- Pack >350VDC; discharge the DC link capacitor inside the contactor box before touching bus bars.
- No factory service disconnect; isolate mechanically and electrically before splitting modules.

## Module Harness
- Connector: TE 1-1670990-1 (aka 6R0 972 930).
- Pinout: 1=GND, 3=Enable 12V, 5=+12V, 6=CAN-H, 7=CAN-L.
- Each half-module needs the enable pin held high and 12V supply to wake the CMU.

## CAN Protocol
- CMUs expect a periodic 0x0BA poll frame (either all zeros or 0x45 0x01 0x28 0x00 0x00 0x00 0x00 0x30).
- Modules reply on 0x1CC–0x1D4; matches `BMSModuleManager::decodecan` mappings.
- `VWBMSV2.ino` already sets `controlid = 0x0BA`, so firmware is aligned with wiki docs.

## Firmware Touchpoints
- Update `VWBMSV2/CONFIG.H` with e-Golf series counts, current sensor choice, and charger thresholds.
- Use `SerialConsole` commands to toggle drive/charge modes and inspect module summaries while commissioning.
- `VW GTE E-Golf Can/VWtest.dbc` mirrors the published DBC for logging via SavvyCAN/Teensy_can.

## Outstanding Questions
- Document contactor/charger wiring and expected GPIO mapping for the pack harness.
- Confirm EEPROM layout migrations before flashing new configs into production packs.
