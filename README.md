# hass-light-state
Add light state awareness to Home Assistant for "Assumed State" light switches such as LightwaveRF

![Home Assistant](https://img.shields.io/badge/Home%20Assistant-2024.1%2B-blue?logo=home-assistant)
![ESPHome](https://img.shields.io/badge/ESPHome-2024.1%2B-blue?logo=esphome)
![MQTT](https://img.shields.io/badge/MQTT-required-orange?logo=mqtt)
![Python Script](https://img.shields.io/badge/dependency-ps__hassio__entities-green?logo=python)
![License](https://img.shields.io/github/license/ronschaeffer/hass-light-state)

The goal of this project was to devise a method of showing at least the correct on/off status of LightwaveRF Gen 1 switches when they are controlled manually. The same method should work for many other kinds of light switches and mains sockets classed as Assumed State.

It's been a constant irritation since I started using Domoticz before Home Assistant that my home automation system was unaware if I turned lights on and off at the wall. Putting aside the fact that I'd spent a substantial amount of money on LightwaveRF switches, I couldn't find another option I was happy enough to swap them out for:

- Smart bulbs don't pass the partner approval test. "Normal" switches are a must.
- Options for my no-neutral-at-the-switch UK lighting system are extremely limited, and most of the few options don't support dimming.
- LightwaveRF Gen 1 switches still look great.

After mulling over lots of options to add state awareness for the switches, it finally dawned on me that this was something I could do with some cheap ESP devices.

Here's a short video showing the solution in action: https://goo.gl/3iW15H

---

## Background

Home Assistant classifies IoT devices according to their state awareness and method. See https://www.home-assistant.io/blog/2016/02/12/classifying-the-internet-of-things/

| Classifier    | Description |
| ------------- | ----------- |
| Assumed State | We are unable to get the state of the device. Best we can do is to assume the state based on our last command. |
| Local Push    | Offers direct communication with device. Home Assistant will be notified as soon as a new state is available. |

Devices like LightwaveRF Gen 1 light switches are in the Assumed State category. Home Assistant can issue commands to turn on, set brightness, etc., but is unaware of any state changes made at the wall switch.

This project effectively moves LightwaveRF Gen 1 and similar switches from Assumed State to Local Push, with a few caveats:

- HA isn't getting state updates directly from the switches. It gets them from a secondary device that mirrors the mains power state of the lighting circuit.
- There is a lag of some seconds at power-on (device boot time + WiFi connect + MQTT birth message) and at power-off (MQTT broker keepalive timeout before LWT is delivered).
- Manually-set brightness level is not known.

---

## How it works

The key enabler is any **mains-powered ESP8266 or ESP32 device running ESPHome** — a Sonoff Basic, Shelly 1, or similar — wired into the lighting ring **after** the LightwaveRF switch, so that it only receives mains power when the switch is on.

**Power on → light on:**
When the LightwaveRF switch is turned on (manually or via HA), mains power reaches the ESP device. It boots, connects to WiFi, connects to the MQTT broker, and sends its **birth message** (`online`) to its configured topic (`light_status/<n>/availability`). A HA automation receives this and sets the corresponding light entity state to `on`.

**Power off → light off:**
When the switch is turned off, mains power is cut. The ESP device drops off the network. The MQTT broker detects the lost connection (within the keepalive window, set to 5s) and delivers the pre-registered **Last Will and Testament (LWT)** message (`offline`) to the same topic. The HA automation receives this and sets the light entity state to `off`.

Both birth and LWT messages are **retained** by the broker, so HA will receive the correct state on restart without waiting for the next state change.

The device's own status LED is turned on at boot (`on_boot: light.turn_on`) purely as a visual indicator that it has power — it has no effect on the room lighting.

![Installation](https://github.com/ronschaeffer/hass-light-state/blob/master/Installation.png)

![Operation](https://github.com/ronschaeffer/hass-light-state/blob/master/Operation.png)

---

## Obligatory mains electricity = danger notice

Do not electrocute yourself or burn your house down. Be sensible.

---

## Method (ESPHome + python_script)

### Dependencies

- **ESPHome** — for firmware on the ESP devices
- **MQTT broker** — e.g. Mosquitto or EMQX
- **[pmazz/ps_hassio_entities](https://github.com/pmazz/ps_hassio_entities)** — install via HACS or copy `hass_entities.py` to your `/config/python_scripts/` directory. Enable `python_script:` in `configuration.yaml` if not already present.

### ESPHome device configuration

One ESPHome device per LightwaveRF switch. Any mains-powered ESP8266 or ESP32 board running ESPHome will work — Sonoff Basic, Shelly 1, or any equivalent device. The `endpoint` substitution must exactly match the `<n>` segment of the corresponding HA light entity ID (e.g. `endpoint: kitchen_main` → `light.kitchen_main`).

The example below is based on a Sonoff Basic (ESP8266, `esp01_1m` board, GPIO13 LED). Adjust `esp8266`/`esp32`, `board`, `led_pin`, and `output` platform to match your hardware.

```yaml
substitutions:
  # Identity
  domain: light_status
  domainshort: ls
  endpoint: kitchen_main              # Must match light.<endpoint> in HA
  esphome_node_name: light-status-kitchen-main
  friendly_name: Kitchen Main Status Light
  area: Kitchen

  # WiFi
  wifissid: !secret wifi_ssid
  wifipwd: !secret wifi_pwd
  appwd: !secret esphome_appassword

  # MQTT — topic format must match the HA automation trigger
  mqttclientid: ${domainshort}_${endpoint}
  birthtopic: ${domain}/${endpoint}/availability
  lwttopic: ${domain}/${endpoint}/availability

  # Pins — adjust for your hardware
  led_pin: GPIO13

  # IDs
  led_id: powerled

esphome:
  name: $esphome_node_name
  friendly_name: $friendly_name
  area: $area
  on_boot:
    - light.turn_on: $led_id   # Turns on the device's own status LED — not the room light

esp8266:
  board: esp01_1m

wifi:
  networks:
    - ssid: $wifissid
      password: $wifipwd
  fast_connect: true
  reboot_timeout: 15min
  ap:
    ssid: esphome_${domainshort}_${endpoint}
    password: $appwd

captive_portal:

logger:
  level: INFO

api:
  encryption:
    key: !secret api_encryption_key

ota:
  - platform: esphome
    password: !secret ota_password

mqtt:
  broker: !secret mqtt_broker
  port: !secret mqtt_port
  client_id: $mqttclientid
  username: !secret mqtt_username
  password: !secret mqtt_password
  discovery: false
  birth_message:
    topic: $birthtopic
    payload: 'online'
  will_message:
    topic: $lwttopic
    payload: 'offline'
  keepalive: 5s

button:
  - platform: restart
    name: "Restart"
    entity_category: config
  - platform: safe_mode
    name: "Safe Mode"
    entity_category: config

output:
  - platform: esp8266_pwm
    id: powerledoutput
    pin:
      number: $led_pin
      inverted: true

light:
  - platform: monochromatic
    output: powerledoutput
    id: $led_id
    name: "LED"

binary_sensor:
  - platform: status
    name: "Connectivity"

sensor:
  - platform: uptime
    name: "Uptime"
    update_interval: 60s
    entity_category: diagnostic
  - platform: wifi_signal
    name: "WiFi Signal"
    id: wifi_signal_db
    update_interval: 60s
    entity_category: diagnostic
  - platform: copy
    source_id: wifi_signal_db
    name: "WiFi Strength"
    filters:
      - lambda: return min(max(2 * (x + 100.0), 0.0), 100.0);
    unit_of_measurement: "%"
    device_class: ""
    entity_category: diagnostic

text_sensor:
  - platform: wifi_info
    ip_address:
      name: "IP Address"
    ssid:
      name: "SSID"
```

### Home Assistant automation

A single automation handles both `on` and `off` using the MQTT payload directly. The `+` wildcard matches all status light devices. `trigger.topic.split('/')[1]` extracts the endpoint name (e.g. `kitchen_main`) to construct the target entity ID (`light.kitchen_main`).

```yaml
alias: "Light State: Set on or off"
description: >
  Sets light entity state based on ESPHome status light device MQTT availability messages.
  Birth message (online) = switch turned on = light on.
  LWT message (offline) = switch turned off = light off.
  Uses pmazz/ps_hassio_entities python script to set state directly without REST API.
mode: parallel
max: 10
trigger:
  - platform: mqtt
    topic: light_status/+/availability
action:
  - action: python_script.hass_entities
    data:
      action: set_state
      entity_id: "light.{{ trigger.topic.split('/')[1] }}"
      state: "{{ 'on' if trigger.payload == 'online' else 'off' }}"
```

**Key points:**
- `mode: parallel` with `max: 10` allows multiple devices to be processed simultaneously — important on HA startup when retained messages replay for all devices at once.
- The state template uses a simple inline ternary. Avoid multi-line Jinja2 `if/elif` blocks here as they can introduce whitespace into the state string.
- MQTT birth/LWT messages are retained by ESPHome by default, so HA receives the correct state on restart via retained message replay.
- Do **not** rely on the ESPHome native API connection (the `_status_light_led` entities in HA) to determine whether a light is on or off. Those entities show `unavailable` when the device has no mains power — which is the correct and expected idle state when the switch is off. The authoritative source is always the MQTT retained message on `light_status/<n>/availability`.

### Verifying state via MQTT

To check the current retained state of all light status devices directly from the broker (EMQX example):

```bash
curl -s -u "<api_key>:<api_secret>" \
  "http://localhost:18083/api/v5/mqtt/retainer/messages?limit=100&topic=light_status%2F%23" \
  | python3 -c "
import json, sys, base64
msgs = json.load(sys.stdin).get('data', [])
for m in sorted(msgs, key=lambda x: x['topic']):
    p = base64.b64decode(m.get('payload', '')).decode() if m.get('payload') else ''
    print(f\"{m['topic']}: {p}\")
"
```

Each `light_status/<n>/availability` topic should show `online` when that switch is on, and `offline` when it is off.

---

## Legacy method (REST commands) — retained for continuity

The original implementation used HA's REST API with a long-lived access token and `rest_command` entries to set entity state. This approach worked but required maintaining an API token in `secrets.yaml` and two separate automations (one for on, one for off).

It has been superseded by the `python_script.hass_entities` approach above, which is simpler and has no external API dependency.

### secrets.yaml (legacy)

```yaml
# Long-lived access token — create via your HA profile page
api-token-with-bearer: 'Bearer <your_token_here>'
```

### automations.yaml (legacy)

```yaml
- alias: light status update to on
  initial_state: true
  trigger:
    platform: mqtt
    topic: 'light_status/+/availability'
  condition:
    condition: template
    value_template: "{{ trigger.payload|string == 'online' }}"
  action:
    - service: rest_command.light_status_update_on
      data_template:
        endpoint: "{{ trigger.topic.split('/')[1] }}"
        friendly_name: "{{ state_attr('light.'+trigger.topic.split('/')[1], 'friendly_name') }}"
        supported_features: "{{ state_attr('light.'+trigger.topic.split('/')[1], 'supported_features')|int }}"
        assumed_state: false
        brightness: >
          {% if state_attr('light.'+trigger.topic.split('/')[1], 'brightness')|int >= 1 %}
            {{ state_attr('light.'+trigger.topic.split('/')[1], 'brightness')|int }}
          {% else %}
            255
          {% endif %}

- alias: light status update to off
  initial_state: true
  trigger:
    platform: mqtt
    topic: 'light_status/+/availability'
  condition:
    condition: template
    value_template: "{{ trigger.payload|string == 'offline' }}"
  action:
    - service: rest_command.light_status_update_off
      data_template:
        endpoint: "{{ trigger.topic.split('/')[1] }}"
        friendly_name: "{{ state_attr('light.'+trigger.topic.split('/')[1], 'friendly_name') }}"
        supported_features: "{{ state_attr('light.'+trigger.topic.split('/')[1], 'supported_features')|int }}"
        assumed_state: false
```

### rest_command.yaml (legacy)

```yaml
light_status_update_on:
  url: 'http://localhost:8123/api/states/light.{{ endpoint }}'
  method: POST
  headers:
    authorization: !secret api-token-with-bearer
    content-type: 'application/json'
  payload: >-
    {
      "state": "on",
      "attributes": {
        "brightness": {{ brightness|int }},
        "friendly_name": "{{ friendly_name }}",
        "assumed_state": "{{ assumed_state }}",
        "supported_features": {{ supported_features }}
      }
    }

light_status_update_off:
  url: 'http://localhost:8123/api/states/light.{{ endpoint }}'
  method: POST
  headers:
    authorization: !secret api-token-with-bearer
    content-type: 'application/json'
  payload: >-
    {
      "state": "off",
      "attributes": {
        "friendly_name": "{{ friendly_name }}",
        "assumed_state": "{{ assumed_state }}",
        "supported_features": {{ supported_features }}
      }
    }
```

---

## Hardware

### ESP devices

Any mains-powered ESP8266 or ESP32 device running ESPHome that can be wired into the lighting ring will work. Tested devices include:

- **Sonoff Basic** — inexpensive, reliable. Stays powered down to ~5% brightness on dimmer switches.
- **Shelly 1** — smaller form factor, works with stock firmware or ESPHome.
- **Sonoff POW** — same on/off operation, with the addition of power monitoring which could potentially be used to approximate brightness (see below).

Tasmota was also tested but found to be less stable — devices would drop from the network unpredictably. ESPHome also allows setting the MQTT `keepalive` time, which Tasmota does not.

### Hardware installation

The ESP device is wired into the lighting ring after the LightwaveRF switch, so it only receives mains power when the switch is on.

![Installation](https://github.com/ronschaeffer/hass-light-state/blob/master/Installation.png)

### Light switches and controllers

- **LightwaveRF Gen 1** switches (one-way, no-neutral) — the primary use case for this project.
- Any other assumed-state light switch or mains socket should work the same way, regardless of the hub or controller used to send commands to the switches.

---

## Notes on the `_status_light_led` entities in HA

When ESPHome devices are discovered by HA via the native API, they appear as entities (e.g. `light.kitchen_main_status_light_led`). These reflect the device's own LED state and API connection status — **not** the room light state.

- When the switch is **off**, the device has no mains power → no WiFi → no API connection → entity shows `unavailable`. This is correct and expected.
- When the switch is **on**, the device boots and connects → entity shows `on` (the status LED is illuminated).

**Do not use these entities to infer room light state.** They are diagnostic only. The authoritative source is always the MQTT retained message on `light_status/<n>/availability`.

---

## Brightness state

It may be possible to approximate current brightness using a Sonoff POW or Shelly 2.5 by monitoring realtime voltage. In tests with halogen GU10s, voltage varied predictably across the dimming range:

| Brightness % | Measured voltage |
|---|---|
| 100 | 220V |
| 80 | 192V |
| 60 | 158V |
| 40 | 110V |
| 20 | 77V |

Tests with LED GU10s were less conclusive — the voltage range was smaller and plateaued at around 50% brightness. This has not been implemented.

---

## Smart bulbs (alternative approach)

This method also works with some smart ESP8266 bulbs (e.g. Lohas MR16) wired behind a LightwaveRF switch. The bulb itself acts as the status device — no separate ESP device is needed.

Caveats:
- Brightness cannot be adjusted via the LightwaveRF switch — on/off only.
- Not all smart bulbs are compatible. Tested incompatible: Lohas, Teckin, Sonoff bulb socket — all flickered or toggled unpredictably when used with LightwaveRF switches.
- If you have more than one bulb on a switch, only enable MQTT on one of them.
