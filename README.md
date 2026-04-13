# RO Meter

This workspace contains a first-pass ESPHome configuration for an RO water meter built around an ESP32-C3 OLED dev board plus two inline pulse flow sensors:

- `ro_meter.yaml` is the firmware config for the ESP32-C3.
- `home_assistant_ro_meter_package.yaml` is a Home Assistant package that derives daily ratio metrics from the ESPHome totals.

## What the device exposes

The ESPHome node publishes:

- purified water flow rate
- waste water flow rate
- purified water lifetime total
- waste water lifetime total
- purified water today total
- waste water today total
- current waste ratio
- run active state

In Home Assistant, these will show up under the `ro_meter` device prefix, for example `sensor.ro_meter_product_volume_total`.

The OLED shows:

- current ratio when the system is running
- imported rolling daily ratio when the system is idle, if that Home Assistant entity exists
- today's purified and waste totals
- a short status label: `RUN`, `IDLE`, or `WIFI`

## Board-specific details confirmed

The board page you shared matches the common 0.42" ABRobot-style ESP32-C3 OLED board and confirms:

- onboard OLED is `SSD1306 72x40`
- OLED I2C address is `0x3C`
- OLED is wired to `GPIO5` (`SDA`) and `GPIO6` (`SCL`)
- onboard LED is on `GPIO8` and is active-low
- BOOT button is on `GPIO9`

That means the current config is now aligned with the board for the display side.

## Important assumptions to review

The board details are now much tighter, but you should still verify the sensor wiring before flashing:

- `board: esp32-c3-devkitm-1`
- `i2c_sda_pin: GPIO5`
- `i2c_scl_pin: GPIO6`
- `product_sensor_pin: GPIO2`
- `waste_sensor_pin: GPIO3`
- `model: "SSD1306 72x40"`
- `address: 0x3C`

The flow sensor pins are still a project choice. The current defaults intentionally avoid:

- `GPIO5` and `GPIO6` because they are used by the OLED
- `GPIO8` because it drives the onboard LED
- `GPIO9` because it is the BOOT button

## Calibration

The current config now assumes the flow meters are `FL-S402B` units.

Published specs for this family are inconsistent across sellers and reposted datasheets:

- your sensor listing reports `F = 23 x Q` where `Q` is in L/min, which implies about `1380 pulses/liter`
- multiple manuals list `F = 32 x Q`, which implies about `1920 pulses/liter`
- some seller pages list `F = 38 x Q`, which implies about `2280 pulses/liter`

Because of that, `ro_meter.yaml` now starts at `1380 pulses/liter` to match your specific listing, but you should treat that only as a first-pass estimate and calibrate both sensors after installation.

Each sensor uses a `*_pulses_per_liter` substitution. Start with the default value, then calibrate:

1. Run a measured amount of water through one sensor.
2. Read the pulse total in Home Assistant or the ESPHome logs.
3. Set `pulses_per_liter = measured_pulses / measured_liters`.
4. Repeat for both the product and waste sensors.

If your measured result is closer to one of the other common variants, update the corresponding constant toward `1920` or `2280`.

## FL-S402B wiring notes

For the common `FL-S402B` / `YF-S402B` style sensors, the published wire colors are typically:

- red: sensor supply
- black: ground
- yellow: pulse output

Important: many product listings describe the yellow lead as an **NPN pulse output**. That usually means the sensor output should be pulled up to the microcontroller logic voltage rather than fed directly with a 5V-high signal.

For this ESP32-C3 build:

- power the sensor from the board's 5V supply if required by the sensor
- share ground between the sensor and ESP32-C3
- feed the pulse line to the ESP32-C3 GPIO with a **3.3V-safe pull-up**
- do not intentionally drive a raw 5V logic-high into an ESP32-C3 GPIO

The current ESPHome config enables the internal pull-up on the pulse pins, which is a good match if your specific sensor behaves like an open-collector / NPN output. If your exact sensor module actively drives 5V on the signal line, add level shifting or a resistor-divider approach before connecting it to the ESP32-C3.

## Home Assistant and InfluxDB

`home_assistant_ro_meter_package.yaml` creates:

- daily product and waste utility meters
- daily waste ratio
- lifetime waste ratio
- recovery percent
- 7-day mean waste ratio
- a maintenance log button and timestamp
- an example notification when the rolling ratio gets too high

If Home Assistant already exports to InfluxDB, these entities will become available for long-term dashboards in InfluxDB or Grafana automatically once the package is loaded and recorder/export includes them.

## Next steps

1. Make sure `wifi_ssid` and `wifi_password` exist in `secrets.yaml`, then verify the board pin substitutions in `ro_meter.yaml`.
2. Verify the OLED model/address and sensor GPIOs.
3. Flash the node with ESPHome.
4. Load `home_assistant_ro_meter_package.yaml` into Home Assistant packages.
5. Calibrate both sensors with real measured water volumes.
