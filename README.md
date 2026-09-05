# Weather Station for ESP32 and ESPHome

This fork is an ESPHome implementation of the original weather station for Home Assistant. The main profile targets ESP32-C6 DevKit boards with 8 MB flash; standalone profiles for ESP32/WROOM, C3, S2 and S3 are in `esphome/`.

## Install in Home Assistant

Use the complete [ESP32-C6 configuration](weatherstation.yaml). Copy the entire file into a new device in ESPHome Device Builder. Each file is standalone; no package files are needed.

| File | Chip | SDA | SCL | Wind | Rain |
|---|---|---:|---:|---:|---:|
| `weatherstation.yaml` | ESP32-C6 board | GPIO0 | GPIO1 | GPIO2 | GPIO3 |
| `esphome/weatherstation-esp32.yaml` | ESP32/WROOM | GPIO21 | GPIO22 | GPIO25 | GPIO26 |
| `esphome/weatherstation-c3.yaml` | ESP32-C3 | GPIO4 | GPIO5 | GPIO6 | GPIO7 |
| `esphome/weatherstation-s3.yaml` | ESP32-S3 | GPIO8 | GPIO9 | GPIO4 | GPIO5 |
| `esphome/weatherstation-s2.yaml` | ESP32-S2 | GPIO8 | GPIO9 | GPIO4 | GPIO5 |

Before installing, replace `wifi_ssid`, `wifi_password`, `api_key`, `ota_password` and `ap_password` in `substitutions`. The example values are public placeholders. Use a fresh 32-byte base64 API key and private passwords. Click **Validate**, then **Install**. The first install is over USB; later installs, logs and updates are managed from ESPHome. If the board is not detected, hold BOOT while pressing RESET. Add the discovered device under **Settings → Devices & services**.

ESPHome uses the encrypted native Home Assistant API and OTA. MQTT and the original web server are not required. WiFi must be 2.4 GHz. A protected fallback AP is available when the station cannot connect.

## Wiring and sensors

Connect all I²C sensors in parallel to the listed SDA/SCL pins and common ground. Use 3.3 V pull-ups. ESP32 GPIOs are not 5 V tolerant. The A3144 output must be open collector with a pull-up to 3.3 V, or use a level shifter if its module pulls up to 5 V.

The configuration supports SHT30 (`0x44`), BMP280 (`0x76`), ENS160 (`0x53`), BH1750 (`0x23`), AS5600 (`0x36`), a wind pulse input and a rain pulse input. Connect the SHT30's four wires as 3.3 V, GND, SDA and SCL. Missing sensors report unavailable while the rest of the device continues running. Change `sht30_address` to `0x45`, `bmp280_address` to `0x77` or `ens160_address` to `0x52` when required.

Calibration values are substitutions near the top of each file: altitude and offsets, rain millimetres per tip, anemometer radius, magnet count, aerodynamic factor and light transmission. Point the vane north and press **Calibrate north** in Home Assistant. Wind speed is reported in m/s, averaged over five two-second samples; gust is the maximum over ten minutes. Rain total and north offset are stored in flash with a five-minute write interval.

## Local validation

```sh
python3 -m venv .venv
.venv/bin/pip install -r requirements-dev.txt
.venv/bin/esphome config weatherstation.yaml
.venv/bin/esphome compile weatherstation.yaml
```

GitHub Actions validates and builds all five profiles. Hardware testing is still needed for the exact board pinout, sensor addresses, pulse polarity, WiFi and OTA.

The old ESP8266 implementation and unrelated original assets were removed from the working tree; they remain available in the upstream repository and Git history.
