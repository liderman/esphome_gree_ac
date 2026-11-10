# Fenix UART command confirmation

This fork includes the driver revision tested on a Royal Thermo Fenix DC air-to-air heat pump with an ESP32-C3 and ESPHome 2026.9.1.

Based on upstream commit `48cc9d49e9ce2faf00a3d7b6ebba8075fb84c4ca`. The modified files are `components/sinclair_ac/esppac_cnt.cpp` and `esppac_cnt.h`.

## Behavior

The driver enforces the existing 300 ms minimum interval between transmissions, waits up to 700 ms for a response, and allows up to 900 ms for requested mode or target-temperature confirmation before sending the clearing frame. A checksum-valid unit report releases the response wait. Requested mode and temperature are compared with the unit report. Unconfirmed changes can be sent up to three times, with the clearing frame between attempts.

INFO messages report the attempt on which confirmation arrived. A warning records exhaustion of the retry limit. Confirmation covers the requested mode/power and target temperature. Fan speed and swing positions have no additional field-level confirmation in this change.

## Validation

Compiled and installed over OTA with ESPHome 2026.9.1. Tested sequences included 16 → 17 °C, a one-minute fan-only mixing cycle, automatic return to heating with 17 °C preserved, and return to 16 °C. A heating setpoint of 28 °C was also tested through the existing preheating automation, followed by fan-only, automatic heating restoration at 28 °C, and return to 16 °C.

The 28 °C sequence passed with both VERBOSE and INFO firmware builds. All four requested changes in each sequence were confirmed on the first attempt. Raw UART responses were inspected in the VERBOSE build. Timeout/retry exhaustion, other HVAC models, and sustained thermal performance were not exercised in these tests.

## Build again

Keep your device YAML, UART pin configuration, and secrets in your own backup. Use your existing device YAML and replace its external component source with this repository. For repeatable builds, set `ref` to the full tested commit SHA shown in GitHub history.

```yaml
external_components:
  - source:
      type: git
      url: https://github.com/liderman/esphome_gree_ac
      ref: 26fb3f55820768201fab8d95b6c3e82d2f41928f
    components: [sinclair_ac]

logger:
  baud_rate: 0
  level: INFO
```

Merge the logger settings into your existing block. Keeping `baud_rate: 0` prevents serial logging from using the HVAC UART.

From a directory containing your device YAML and its referenced secrets, use the tested ESPHome version.

```sh
docker run --rm -v "$PWD":/config ghcr.io/esphome/esphome:2026.9.1 config fenix.yaml
docker run --rm -v "$PWD":/config ghcr.io/esphome/esphome:2026.9.1 compile fenix.yaml
docker run --rm -it -v "$PWD":/config ghcr.io/esphome/esphome:2026.9.1 upload fenix.yaml --device DEVICE_IP
```

Replace `DEVICE_IP` with the module's reachable address. The final command uploads firmware and restarts the module. The Docker host must be able to reach the device. ESPHome downloads its build dependencies during compilation.

For diagnostics, rebuild with `level: VERBOSE` to include raw UART frames. Retain a known-working OTA image and its matching YAML and driver revision in a private backup. Compiled images contain device credentials.
