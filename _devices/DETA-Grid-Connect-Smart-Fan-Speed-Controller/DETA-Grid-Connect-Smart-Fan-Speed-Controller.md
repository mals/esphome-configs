---
title: Deta Grid Connect Smart Fan Speed Controller with Touch Light Switch
date-published: 2020-10-16
type: switch
standard: au
---

1. TOC
{:toc}

## General Notes

The DETA [Smart Fan Speed Controller with Touch Light Switch (6914HA)](https://www.bunnings.com.au/deta-grid-connect-smart-fan-speed-controller-with-touch-light-switch_p0098815) are made by Arlec as part of the [Grid Connect ecosystem](https://grid-connect.com.au/), and are sold at Bunnings in Australia and New Zealand.  They can be flashed by using a serial programmer.

## GPIO Pinout

| Pin     | Function                           |
|---------|------------------------------------|
| GPIO0   | Button, Middle (Fan On/Off)        |
| GPIO3   | Status LED *(inverted)*            |
| GPIO4   | Relay 3, Bottom (Fan Speed, LED)   |
| GPIO5   | Button, Bottom (Fan Speed)         |
| GPIO13  | Relay 2, Middle *(includes LED)*   |
| GPIO14  | Relay 1, Top *(includes LED)*      |
| GPIO15  | Relay 4, Hidden (Fan Speed)        |
| GPIO16  | Button, Top (Light)                |

Note that each relay (1, 2, 3) shares a pin with its associated LED; it's not possible to turn either relay on/off independently of its button LED.
The layout designation here assumes that it is installed vertically, with the status LED (group of 6 dots) on the right-hand side.

The fan speed is controlled by relay 3 and relay 4 as described below:

| Fan Speed | Relay 3 | Relay 4 |
|-----------|---------|---------|
| LOW       |  Off    |  Off    |
| MED       |  Off    |  On     |
| HIGH      |  On     |  On     |

## Getting it up and running
```yaml
substitutions:
  device_name: esphome_fan_switch
  friendly_name: "ESPHome Fan Switch"
  
#################################

esphome:
  name: ${device_name}
  platform: ESP8266
  board: esp01_1m

wifi:
  ssid: "ssid"
  password: "WIFI_PASSWORD"

  # Enable fallback hotspot (captive portal) in case wifi connection fails
  ap:
    ssid: "ESPHome Fan Switch"
    password: "WIFI_FAILBACK_PASSWORD"

captive_portal:

# Enable logging
logger:

# Enable Home Assistant API
api:
  password: "API_PASSWORD"

ota:
  password: "OTA_PASSWORD"

#################################


status_led:
  pin:
    number: GPIO3
    inverted: True

output:
  # Top button
  - platform: gpio
    pin: GPIO14
    id: relay1

  # Middle (Fan On/Off) button
  - platform: gpio
    pin: GPIO13
    id: relay2
    
  # Bottom (Fan Speed) button
  - platform: gpio
    pin: GPIO04
    id: relay3
    
  # Hidden relay
  - platform: gpio
    pin: GPIO15
    id: relay4

  - platform: template
    id: custom_fan
    type: float
    write_action:
      - if:
          condition:
            lambda: return (( state == 0 ));
          then:
            - output.turn_off: relay2
            - output.turn_off: relay3
            - output.turn_off: relay4
      - if:
          condition:
            lambda: return (( state > 0 )) && (( state < 0.34 ));
          then:
            - output.turn_on: relay2
            - output.turn_off: relay3
            - output.turn_off: relay4
      - if:
          condition:
            lambda: return (( state >= 0.34 )) && (( state < 0.67 ));
          then:
            - output.turn_on: relay2
            - output.turn_off: relay3
            - output.turn_on: relay4
      - if:
          condition:
            lambda: return (( state >= 0.67 )) && (( state <= 1 ));
          then:
            - output.turn_on: relay2
            - output.turn_on: relay3
            - output.turn_on: relay4
            
light:
  # Top button
  - platform: binary
    name: "${friendly_name} Light"
    output: relay1
    id: light1

fan:
  - platform: speed
    name: "${friendly_name} Fan"
    output: custom_fan
    id: fan_1


# Buttons
binary_sensor:
  # Top button
  - platform: gpio
    pin:
      number: GPIO16
      mode: INPUT_PULLUP
      inverted: True
    name: "${friendly_name} Top Button"
    #toggle relay on push
    on_press:
      - light.toggle: light1

  # Middle button
  - platform: gpio
    pin:
      number: GPIO0
      mode: INPUT_PULLUP
      inverted: True
    name: "${friendly_name} Fan Button"
    #toggle relay on push
    on_press:
    - fan.toggle: fan_1
    
  # Bottom button
  - platform: gpio
    pin:
      number: GPIO5
      mode: INPUT_PULLUP
      inverted: True
    name: "${friendly_name} Fan Speed Button"
    on_press:
      then:
        - fan.turn_on:
            id: fan_1
            speed: !lambda |-
              if (id(fan_1).speed == FAN_SPEED_LOW) {
                return FAN_SPEED_MEDIUM;
              }
              else if (id(fan_1).speed == FAN_SPEED_MEDIUM) {
                return FAN_SPEED_HIGH;
              }
              else {
                return FAN_SPEED_LOW;
              };

switch:
  - platform: restart
    name: "${friendly_name} REBOOT"
    
```
