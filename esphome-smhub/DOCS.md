# ESPHome for SMHUB

This add-on allows you to compile and deploy ESPHome RTOS firmware natively on your SMLIGHT SMHUB SG2000 hardware.

## Installation

1. In your Home Assistant UI, navigate to **Settings** > **Add-ons** > **Add-on Store**.
2. Click the three vertical dots in the top-right corner and select **Repositories**.
3. Add the URL of this repository: `https://github.com/smlight-smhub/smhub-addons`
4. Search for **ESPHome for SMHUB** and click **Install**.
5. Once installed, start the add-on and toggle **Show in sidebar**.

## Configuration

Upon launching the add-on for the first time with an empty dashboard, it will automatically populate a default configuration file named `smhub-esphome.yaml` (pre-configured with a randomly generated secure API encryption key) for you to start from. 

If you need to configure additional nodes or recreate it, use the following template blueprint:

```yaml
esphome:
  name: rtos
  min_version: 2026.5.3
  project:
    name: "smlight.smhub"
    version: "1.0.0"

dashboard_import:
  package_import_url: github://smlight-smhub/smhub-addons/esphome-smhub/smhub-esphome.yaml@main

sg2000:
  board: SMHUB
  version: "2026.5.3"

logger:
  level: DEBUG
  baud_rate: 115200

api:
  reboot_timeout: 0s
  encryption:
    key: "YOUR_ENCRYPTION_KEY"
  services:
    - service: play_rtttl
      variables:
        song_str: string
      then:
        - rtttl.play:
            id: smhub_buzzer
            rtttl: !lambda 'return song_str;'

switch:
  - platform: gpio
    name: "Blue led"
    pin: "led_cus"
    restore_mode: RESTORE_DEFAULT_OFF

binary_sensor:
  - platform: gpio
    name: "User Button"
    pin: 
      number: "btn_2"
      mode:
        input: true
        pullup: true
    filters:
      - delayed_on: 10ms

light:
  - platform: sg2000_ws2812
    id: status_led
    name: "Status LED"
    spi_base: 0x04180000
    num_leds: 12
    effects:
      - pulse:
      - strobe:
      - random:
      - addressable_rainbow:
          name: "Rainbow Effect"
          speed: 10
      - addressable_color_wipe:
          name: "Color Wipe Effect"
      - addressable_scan:
          name: "Scan Effect"
          move_interval: 100ms
          scan_width: 1
      - addressable_fireworks:
          name: "Fireworks Effect"

rtttl:
  output: smhub_pwm
  id: smhub_buzzer

bluetooth_proxy:
  active: true

i2c:
  i2c_id: 4
  frequency: 100kHz

output:
 - platform: sg2000_pwm
   id: smhub_pwm
   pwm_id: 3
   channel: 2

time:
  - platform: smhub_time
    id: smlight_time

ota:
  - platform: esphome
    port: 3232
```

## How It Works

1. The custom dashboard compiles your configuration into C++ source files.
2. PlatformIO runs under the hood, building the binary against `framework-sg2000-rtos`.
3. When you click **Install** or **OTA**, the resulting `.elf` file is sent to the Linux host's `smhub-broker` daemon over the local API socket.
4. `smhub-broker` intercepts the OTA stream, replaces the active firmware on the eMMC, and restarts the RTOS core.
