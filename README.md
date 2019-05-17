# hass-light-state
Add light state awareness to Home Assistant for "Assumed State" light switches such as LightwaveRF 

The goal of this project was to devise a method of showing at least the correct on/off status of LightwaveRF Gen 1switches when they are controlled manually. The same method should work for many other kinds of lights switches and mains sockets classed as Assumed State.

It's been a constant irritation since I started using Domoticz before Home Assistant that my home automation system was unaware if I turned lights on and off at the wall.  Putting aside the fact that I'd spent a substantial amount of money on LightwaveRF switches, I couldn't find another option I was happy enough to swap them out for:

- Smart bulbs don't pass the partner approval test. "Normal" switches are a must.
- Options for my no-neutral-at-the-switch UK lighting system are extremely limited, and most of the few  options don't support dimming.
- LightwaveRF has only just published an API for their Gen 2 two-way, presumably Cloud Polling switches after promising it for two years, and they're shockingly expensive.
- For all their shortcomings, LightwaveRF Gen 1 switches still look great. 

After mulling over lots of options to add state awareness for the switches and thinking that the hardware was going to be the hard part, it finally dawned on me that this was something I could do with some cheap Sonoffs.

The hard part proved to be the learning curve to get to grips with states, templates, and the REST API in Home Assistant. The solution is working flawlessly so far.

For the project, I've used:

- Home Assistant
- Sonoff Basics, Sonofff POWs and Shelly 1s - You'll need one of any of them per switch. Some smart bulbs also work. See the end of this read me for more details.
- configuration.yaml edits under homeassistant, light, automation and rest_command. No custom components, Python, automation packages or shell scripts are required.
- @OttoWinter 's excellent ESPHome  firmware and Hassio add on - Neither are necessary, but they'll make your life easier.
- MQTT broker

and, of course, the original switches and the 433MHz RF transceiver that sends commands to them.

Here's a short video showing the solution in action: https://goo.gl/3iW15H

### Background

Home Assistant classifies IoT devices according to its state awareness and method https://www.home-assistant.io/blog/2016/02/12/classifying-the-internet-of-things/. 

| Classifier    | Description                                                  |
| ------------- | ------------------------------------------------------------ |
| Assumed State | We are unable to get the state of the device. Best we can do is to assume the state based on our last command. |
| Cloud Polling | Integration of this device happens via the cloud and   requires an active internet connection. Polling the state means that an   update might be noticed later. |
| Cloud Push    | Integration of this device happens via the cloud and   requires an active internet connection. Home Assistant will be notified as   soon as a new state is available. |
| Local Polling | Offers direct communication with device. Polling the state   means that an update might be noticed later. |
| Local Push    | Offers direct communication with device. Home Assistant   will be notified as soon as a new state is available. |

Devices like LightwaveRF Gen 1 light switches are in the Assumed State category. Home Assistant can issue commands to turn on, set the brightness, etc. But, it is unaware of any other state changes, such as controlling the lights at the wall switch.

This project moves LightwaveRF Gen 1 and similar switches from the Assumed State to the Local Push category with a few caveats:

- HA isn't actually getting state updates directly with the switches. It gets them from another device, such as a Sonoff Basic, that mirrors the power state of the lights. 
- There is a lag of some seconds before the Home Assistant shows the updated status due to boot time of the Sonoff at power on and then the MQTT keep-alive time at power off.  
- Manually-set brightness level is not known, but it may be possible to approximate in a later variant of the project. (See below.)

##### Installation & operation

The key enabler is a Sonoff Basic, Sonoff POW or Shelly 1 connected to the lighting ring. When you turn on the lights at the wall switch, the Sonoff boots and sends an MQTT birth message which triggers an automation in Home Assistant. The automation in turn calls a rest_command to change the state of the light entity in Home Assistant to on, for example, without actually calling the light.turn_on service.

It is possible to do a quick and dirty implementation which simply calls light.turn_on, for example, but that would cause problems when brightness state is added later. Plus, all of my painful trial and error with rest_commands and templates would be for nothing. ;-)

The hardware set up is simple. I show a Shelly 1 here, but the connections are the same for a Sonoff Basic.

![Installation](https://github.com/ronschaeffer/hass-light-state/blob/master/Installation.png))

In my tests, the Shelly 1s and Sonoffs stayed powered on down to 5% brightness.

When you manually turn off the switch, the Sonoff powers off and disappears from the network, prompting the MQTT broker to send its last will message. That triggers another automation and rest_command pair to set the light status to off.

![Operation](https://github.com/ronschaeffer/hass-light-state/blob/master/Operation.png)

The automations also run when you turn the light on or off through Home Assistant. They're set up to keep the same state attributes, though, so they're invisible to the user.

The MQTT messages are retained, so Home Assistant will show the correct state even after a restart.

### Obligatory mains electricity = danger notice

Do not electrocute yourself or burn your house down. Be sensible.

### Home Assistant configuration examples

I use a split configuration. This project required additions to following files:

##### configuration.yaml

```yaml
# Excerpt from homeassistant
homeassistant:
  customize_glob: !include customize_glob.yaml

# Excerpt of split configuration
automation: !include automations.yaml
light: !include lights.yaml
rest_command: !include rest_command.yaml
```

##### automations.yaml

```yaml
# Set light status to on and keep HA known brightness level if >= 1; Show as full brightness if not; Set other attributes

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
         endpoint: "{{ (trigger.topic.split('/')[1]) }}"
         friendly_name: "{{ state_attr('light.'+trigger.topic.split('/')[1], 'friendly_name') }}"
         supported_features: "{{ (state_attr('light.'+trigger.topic.split('/')[1], 'supported_features'))|int }}"
         assumed_state: false
         brightness: >
           {% if (state_attr('light.'+trigger.topic.split('/')[1], 'brightness'))|int >= 1 %}
             {{ (state_attr('light.'+trigger.topic.split('/')[1], 'brightness'))|int }}
           {% else %}
             255
           {% endif %}

# Set light status update to off without setting brightness
# Also keep current attribute values for friendly_name, supported_features & assumed_state
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
         endpoint: "{{ (trigger.topic.split('/')[1]) }}"
         friendly_name: "{{ state_attr('light.'+trigger.topic.split('/')[1], 'friendly_name') }}"
         supported_features: "{{ (state_attr('light.'+trigger.topic.split('/')[1], 'supported_features'))|int }}"
         assumed_state: false
```

##### customize_glob.yaml

```yaml
# Set lights to assumed_state false to force frontend to show toggles instead of lightning bolts
"light.*":
  assumed_state: false
```

##### lights.yaml

```yaml
# Set up LightwaveRF light switches as lights under the rfxtrx component and name them; Obviously, this would be different if you use a different controller or switches
- platform: rfxtrx
  automatic_add: false
  signal_repetitions: 3
  devices:
    0a14001605668002010080:
      name: Living Room
    0a14001905668003010080:
      name: Living Room Lamp
    0a14001b05668401010080:
      name: Dining Room
    0a14001e05668402010080:
      name: Dining Room Pendants
    0a14000505668101010080:
      name: Garden
      
# etc., etc.
```

##### rest_command.yaml

```yaml
# Update light state to on and set other attributes
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

# Update light state to off and set other attributes
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

### ESPHome device configuration example

Here's an example configuration for a Sonoff Basic. The key here is that your MQTT birth and LWT messages follow a consistent structure that can be processed by a templates in HA. Look in automations.yaml to see how that happens. 

```yaml
# Basic config
esphome:
  name: light_status_study
  platform: ESP8266
  board: esp01_1m

# Enable WiFi
wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_pwd

# Enable ESPHome native API; Optional; Home Assistant doesn't actually have to be aware of the Sonoff itself; HA just needs to receive MQTT messages from it
api:

# Enable OTA firmware update
ota:

# Enable MQTT; Choose your topics and payload carefully
mqtt:
  broker: !secret mqtt_broker
  port: !secret mqtt_port
  client_id: ls_study
  username: !secret mqtt_username
  password: !secret mqtt_password
  discovery: false
  birth_message:
    topic: 'light_status/study/availability'
    payload: 'online'
  will_message:
    topic: 'light_status/study/availability'
    payload: 'offline'
  keepalive: 5s
```

### Software, switches, controllers & components

I used the following in my testing. I've also include some comments on alternatives.

##### Software

- Hassio 0.86, 0.87, 0.88 and 0.89 in a Docker container on Ubuntu Server 18.04
- (Optional) ESPHome Hassio add-on (see below) https://esphome.io/guides/getting_started_hassio.html
- Mosquitto MQTT broker running in a separate Docker container. But, the Hassio MQTT add-on or any other broker shoudl also work.

##### Light switches and controllers

- LightwaveRF Gen 1 light switches https://lightwaverf.com/collections/all/series-connect, many of which are 50% off on clearance now.
- I see no reason why other light switches and mains sockets which are one-way only (assumed_state) paired with other assumed state hubs/controllers would not work in the same way. 
- RFXtrx433E controller, which appears to have been replaced by the RFXtrxXL with more memory http://www.rfxcom.com/epages/78165469.sf/en_GB/?ObjectPath=/Shops/78165469/Categories/Transceivers. 
- Also tested with a LightwaveRF Wi link hub (cloud-connected) which I no longer use otherwise. It also works fine for this project. Really, whatever method you're already using to send commands to your switches should be fine.

##### WiFi relays

- Sonoff Basics with ESPHome 1.11.2 firmware

- Started out with a Shelly 1 with stock firmware, which supports MQTT and can be made to work out of the box. But, ESPHome offers a lot more MQTT flexibility. Shelly 1s are very small compared to Sonoff Basics but more expensive. I will be flashing them with ESPHome at some point in order to use in places where space is tight.
- Also tested Sonoff Basics with Tasmota v6.4.1, but they too unstable. They kept disappearing from the network. Also, ESPHome lets you set the MQTT keep_alive time while Tasmota does not.
- Also tested a Sonoff POW with ESPHome and Tasmota. On/off status operation is the same as with the Sonoff Basic, and these may be a means of approximating brightness state.

### Brightness state (future)

It looks possible to approximate the current brightness by looking at the realtime voltage as measured with a device like the Sonoff POW or a Shelly 2.5. In my tests with halogen GU10s, it looks like voltage varies enough across the range of brightness that it can be used as a proxy for the current brightness level.

I connected one of the GU10s to the output terminals on the Sonoff POW instead of connecting it directly to the lighting ring, and measured the following.

| Dimming tests with halogen GU10s |                      |
| -------------------------------- | -------------------- |
| **Brightness %**                 | **Measured voltage** |
| 100                              | 220                  |
| 90                               | 205                  |
| 80                               | 192                  |
| 70                               | 177                  |
| 60                               | 158                  |
| 50                               | 132                  |
| 40                               | 110                  |
| 30                               | 94                   |
| 20                               | 77                   |
| 10                               | 56                   |

My idea is to set a range for each 10% change in brightness. A voltage measurement in a given range sent via MQTT would trigger an automation to update the indicated brightness state accordingly using a a rest_command. I'll play around with this when I have more time.

Tests with LED GU10s look less promising so far. Overall, the voltage range was smaller, and voltage plateaued at about 50% brightness. I need to give this more thought.

### Possible next steps

1. Include indication of approximate brightness when using Sonoff POWs or Shelly 1s.

2. Refine the HA config. I'd really like to get to one automation and one rest_command to handle both on and off. I couldn't figure out how to do that without setting brightness to 0 when off instead of the absence of a value for brightness as appears in dev-state when HA controls the light switches directly.

3. See if using ESPHome's new native API is a good alternative to MQTT.

### Update:  Smart bulbs

This method also works with some smart ESP8266 light bulbs such as [Lohas MR16](https://www.amazon.co.uk/gp/product/B07B49MRGH/ref=ppx_yo_dt_b_search_asin_title?ie=UTF8&psc=1). In that case, you don't need a separate Sonoff/Shelly.

Caveats:
- You can't adjust the brightness of smart bulbs with the LightwaveRF switches. It's on/off only.
- I've tried a few other ESP8266 bulbs and socket adapters such as [Lohas bulb](https://www.amazon.co.uk/gp/product/B07LBPSR83/ref=ppx_yo_dt_b_search_asin_title?ie=UTF8&psc=1), [Teckin bulb](https://www.amazon.co.uk/gp/product/B07KYFGB3M/ref=ppx_yo_dt_b_search_asin_title?ie=UTF8&psc=1) and [Sonoff bulb socket](https://www.amazon.co.uk/gp/product/B071FQCD6B/ref=ppx_yo_dt_b_search_asin_title?ie=UTF8&psc=1). As far as I can tell, they can't be made to play well with LightwaveRF switches. All three flickered or occasionally switched on and off.

An example configuration for a Lohas MR16 bulb connected to the LightawaveRF switch named ```light.entry``` follows. Note that if you have more than one bulb associated to a switch, you should only enable MQTT on one of the bulbs.

```
# Substitutions
substitutions:
  devicename: light_entry_bulb_1
  staticip: 192.168.xxx.xxx
  mqttclientid: ls_entry
  birthtopic: light_status/entry/availability
  lwttopic: light_status/entry/availability
  lightname: Entry Bulb 1

# Basic config
esphome:
  name: $devicename
  platform: ESP8266
  esp8266_restore_from_flash: true
  board: esp01_1m
  on_boot:
    then:
      - light.turn_on:
          id: light
          brightness: 100%
          red: 100%
          green: 85%
          blue: 42%
          white: 75%
          transition_length: 2s

# Enable WiFi
wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_pwd
  manual_ip:
    static_ip: $staticip
    gateway: 192.168.86.1
    subnet: 255.255.255.0

# Enable logging
logger:

# Enable ESPHome API
api:

# Enable OTA firmware update
ota:

# MQTT
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

my9231:
 data_pin: GPIO13  
 clock_pin: GPIO15  
 num_channels: 6
 num_chips: 2 

output:
  - platform: my9231
    id: output_white
    channel: 0
  - platform: my9231
    id: output_blue
    channel: 1
  - platform: my9231
    id: output_red
    channel: 3
  - platform: my9231
    id: output_green
    channel: 2

light:
  - platform: rgbw
    name: $lightname
    id: light
    default_transition_length: 0s
    red: output_red
    green: output_green
    blue: output_blue
    white: output_white
    ```

