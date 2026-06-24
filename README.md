# ESPHome Configs

ESPHome YAML files for a small hydroponics and nursery setup. The repo includes NFT pump and sensor nodes, nursery automation nodes, and a couple of water-level test/utility configs.

## Repository Layout

- `nft/pumpy.yaml`
  ESP8266 relay node for the NFT pump with selectable operating modes.
- `nft/nft_sensor.yaml`
  ESP32-C3 sensor node for pH, TDS, and DS18B20 water temperature.
- `nft/water_temp.yaml`
  ESP32-C3 standalone DS18B20 water temperature probe.
- `nursery/seedling_bot.yaml`
  ESP32-C3 nursery controller with BH1750 light sensing, fan relay, and grow light scheduling.
- `nursery/light.yaml`
  ESP8266 relay node for a nursery light.
- `esp32_pump_waterlevel.yaml`
  ESP32-C3 ultrasonic water tank level monitor with derived level values.
- `esp32_mini_s2.yaml`
  Simple ultrasonic reference/test config. Despite the filename, it currently targets an ESP32-C3 board.
- `home_assistant_scipts/nft_pump_schedule.yaml`
  Home Assistant automation that switches NFT pump modes throughout the day.
- `style.css`
  Small CSS override file kept in the repo for quick UI styling tests.

## Devices

### NFT Pump Controller

- File: `nft/pumpy.yaml`
- Device name: `pumpy`
- Friendly name: `Pumpy`
- Board: `esp8266` / `esp01_1m`
- Static IP: `192.168.1.110`
- Web server: enabled on port `80`
- Relay pin: `GPIO0` with `inverted: true`
- SNTP mode schedule: `06:00 Eco`, `11:00 Noon-Time`, `15:00 Eco`, `18:00 Super Eco`

Entities:

- `NFT Pump`
- `NFT Pump Mode`

Pump modes:

- `Normal`
  Pump is turned on immediately and stays on until the mode changes.
- `Eco`
  Runs continuously in a loop: `15min` on, `45min` off.
- `Super Eco`
  Runs continuously in a loop: `15min` on, `2h` off.
- `Noon-Time`
  Runs continuously in a loop: `15min` on, `15min` off.
- `Emergency`
  Runs continuously in a loop: `5min` on, `5min` off.

Behavior:

- On each mode change, the config stops any previously running cycle script before starting the selected one.
- The node also has internal SNTP-based schedule changes at `06:00`, `11:00`, `15:00`, and `18:00`.
- `Normal` is the only mode that leaves the pump on continuously.

Hardware note:

- `GPIO0` is boot-sensitive on ESP8266. If flashing becomes unreliable, disconnect or isolate the relay input during boot.

### NFT Sensor Controller

- File: `nft/nft_sensor.yaml`
- Device name: `nft-sensor`
- Friendly name: `NFT-sensor`
- Board: `esp32-c3-devkitm-1`
- Static IP: `192.168.1.112`
- Web server: enabled on port `80`
- DS18B20 pin: `GPIO4`
- TDS ADC pin: `GPIO0`
- pH `Po` pin: `GPIO1`
- pH `Do` pin: `GPIO3`
- pH `To` pin: `GPIO2`
- Home Assistant pump entity substitution: `switch.nft_pump`
- Sensor resume delay substitution: `30s`

Entities:

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
- `Calibrate pH 4.01`
- `Calibrate pH 6.86`
- `Calibrate pH 9.18`
- `NFT Pump Running Sync`

Behavior:

- TDS and pH analog updates are paused while the linked Home Assistant pump entity is on.
- After the pump turns off, the node waits for `sensor_resume_delay` before resuming updates.
- When updates resume, the config forces immediate refreshes for the TDS sensors.
- The node exposes three pH calibration buttons for standard buffer solutions: `4.01`, `6.86`, and `9.18`.
- To calibrate, immerse the probe in the selected buffer, press the matching button, and the current ADC reading is used to update the stored pH offset for the live calculation.

### Standalone Water Temperature Probe

- File: `nft/water_temp.yaml`
- Device name: `water_temp`
- Friendly name: `water_temp`
- Board: `esp32-c3-devkitm-1`
- Static IP: `192.168.1.115`
- DS18B20 pin: `GPIO4`

Entities:

- `Hydroponic Water Temperature`

### Nursery Controller

- File: `nursery/seedling_bot.yaml`
- Device name: `seedling_fan`
- Friendly name: `Seedling Fan`
- Board: `esp32-c3-devkitm-1`
- Static IP: `192.168.1.118`
- Time zone: `Asia/Manila`
- Main I2C bus: `GPIO3` SDA / `GPIO4` SCL for the BH1750
- Expansion I2C bus: `GPIO6` SDA / `GPIO7` SCL for the AHT20
- Fan relay: `GPIO5`
- Grow light relay: `GPIO10`
- BH1750 address: `0x23`
- AHT20/AHT10 address: `0x38`

Entities:

- `Light Level`
- `Temperature`
- `Humidity`
- `Seedling Fan`
- `Seedling Grow Light`

Automation behavior:

- On boot, the grow light is restored based on the current time if SNTP time is already valid.
- The grow light turns on daily at `06:00`.
- The grow light turns off daily at `22:00`.

### Nursery Light Relay

- File: `nursery/light.yaml`
- Device name: `t5_light`
- Friendly name: `T5 Light`
- Board: `esp8266` / `esp01_1m`
- Static IP: `192.168.1.111`
- Web server: enabled on port `80`
- Relay pin: `GPIO0` with `inverted: true`
- Time zone: `Asia/Manila`
- Daily schedule: on at `05:00`, off at `21:00`
- Relay restore mode: `RESTORE_DEFAULT_OFF`

Entities:

- `Nursery Light Relay`

Behavior:

- The node runs its daily light schedule locally using SNTP time.
- After boot, a `5s` interval checks for valid time and then corrects relay state based on the current hour.
- Once the initial recovery check finishes, the interval suspends itself.

### Water Tank Level Monitor

- File: `esp32_pump_waterlevel.yaml`
- Device name: `pump-waterlevel`
- Board: `esp32-c3-devkitm-1`
- Web server: enabled on port `80`
- Ultrasonic trigger pin: `GPIO2`
- Ultrasonic echo pin: `GPIO1`
- Distance unit: meters

Entities:

- `Water Tank Distance`
- `Water Level`
- `Water Level Percentage`

Behavior:

- `Water Level` is calculated as `(0.68 - distance) * 100` and reported in centimeters.
- `Water Level Percentage` is calculated as `((0.68 - distance) / 0.68) * 100`.

Hardware notes:

- `GPIO2` is a strapping pin on ESP32-C3. If boot becomes unreliable, move the trigger wire to a safer pin and update the YAML.
- The commented relay example notes that `GPIO12` should not be used on ESP32-C3 because it is reserved for internal flash.

### Alternate Water-Level Test Node

- File: `esp32_mini_s2.yaml`
- Device name: `water_level`
- Friendly name: `water_level`
- Board: `esp32-c3-devkitm-1`
- Web server: enabled on port `80`
- Ultrasonic trigger pin: `GPIO1`
- Ultrasonic echo pin: `GPIO2`
- Distance unit: centimeters

Entities:

- `Water Tank Distance`

## Home Assistant Automation

### NFT Pump Schedule

- File: `home_assistant_scipts/nft_pump_schedule.yaml`
- Alias: `NFT Pump: Schedule Modes`
- Target entity: `select.pumpy_nft_pump_mode`

Behavior:

- Sets pump mode to `Eco` at `06:00`.
- Sets pump mode to `Noon-Time` at `11:00`.
- Sets pump mode to `Eco` at `15:00`.
- Sets pump mode to `Super Eco` at `18:00`.
- On Home Assistant startup, it picks the correct mode based on the current time.

Note:

- The folder name is currently `home_assistant_scipts` in the repo.
- This schedule overlaps with the internal SNTP schedule already defined in `nft/pumpy.yaml`.

## Wiring Notes

### DS18B20

- `VCC` -> `3.3V`
- `GND` -> `GND`
- `DATA` -> `GPIO4`
- Add a `4.7k` pull-up resistor between `DATA` and `3.3V`

### TDS Module

- Analog output -> `GPIO0`
- `VCC` -> module-required supply
- `GND` -> `GND`

### pH Module

- `Po` -> `GPIO1`
- `Do` -> `GPIO3`
- `To` -> `GPIO2`
- `VCC` -> module-required supply
- `GND` -> `GND`

Notes:

- `Po` is the analog input used to compute the `pH` entity.
- `Do` is exposed as `pH Threshold Digital`.
- `To` is exposed as `pH Module To Voltage`.

### Nursery Controller

- `BH1750 SDA` -> `GPIO3`
- `BH1750 SCL` -> `GPIO4`
- `AHT20/AHT10 SDA` -> `GPIO6`
- `AHT20/AHT10 SCL` -> `GPIO7`
- `Fan relay input` -> `GPIO5`
- `Grow light relay input` -> `GPIO10`

## ESP32-C3 Pin Notes

Pins used in this repo:

- `GPIO0`
  Used for the NFT TDS ADC input.
- `GPIO1`
  Used for pH analog input and ultrasonic signaling.
- `GPIO2`
  Used for pH module `To` and ultrasonic signaling.
- `GPIO3`
  Used for the pH digital threshold input and the nursery fan relay.
- `GPIO4`
  Used for DS18B20 data and the nursery grow light relay.
- `GPIO6`
  Used for I2C SDA.
- `GPIO7`
  Used for I2C SCL.

Cautions:

- `GPIO0` and `GPIO2` are strapping-sensitive on ESP32-C3.
- `GPIO12` should not be used on ESP32-C3 boards with internal flash.

## Calibration Notes

TDS calculation flow:

1. `TDS ADC` reads the raw voltage.
2. `TDS Raw ppm` applies temperature compensation and the standard polynomial.
3. `TDS ppm` applies user calibration with `calibrated = raw * scale + offset`.

pH calibration flow:

1. Immerse the pH probe in a known buffer solution such as `4.01`, `6.86`, or `9.18`.
2. Press the matching calibration button for that buffer.
3. The node stores the resulting offset and uses the calibrated slope/offset values in the live `pH` calculation.

Tunable TDS values:

- `TDS Calibration Scale`
- `TDS Calibration Offset`
- `Water Temperature`

Tunable water temperature values:

- `Water Temperature Calibration Scale`
- `Water Temperature Calibration Offset`

pH calculation flow:

1. `pH ADC` reads the module voltage from `Po`.
2. `pH` applies `ph = voltage * slope + offset`.
3. The result is clamped to the `0` to `14` range.

Tunable pH values:

- `pH Calibration Slope`
- `pH Calibration Offset`
- `pH Calibration Buffer`

Buffer calibration:

1. Place the probe in a known buffer solution (4.01, 6.86, or 9.18).
2. Set `pH Calibration Buffer` to the matching solution.
3. Wait for `pH ADC` to stabilize.
4. Press `Calibrate pH Buffer`.

This updates the pH offset so `pH` is calculated from the live ADC voltage using the chosen buffer reference.

## Network Notes

Static IPs currently set in the configs:

- `192.168.1.110` -> `nft/pumpy.yaml`
- `192.168.1.111` -> `nursery/light.yaml`
- `192.168.1.112` -> `nft/nft_sensor.yaml`
- `192.168.1.115` -> `nft/water_temp.yaml`
- `192.168.1.118` -> `nursery/seedling_bot.yaml`

Configs without a fixed IP in this repo:

- `esp32_pump_waterlevel.yaml`
- `esp32_mini_s2.yaml`

Nodes with `web_server` enabled expose a browser UI on port `80`.

## Typical Workflow

1. Edit the target YAML file.
2. Validate it with ESPHome.
3. Flash the correct board.
4. Confirm the device comes online in ESPHome and Home Assistant.
5. Verify the expected sensors, relays, or schedules on the hardware.
