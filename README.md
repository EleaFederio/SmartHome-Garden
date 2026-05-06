# NFT ESPHome Setup

This folder contains ESPHome configs for the NFT hydroponics setup plus nursery controllers for seedling airflow and lighting.

## Files

- `pumpy.yaml`
  Controls the NFT water pump using an ESP8266 relay board.
- `nft_sensor.yaml`
  Reads TDS, pH, and DS18B20 water temperature on an ESP32-C3 and pauses noisy analog readings while the pump is running.
- `water_temp.yaml`
  Standalone ESP32-C3 config for a DS18B20 water temperature probe.
- `nursery/seedling_fan.yaml`
  ESP32-C3 nursery controller with a DHT22 humidity sensor, a transistor-driven 5V fan output, and a relay-controlled grow light schedule.

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
- Web server: enabled on port `80`
- TDS ADC pin: `GPIO0`
- pH `Po` ADC pin: `GPIO1`
- pH `Do` digital pin: `GPIO3`
- pH `To` ADC pin: `GPIO2`
- DS18B20 data pin: `GPIO4`

Exposed sensor values:

- `Hydroponic Water Temperature`
- `pH ADC`
- `pH Threshold Digital`
- `pH Module To Voltage`
- `pH`
- `TDS ADC`
- `TDS Raw ppm`
- `TDS ppm`
- `TDS Compensated Voltage`
- `Water Temperature`
- `TDS Calibration Scale`
- `TDS Calibration Offset`
- `pH Calibration Slope`
- `pH Calibration Offset`
- `pH Zero Point Reference`
- `Calibrate pH Zero Point`

### Standalone water temperature controller

- Device name: `water_temp`
- Board: `esp32-c3-devkitm-1`
- DS18B20 data pin: `GPIO4`
- Static IP: `192.168.1.115`

Exposed sensor values:

- `Hydroponic Water Temperature`

### Nursery seedling fan controller

- Device name: `seedling_fan`
- Board: `esp32-c3-devkitm-1`
- DHT22 data pin: `GPIO10`
- Fan control pin: `GPIO3`
- Grow light relay pin: `GPIO4`
- Time source: `sntp`

Exposed entities:

- `Seedling Temperature`
- `Seedling Humidity`
- `Seedling Fan`
- `Seedling Grow Light`

Automation behavior:

- Fan turns on when humidity rises above `80%`.
- Fan turns off when humidity falls below `65%`.
- Grow light turns on daily at `06:00`.
- Grow light turns off daily at `22:00`.
- On boot, the device checks current time and restores the correct light state for the active schedule window.

## ESP32-C3 Pinout Reference

This repo uses `esp32-c3-devkitm-1` for the sensor nodes. The table below is a practical wiring reference for the pins already used in these configs.

### Pins used in this repo

- `GPIO0`
  Used for `TDS ADC` in `nft_sensor.yaml`.
  This is also a strapping/boot-related pin, so avoid forcing it to the wrong level during reset.
- `GPIO1`
  Used for `pH ADC` in `nft_sensor.yaml`.
  Safe for analog input in this project and also used as an echo pin in `esp32_pump_waterlevel.yaml`.
- `GPIO4`
  Used for `DS18B20 DATA` in `nft_sensor.yaml` and `water_temp.yaml`.
  Good general-purpose digital pin for the OneWire bus.
- `GPIO10`
  Used for the `DHT22` data line in `nursery/seedling_fan.yaml`.
  Suitable for simple digital sensor input on ESP32-C3.

### ESP32-C3 wiring cautions

- `GPIO0`
  Boot-sensitive strapping pin. Be careful when attaching relays, pull-downs, or modules that may hold the pin low during startup.
- `GPIO2`
  Also commonly treated as a strapping-sensitive pin. The water level config notes that boot issues may require moving peripherals off this pin.
- `GPIO12`
  Do not use. It is reserved for the internal SPI flash on ESP32-C3 boards and is already called out in `esp32_pump_waterlevel.yaml`.

### Suggested sensor mapping for this repo

- `DS18B20 DATA` -> `GPIO4`
- `pH analog output (Po)` -> `GPIO1`
- `pH digital output (Do)` -> `GPIO3`
- `pH To output` -> `GPIO2`
- `TDS analog output` -> `GPIO0`
- `DHT22 DATA` -> `GPIO10`
- `Fan transistor input` -> `GPIO3`
- `Grow light relay input` -> `GPIO4`

### pH module pin mapping

- `V+` -> `3.3V` or `5V` based on the pH board specification
- `G` -> `GND`
- `G` -> `GND`
- `Po` -> `GPIO1`
- `Do` -> `GPIO3`
- `To` -> `GPIO2`

### pH module full-function behavior

With the current `nft_sensor.yaml`, all six pins on the pH board are connected and used like this:

- `V+`
  Powers the pH module board.
- `G` and `G`
  Shared ground between the pH module and the ESP32-C3.
- `Po`
  Main analog pH output. Published in ESPHome as `pH ADC`, then converted into the `pH` entity using the configured slope and offset.
- `Do`
  Digital threshold output. Published as `pH Threshold Digital`.
  This usually changes state when the board crosses the trigger level set by the onboard potentiometer.
- `To`
  Extra analog output. Published as `pH Module To Voltage`.
  It is currently exposed as a voltage only, because the exact meaning and conversion of `To` depends on the specific pH module board.

Important note:

- `Po` is the pin that provides the real pH measurement used by the `pH` entity.
- `Do` is not a second pH reading. It is only a threshold signal.
- `To` is not automatically a temperature sensor in ESPHome unless the exact module documentation confirms what that output represents and how to convert it.

## Wiring Notes

### DS18B20

- `VCC` -> `3.3V`
- `GND` -> `GND`
- `DATA` -> `GPIO4`
- Add a `4.7k` pull-up resistor between `DATA` and `3.3V`

### TDS module

- Analog output -> `GPIO0`
- `VCC` -> module-required power
- `GND` -> `GND`

### pH module

- `Po` / analog output -> `GPIO1`
- `VCC` -> module-required power
- `GND` -> `GND`
- `Do` / digital threshold output -> `GPIO3`
- `To` / extra analog output -> `GPIO2`

Important note:

- In the current ESPHome config, `Do` is exposed as `pH Threshold Digital`.
- `To` is exposed as `pH Module To Voltage`.
- `To` is currently read as a voltage because pH module boards do not all use the same conversion formula for that pin.
- If your exact module documentation includes a temperature formula for `To`, the YAML can be updated to publish it as real degrees Celsius.
- For most pH boards, `Po` is the actual measurement output and `Do` is only an adjustable comparator output.

### Nursery seedling controller

- `DHT22 DATA` -> `GPIO10`
- `DHT22 VCC` -> `3.3V` or module-rated supply
- `DHT22 GND` -> `GND`
- Add the usual DHT22 pull-up resistor if your breakout does not already include one.
- `Fan transistor control` -> `GPIO3`
- `Fan power` -> external `5V` supply sized for the fan
- `Fan ground` -> shared `GND` with the ESP32-C3
- `Grow light relay input` -> `GPIO4`

Important notes:

- The ESP32-C3 GPIO pin should drive the fan through a transistor or MOSFET, not power the fan directly.
- If the relay module or transistor stage is active-low, add `inverted: true` to that switch in `nursery/seedling_fan.yaml`.
- The light schedule in `nursery/seedling_fan.yaml` is currently fixed at `06:00-22:00` for a `16h on / 8h off` cycle.

## TDS Noise Protection

The sensor config is designed to avoid noisy analog readings caused by water movement and pump electrical noise.

How it works:

1. `nft_sensor.yaml` listens to the Home Assistant entity defined by `pump_switch_entity`.
2. When the pump turns on, these components are suspended:
   `ph_adc`, `ph_value`, `tds_adc`, `tds_raw_ppm`, `tds_calibrated_ppm`, and `tds_comp_voltage`.
3. When the pump turns off, the sensor waits for `sensor_resume_delay`.
4. After the delay, if the pump is still off, analog sensor updates resume and the TDS chain gets an immediate refresh.

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

pH is calculated in one step:

1. `pH`
   Applies a linear calibration to the measured pH module voltage:
   `ph = voltage * slope + offset`

Tunable pH calibration values:

- `pH Calibration Slope`
- `pH Calibration Offset`
- `pH Zero Point Reference`

Zero-point calibration workflow:

1. Rinse the pH probe and place it in a known reference solution.
2. Set `pH Zero Point Reference` to the value of that solution.
   For a standard neutral calibration, use `7.00`.
3. Wait for `pH ADC` to stabilize.
4. Press `Calibrate pH Zero Point`.
5. ESPHome recalculates and saves `pH Calibration Offset` using:
   `offset = reference_ph - (measured_voltage * slope)`

Important notes:

- This feature calibrates the `Po` analog output zero point by updating `pH Calibration Offset`.
- It does not change `pH Calibration Slope`.
- For best results, do zero-point calibration first, then fine-tune slope with a second buffer solution if needed.

Important note:

- `Hydroponic Water Temperature` is the real DS18B20 reading.
- `Water Temperature` is still a manual template value used by the current TDS compensation formula.
- If you want TDS to use the DS18B20 automatically, update the TDS lambdas to read `hydro_water_temp_c`.

## Network Settings

Current static IP assignments:

- Pump: `192.168.1.110`
- Sensor: `192.168.1.112`
- Water temperature: `192.168.1.115`

Make sure these addresses do not conflict with other devices on your network.

Web UI addresses:

- Sensor web server: `http://192.168.1.112/`

## Typical Workflow

1. Edit `pumpy.yaml`, `nft_sensor.yaml`, or `water_temp.yaml`.
   For nursery airflow and lighting, edit `nursery/seedling_fan.yaml`.
2. Validate the config with ESPHome.
3. Upload to the correct board.
4. Confirm the entities appear in Home Assistant.
5. Test pump on/off behavior and verify that analog readings stop updating while the pump is running.

## Suggested Checks After Flashing

- Switch pump mode between `Normal`, `Eco`, and `Super Eco`.
- Confirm the `NFT Pump` entity changes state correctly in Home Assistant.
- Verify the DS18B20 temperature reading updates on `GPIO4`.
- Verify the pH module reports a changing voltage on `GPIO1`.
- Turn the pump on and verify pH and TDS values stop updating.
- Turn the pump off and verify readings resume after about `30s`.
- Recheck `pump_switch_entity` if the sensor does not pause as expected.
- For `nursery/seedling_fan.yaml`, verify humidity updates on `GPIO10`, the fan turns on above `80%`, turns off below `65%`, and the grow light follows the `06:00-22:00` schedule.
