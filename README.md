# ESPHome Configs

This repository contains ESPHome YAML files for a small hydroponics and nursery setup, plus a couple of water-level utility nodes.

## Repository Layout

- `nft/pumpy.yaml`
  ESP8266 NFT pump relay with selectable pump modes.
- `nft/nft_sensor.yaml`
  ESP32-C3 NFT sensor node for TDS, pH, and DS18B20 water temperature.
- `nft/water_temp.yaml`
  ESP32-C3 standalone DS18B20 water temperature probe.
- `nursery/seedling_bot.yaml`
  ESP32-C3 nursery controller with BH1750 light sensing, fan relay, and grow light relay.
- `nursery/light.yaml`
  ESP8266 nursery light relay node.
- `esp32_pump_waterlevel.yaml`
  ESP32-C3 ultrasonic tank level monitor with derived water level values.
- `esp32_mini_s2.yaml`
  Alternate ultrasonic water-level test config stored for reference.

## Device Summary

### NFT pump controller

- File: `nft/pumpy.yaml`
- Device name: `pumpy`
- Friendly name: `Pumpy`
- Board: `esp8266` / `esp01_1m`
- Static IP: `192.168.1.110`
- Web server: enabled on port `80`
- Relay pin: `GPIO0` with `inverted: true`

Exposed entities:

- `NFT Pump`
- `NFT Pump Mode`

Pump modes:

- `Normal`
  Pump stays on.
- `Eco`
  Turns on for `15min`, then repeats on a `30min` interval.
- `Super Eco`
  Turns on for `15min`, then repeats on a `2h` interval.

Hardware note:

- `GPIO0` is boot-sensitive on ESP8266. If flashing becomes unreliable, disconnect the relay input or isolate it during boot.

### NFT sensor controller

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

Exposed entities:

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
- `NFT Pump Running Sync`

Behavior:

- TDS and pH analog updates pause while the Home Assistant pump entity is on.
- Readings resume after `sensor_resume_delay`, which defaults to `30s`.
- pH zero-point calibration updates `pH Calibration Offset` from the live `pH ADC` value and the selected reference solution.

Home Assistant dependency:

- `NFT Pump Running Sync` uses `platform: homeassistant`.
- Default linked entity: `switch.nft_pump`

### Standalone water temperature probe

- File: `nft/water_temp.yaml`
- Device name: `water_temp`
- Friendly name: `water_temp`
- Board: `esp32-c3-devkitm-1`
- Static IP: `192.168.1.115`
- DS18B20 pin: `GPIO4`

Exposed entities:

- `Hydroponic Water Temperature`

### Nursery controller

- File: `nursery/seedling_bot.yaml`
- Device name: `seedling_fan`
- Friendly name: `Seedling Fan`
- Board: `esp32-c3-devkitm-1`
- Static IP: `192.168.1.118`
- Time zone: `Asia/Manila`
- I2C SDA: `GPIO6`
- I2C SCL: `GPIO7`
- Fan relay: `GPIO3`
- Grow light relay: `GPIO4`
- BH1750 address: `0x23`

Exposed entities:

- `Light Level`
- `Seedling Fan`
- `Seedling Grow Light`

Automation behavior:

- On boot, the grow light is restored according to the current time if SNTP time is already valid.
- The grow light turns on daily at `06:00`.
- The grow light turns off daily at `22:00`.

### Nursery light relay

- File: `nursery/light.yaml`
- Device name: `t5_light`
- Friendly name: `T5 Light`
- Board: `esp8266` / `esp01_1m`
- Static IP: `192.168.1.111`
- Web server: enabled on port `80`
- Relay pin: `GPIO0` with `inverted: true`
- Relay restore mode: `RESTORE_DEFAULT_ON`

Exposed entities:

- `Nursery Light Relay`

### Water tank level monitor

- File: `esp32_pump_waterlevel.yaml`
- Device name: `pump-waterlevel`
- Board: `esp32-c3-devkitm-1`
- Web server: enabled on port `80`
- Ultrasonic trigger pin: `GPIO2`
- Ultrasonic echo pin: `GPIO1`

Exposed entities:

- `Water Tank Distance`
- `Water Level`
- `Water Level Percentage`

Behavior:

- Distance is measured in meters.
- `Water Level` is calculated as `(0.68 - distance) * 100` and reported in centimeters.
- `Water Level Percentage` is calculated against a `0.68m` tank height reference.

Hardware notes:

- `GPIO2` is a strapping pin on ESP32-C3. If the board becomes hard to boot, move the trigger wire to a safer pin and update the YAML.
- The commented relay example in this file notes that `GPIO12` should not be used on ESP32-C3 because it is reserved for internal flash.

### Alternate water-level test node

- File: `esp32_mini_s2.yaml`
- Device name: `water_level`
- Friendly name: `water_level`
- Board: `esp32-c3-devkitm-1`
- Web server: enabled on port `80`
- Ultrasonic trigger pin: `GPIO1`
- Ultrasonic echo pin: `GPIO2`

Exposed entities:

- `Water Tank Distance`

## ESP32-C3 Pin Notes

Pins used in this repo:

- `GPIO0`
  Used for the NFT TDS ADC input.
- `GPIO1`
  Used for pH analog input and one ultrasonic signal.
- `GPIO2`
  Used for pH module `To` and one ultrasonic signal.
- `GPIO3`
  Used for pH threshold digital input and the nursery fan relay.
- `GPIO4`
  Used for DS18B20 data and the nursery grow light relay.
- `GPIO6`
  Used for I2C SDA.
- `GPIO7`
  Used for I2C SCL.

Cautions:

- `GPIO0` and `GPIO2` are strapping-sensitive on ESP32-C3.
- `GPIO12` should not be used on ESP32-C3 boards with internal flash.

## Wiring Notes

### DS18B20

- `VCC` -> `3.3V`
- `GND` -> `GND`
- `DATA` -> `GPIO4`
- Add a `4.7k` pull-up resistor between `DATA` and `3.3V`

### TDS module

- Analog output -> `GPIO0`
- `VCC` -> module-required supply
- `GND` -> `GND`

### pH module

- `Po` -> `GPIO1`
- `Do` -> `GPIO3`
- `To` -> `GPIO2`
- `VCC` -> module-required supply
- `GND` -> `GND`

Notes:

- `Po` is the analog input used to compute the `pH` entity.
- `Do` is exposed as `pH Threshold Digital`.
- `To` is exposed as `pH Module To Voltage`.

### Nursery controller

- `BH1750 SDA` -> `GPIO6`
- `BH1750 SCL` -> `GPIO7`
- `Fan relay input` -> `GPIO3`
- `Grow light relay input` -> `GPIO4`

## Calibration Notes

TDS calculation flow:

1. `TDS ADC` reads the raw voltage.
2. `TDS Raw ppm` applies the standard polynomial plus temperature compensation.
3. `TDS ppm` applies user calibration with:
   `calibrated = raw * scale + offset`

Tunable TDS values:

- `TDS Calibration Scale`
- `TDS Calibration Offset`
- `Water Temperature`

pH calculation flow:

1. `pH ADC` reads the module voltage from `Po`.
2. `pH` applies:
   `ph = voltage * slope + offset`

Tunable pH values:

- `pH Calibration Slope`
- `pH Calibration Offset`
- `pH Zero Point Reference`

Zero-point calibration:

1. Place the probe in a known reference solution.
2. Set `pH Zero Point Reference`.
3. Wait for `pH ADC` to stabilize.
4. Press `Calibrate pH Zero Point`.

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

Web UI endpoints for nodes with `web_server` enabled will use the device IP assigned on your network.

## Typical Workflow

1. Edit the target YAML file.
2. Validate it with ESPHome.
3. Flash the correct board.
4. Confirm the device comes online in ESPHome and Home Assistant.
5. Verify the expected sensors, relays, or schedules on hardware.
