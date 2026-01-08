---
title: Sonoff Mini R4 Extreme
date-published: 2023-12-01
type: relay
standard: global
board: esp32
made-for-esphome: False
difficulty: 4
---

![Sonoff MiniR4 Extreme](sonoff-mini-r4.jpg "Product Image")

Advertised as Smallest Wi-Fi Smart Switch Ever, with one relay, one external input, one button. Switch is really small,
approx 40x30mm, easily fits behind wall switch.

Product page:
[https://sonoff.tech/product/diy-smart-switches/minir4/](https://sonoff.tech/product/diy-smart-switches/minir4/)

## Features

Detected with esptool:

```bash
# esptool.py v4.6.2
# Detecting chip type... ESP32
# Chip is ESP32-D0WD-V3 (revision v3.0)
# Features: WiFi, BT, Dual Core, 240MHz, VRef calibration in efuse, Coding Scheme None
# Crystal is 40MHz
# Detected flash size: 4MB
```

## Programming

Firmware programming need disassembly (easy) and moderate soldering skills to attach USB-RS232 converter cables.
Pins are very easy to find as they are lebelled, but due to small size they are quite hard to solder.

![Sonoff MiniR4 Extreme](view_top.jpg "Top View")
![Sonoff MiniR4 Extreme](view_side.jpg "Top View")
![Sonoff MiniR4 Extreme](wires_angle.jpg "Top View")

Rx, Tx, Gnd available on ESP module,
Vcc (3V3) on main board, pad is covered with some protecting coating - needs to be scratched, very subtly, for solder to
cover it.

In order to enter programming mode need to hold button pressed while enabling power supply.
Bootloader uses 76800 baud rate, once application is started 115200 is used.

Programming can be done with esptool or directly through ESPHome (I'm using docker image)

## GPIO Pinout

| Pin    | Function                   |
|--------|----------------------------|
| GPIO00 | BUTTON                     |
| GPIO01 | likely TX (not tested)     |
| GPIO03 | likely RX (not tested)     |
| GPIO19 | blue LED                   |
| GPIO26 | Relay output               |
| GPIO27 | S2 (external switch input) |
| GND    | S1 (external switch input) |

## Basic Config with light

```yaml
substitutions:
  device_name: minir4-extreme

esphome:
  name: ${device_name}
  comment: "Sonoff MiniR4 Extreme"

esp32:
  variant: esp32
  framework:
    type: arduino

# Enable logging
logger:

# Enable Home Assistant API
api:
  password: ""

ota:
  password: ""

wifi:
  networks:
    - ssid: !secret wifi_ssid_1
      password: !secret wifi_password_1
    - ssid: !secret wifi_ssid_2
      password: !secret wifi_password_2

  # Enable fallback hotspot (captive portal) in case wifi connection fails
  ap:
    ssid: ${device_name} Fallback Hotspot
    password: ""

captive_portal:

web_server:
  port: 80

sensor:
  - platform: wifi_signal
    name: ${device_name} Wifi Signal Strength
    update_interval: 90s
    entity_category: "diagnostic"

  - platform: uptime
    name: ${device_name} Uptime
    update_interval: 300s
    entity_category: "diagnostic"

text_sensor:
  - platform: wifi_info
    ssid:
      name: Connected SSID
    ip_address:
      name: IP Address
    dns_address:
      name: DNS Address

#######################################
# Device specific Config Begins Below #
#######################################

status_led:
  pin:
    number: GPIO19
    inverted: true

output:
  # Physical relay on GPIO
  - platform: gpio
    pin: GPIO26
    id: relay_1

light:
  - platform: binary
    id: light_1
    name: ${device_name}
    icon: mdi:ceiling-light-multiple-outline
    restore_mode: restore_default_off
    output: relay_1

binary_sensor:
  - platform: gpio
    pin: GPIO00
    id: button
    filters:
      - invert:
      - delayed_off: 50ms
    on_press:
      - light.toggle:
          id: light_1

  - platform: gpio
    name: s1
    pin: GPIO27
    id: s1
    filters:
      - invert:
      - delayed_off: 50ms
    on_press:
      then:
        - light.turn_on:
            id: light_1
    on_release:
      then:
        - light.turn_off:
            id: light_1
```

## Improved config
  ## Configuration for input - flip switch / momentary button
  ## Configuration for relay - detach (input and relay can be used independently)

```yaml
substitutions:
  device_name: minir4
  friendly_name: SONOFF MINI R4
  update_interval: 600s

globals:
  # push button (true) or flip switch (false)
  # works if the relay is not disconnected, otherwise has no effect
  - id: input_as_button
    type: bool
    restore_value: yes
    initial_value: 'false'

  # detach relay
  - id: detach_relay
    type: bool
    restore_value: yes
    initial_value: 'false'

esphome:
  name: ${device_name}
  friendly_name: ${friendly_name}

esp32:
  board: esp32dev
  framework:
    type: arduino

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password

  ap:
    ssid:  ${device_name}
    password: !secret hotspot_password

captive_portal:

logger:

api:
  encryption:
    key: !secret api_encryption_key

ota:
  - platform: esphome
    password: !secret ota_password

time:
  - platform: homeassistant

web_server:
  port: 80
  version: 3

# Diagnostics
sensor:
  - platform: wifi_signal
    name: Wifi Signal Strength
    update_interval: ${update_interval}
    entity_category: "diagnostic"
    icon: mdi:signal-cellular-outline

  - platform: uptime
    name: Uptime
    update_interval: ${update_interval}
    entity_category: "diagnostic"
    icon: mdi:timer-outline

text_sensor:
  - platform: wifi_info
    ssid:
      name: Connected to
      icon: mdi:wifi
    dns_address:
      name: DNS Address
      icon: mdi:dns
    ip_address:
      name: IP Address
      icon: mdi:ip-network-outline

status_led:
  pin:
    number: GPIO19
    inverted: true

output:
  # Relay on GPIO
  - platform: gpio
    pin: GPIO26
    id: relay_1


light:
  - platform: binary
    id: light_1
    name: Relay
    entity_category: ""
    web_server:
      sorting_weight: 20
    icon: mdi:electric-switch
    output: relay_1

# Button on device
binary_sensor:
  - platform: gpio
    pin: GPIO00
    id: button
    filters:
      - invert:
      - delayed_off: 50ms
    on_press:
      - light.toggle:
          id: light_1

# S1 S2
  - platform: gpio
    name: Input
    pin: GPIO27
    entity_category: ""
    icon: mdi:import
    web_server:
      sorting_weight: 10
    id: s1
    filters:
      - invert:
      - delayed_off: 50ms 
    on_press:
      then:
        - if:
            # Relay affected only if not detached
            condition:
              lambda: 'return !id(detach_relay);'
            then:
              - if:
                  # Toggle if input as button
                  condition:
                    lambda: 'return id(input_as_button);'
                  then:
                    - light.toggle:
                        id: light_1
                  # Turning on if input as switch
                  else:
                    - light.turn_on:
                        id: light_1
    on_release:
      then:
        - if:
            # Turning off if not detached and input is not button
            condition:
              lambda: 'return !id(detach_relay) && !id(input_as_button);'
            then:
              - light.turn_off:
                  id: light_1

# Configuration switches
switch:
  - platform: template
    name: "Input as button"
    id: act_as_button_ui
    icon: mdi:gesture-tap-button
    entity_category: "config"
    web_server:
      sorting_weight: 30
    optimistic: true
    restore_mode: RESTORE_DEFAULT_OFF
    lambda: 'return id(input_as_button);'
    turn_on_action:
      - globals.set: {id: input_as_button, value: 'true'}
      # Force sync
      - lambda: 'global_preferences->sync();'
    turn_off_action:
      - globals.set: {id: input_as_button, value: 'false'}
      # Force sync
      - lambda: 'global_preferences->sync();'

  - platform: template
    name: "Detach relay"
    id: detach_relay_ui
    icon: mdi:power-plug-off-outline
    entity_category: "config"
    web_server:
      sorting_weight: 40
    optimistic: true
    restore_mode: RESTORE_DEFAULT_OFF
    lambda: 'return id(detach_relay);'
    turn_on_action:
      - globals.set: {id: detach_relay, value: 'true'}
      # Force sync
      - lambda: 'global_preferences->sync();'
    turn_off_action:
      - globals.set: {id: detach_relay, value: 'false'}
      # Force sync
      - lambda: 'global_preferences->sync();'
```
