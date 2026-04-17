# NFT ESPHome Setup

This folder contains the ESPHome configs for the NFT hydroponics pump and nutrient sensor.

## Files

- `pumpy.yaml`
  Controls the NFT water pump using an ESP8266 relay board.
- `nft_sensor.yaml`
  Reads TDS-related values on an ESP32-C3 and pauses sensor updates while the pump is running to reduce reading noise.

## Devices

### Pump controller

- Device name: `pumpy`
- Board: `esp8266` / `esp01_1m`
- Main switch id: `nft_pump`
- Pump mode select id: `nft_pump_mode`

Available pump modes:

- `Normal`
  Pump stays on continuously.
- `Eco`
  Pump turns on for `15min`, then the interval triggers again every `20min`.
- `Super Eco`
  Pump turns on for `15min`, then the interval triggers again every `25min`.

Important hardware note:

- The relay uses `GPIO0` with `inverted: true`.
- `GPIO0` is a boot-sensitive pin on ESP8266. If flashing fails, disconnect the relay input or use a temporary jumper/setup that allows normal boot.

### Sensor controller

- Device name: `nft-sensor`
- Board: `esp32-c3-devkitm-1`
- ADC pin: `GPIO0`

Exposed sensor values:

- `TDS ADC`
- `TDS Raw ppm`
- `TDS ppm`
- `TDS Compensated Voltage`
- `Water Temperature`
- `TDS Calibration Scale`
- `TDS Calibration Offset`

## TDS Noise Protection

The sensor config is designed to avoid noisy TDS/EC readings caused by water movement and pump electrical noise.

How it works:

1. `nft_sensor.yaml` listens to the Home Assistant entity defined by `pump_switch_entity`.
2. When the pump turns on, these components are suspended:
   `tds_adc`, `tds_raw_ppm`, `tds_calibrated_ppm`, and `tds_comp_voltage`.
3. When the pump turns off, the sensor waits for `sensor_resume_delay`.
4. After the delay, if the pump is still off, sensor updates resume and an immediate refresh is triggered.

Default values:

- `pump_switch_entity: switch.nft_pump`
- `sensor_resume_delay: 30s`

If your Home Assistant entity id is different, update this value in `nft_sensor.yaml`.

## Home Assistant Dependency

The pause/resume logic in `nft_sensor.yaml` depends on Home Assistant because it uses:

- `binary_sensor:`
- `platform: homeassistant`

This means:

- The sensor device must be connected to Home Assistant through the ESPHome API.
- The pump switch entity must exist in Home Assistant.
- If the entity id changes, the pause logic will stop working until `pump_switch_entity` is updated.

## Calibration Notes

TDS is calculated in two steps:

1. `TDS Raw ppm`
   Converts ADC voltage into a raw TDS value using the polynomial formula and water temperature compensation.
2. `TDS ppm`
   Applies user calibration:
   `calibrated = raw * scale + offset`

Tunable calibration values:

- `TDS Calibration Scale`
- `TDS Calibration Offset`
- `Water Temperature`

## Network Settings

Current static IP assignments:

- Pump: `192.168.1.110`
- Sensor: `192.168.1.112`

Make sure these addresses do not conflict with other devices on your network.

## Typical Workflow

1. Edit `pumpy.yaml` or `nft_sensor.yaml`.
2. Validate the config with ESPHome.
3. Upload to the correct board.
4. Confirm the entities appear in Home Assistant.
5. Test pump on/off behavior and verify that TDS values stop updating while the pump is running.

## Suggested Checks After Flashing

- Switch pump mode between `Normal`, `Eco`, and `Super Eco`.
- Confirm the `NFT Pump` entity changes state correctly in Home Assistant.
- Turn the pump on and verify TDS values stop updating.
- Turn the pump off and verify readings resume after about `30s`.
- Recheck `pump_switch_entity` if the sensor does not pause as expected.
