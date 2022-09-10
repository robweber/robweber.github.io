---
layout: post
title:  "Fun With Light Sensors"
author: robweber
categories: automation smarthome
tags: home-assistant esphome yaml
---

In our living room we have several very large windows that can let in a lot of light. We also have some lights with either LIFX bulbs or Z-Wave switches that we can control via Home Assistant. It seemed like a no-brainer to try and make a system to auto turn the lights on or off depending on the amount of light in the room. On overcast days, or during the winter when it isn't light until 7:30am, we could have the lights turn on when we wanted them to and off again when the sun provided enough light in the room. With this very simple idea in mind I jumped down the rabbit hole of complications as our family started to run in to all the edge cases.

For reference, here is a basic diagram of our living room. There are 5 windows around the edge and 3 lights which labeled _Couch Lights_, _Reading Light_ and _Bookshelf Lights_. In this project I'll be controlling all of them, but under different circumstances and at different times.

The heavy lifting on the automation side is being done with ESPHome, light sensor compatible with ESPHome, and Home Assistant.

![Living Room](/images/2022-04/living_room.png)

<!--more-->

{% include toc %}

## Hardware Setup

For this I'm using a NodeMCU board and ESPHome to get the information from the light sensor into Home Assistant. The light sensor is a [BH1750 sensor][light-sensor-amazon] which is compatible with ESPHome out of the box. It works by calculating the ambient light in the room, which is returned in [Lux (lx)](https://en.wikipedia.org/wiki/Lux). Lux is a derived unit of illuminance equal to one lumen per square meter. The higher the lux value, the more light spread out over the room.

If you're not familiar with ESPHome, I'm not going to dive into all the details here. It can run on it's own but primarily is used as an addon for Home Assistant which allows you to easily program ESP32 and ESP8266 devices and allow them to integrate with Home Assistant. I'll post the code snippets for this project here but to get a full blown walk through of how to setup ESPHome and get a device working you can checkout [their tutorials](https://esphome.io/guides/getting_started_hassio.html).

### Wiring

Wiring up the sensor was pretty easy. It has 4 pins, ground, power, SDA and SCL. The SDA and SCL values are used in ESPHome to setup the I2C bus which allows communication with the chip on the sensor.  I ran SDA to pin 14 and SCL to pin 2.

### ESPHome Code

Once the wiring is done the device needed to be configured within ESPHome. This is done in YAML. Besides all the normal stuff needed to get it talking to the ESPHome system (and Home Assistant) I needed to define the I2C bus and the sensor itself. The code below is from my setup. If duplicating be sure to check the pin numbers on your device.

```
# specify pins for the light sensor
# source: https://esphome.io/components/i2c.html#i2c
i2c:
  sda: 14
  scl: 2
  scan: True
  id: bus_a

# create the sensor for HA
# source: https://esphome.io/components/sensor/bh1750.html
sensor:
  - platform: bh1750
    name: "Living Room Illuminance"
    address: 0x23
    update_interval: 60s
    force_update: True  # I actually added this later - will be important below
```

Once the firmware was flashed on the device it started reporting back to ESPHome and Home Assistant. I've noticed that for Home Assistant I sometimes need to go into the devices in Home Assistant and add the new ESP device in order to see it there. Once everything was talking I could see the lux value in Home Assistant. I have the update interval above set to once every minute. I figured this was quick enough to respond to light changes in the area but not so much I'm spamming HA with a lot of updates.

## Path To Automation

With the lux value in HA the next step was to come up with some automations. My ruleset was pretty simple, or so I thought. Once up and running things got complicated quickly. For positioning the device in the room I decided on a shelf near the largest windows in the room (designated with a red x on the diagram above). This would gather the most light but was far enough away from the lamps I was attempting to control that they wouldn't influence the lux values too much. I didn't want to create a feedback loop where the lights created too much influence, which in turn triggered them to turn off again.

### First Attempt

The first attempt was pretty basic. I watched the lux value change over a period of a few days before doing anything. As you'd expect, when the room is dark it's basically 0. On overcast days the value would waver between 3 and 30 depending on the cloud cover. On sunny days values would go well over 50. Of course a lot of this was dependent on the time of day and position of the sun. In the end I decided that __5lx__ was my threshold for if the lights should turn on. This would cover days when it was dark prior to sunrise and I wanted lights on that would turn off as the sun came up, and very overcast days where we just didn't get enough sunlight all day.

![Graph](/images/2022-04/week_graph.png)

The first automation for this was very simple. I don't have the YAML anymore but the flow was something like:

1. If lux value is less than 5 for 1 minute and lights are off, turn them on.
2. If lux value is greater than 5 for 1 minute and lights are on, turn them off.

Pretty simple right? This definitely worked but had a lot of real world problems. We ended up with lights that were on when we didn't want them on, lights that turned off when we did want them on, and lights that turned on/off erratically. In short, this did not work as expected.

### Defining Edge Cases

Some of the edge cases in this situation are obvious and others took time to really think through. In the end the list of issues fell into one of the below categories.

#### Non-Sensor Light Control

How to handle control of the lights that aren't triggered by the sensor? There are times that I want to turn the lights on manually, or as part of another automation, and just have them stay on. For this to work the automation needs to know if it is in control of the lights or not.

#### When It's Dark

Just because it's dark outside doesn't mean I want the lights on all night. It's part of the manual control problem but specific to night time. We have other light automations that handle evening/night routines and wanted the room to stay dark if the lights are turned off on purpose during those times.

#### Shades Are Closed

When the shades are closed this distorts the lux value as the light is now filtered through the shades. Logically thinking if the shades are closed we obviously don't care about automating the lighting in this room; the whole point is to supplement light when the sun isn't bright enough. We're either not in that room, maybe sleeping late, or have gone to bed early. Opening the shades is generally something we do right away on week days but may not touch them at all on weekends.

#### Erratic Behavior

This one is the most annoying. On partly cloudy days the light level in the room can bounce between two values suddenly. It will often go from 3lx to 30lx or 4lx to 50lx as the clouds shift over the sun. This can cause situations where the lights are constantly turning on/off over a period of time. This needed to be smarter and figure out more of a average for if it was lighter or darker outside.

### Solving Edge Cases

I was able to come up with solutions to all of these edge cases in a way that worked for us. Ultimately what we found was that the main conditions we had to work around were the time of day and position of the window shades. These two variables generally determined how the lights should respond.

The shades were a fairly easy problem to solve since they were already automated via Z-wave integration. By playing around a bit I was able to figure out that if they were open less than 25% the shades obstructed the sensor and I should just ignore the lux output value. This was easy to fit into an HA Automation. Adding a condition to check if the shades were open more than 25% before continuing solved this problem entirely. On days where we didn't open the shades the lights would just remain however we manually set them.

The time of day seems like it would have been easy to solve but took some additional thinking. Consider the following situations. The first is as the sun is setting. This results in the light level dropping naturally over a period of time, which will eventually trigger the lights at a certain point. We have a separate automation that is triggered prior to sunset that turns on some "mood lighting", which includes _only_ the Bookshelf Lights in the living room. Without modification we'd end up with a situation where all the lights are turned on as the sun goes down each day. Alternatively we may decide we want the lights on manually for whatever reason so we don't want the Evening Automation to just turn off lights assuming they shouldn't be on. The solution to this was a combination of using the sunset time trigger and a custom boolean Helper.

As part of the trigger conditions I added a condition to check if sunset was less than 45 min away. If it is, then we ignore the lux value trigger. The thought process here is that on a total overcast day the lower lux threshold will trigger the lights prior to this time period. If it's a sunny day the loss of light this close to sunset is normal and I can just ignore it. The Evening Automation will turn on mood lights at 15 min before sunset anyway so I'm only playing with a 30 minute time window here. It allows the room to lose light naturally as part of the flow of the day. The second fix I implemented was to automatically set a boolean helper to `True` when the light automation turns on the lights and set it back to `False` when it turns off the lights. The Evening Automation checks if this helper is `True` as a conditional. If `True` it means that light automation has triggered the lights and they should be turned off at dusk. If it is `False` the lights are all left alone as they may have manually triggered by someone in the house.

These fixes all handled the various time of day and manual use conditions we had. The final piece of the puzzle was the erratic on/off behavior that happened on partly cloudy days. The sun coming and going out of cloud cover constantly changed the lux level and triggered the lights in an undesirable pattern. To solve this problem I turned to the [Statistics sensor](https://www.home-assistant.io/integrations/statistics/) of Home Assistant. This is a very powerful sensor with a lot of features. I decided to use it to calculate the mean value from the lux ESP sensor over a 15 minute period. This smooths out the highs and lows from the sensor and results in a more even value. On overcast and sunny days the average is basically the sensor reading; but on partly-cloudy days this value trends up and down slowly based on how cloudy or sunny it is outside. While you can still get some erratic behavior when the light value hovers close to 5lx, overall this is much better with triggers that aren't super close together. To get this to work properly I did have to add `force_update: True` to the ESPHome config above. This forces the value to send every minute, even if it hasn't changed. This was necessary for the Statistics sensor to work properly when the value stays the same.

#### Automation Code

The automation code I ended up with is below. It is split between a Script and an Automation to keep the logic easier to read.

The Script is where the logic of if to turn the lights on or off is kept. It's a simple condition that checks the current light level, based on the Statistic sensor, and if the lights are already on or off. This in turn triggers if the state of the lights should be changed. The boolean helper is turned on or off here as well as appropriate.

```
sequence:
  - choose:
      - conditions:
          - condition: and
            conditions:
              - condition: numeric_state
                entity_id: sensor.living_room_illuminance
                above: '5'
              - condition: state
                entity_id: light.couch_lamp
                state: 'on'
        sequence:
          - service: light.turn_off
            target:
              entity_id: light.couch_lamp
            data: {}
          - service: switch.turn_off
            target:
              entity_id: switch.bookshelf_lights_switch
            data: {}
          - service: input_boolean.turn_off
            target:
              entity_id: input_boolean.living_room_light_automation
            data: {}
      - conditions:
          - condition: and
            conditions:
              - condition: numeric_state
                entity_id: sensor.living_room_illuminance
                below: '5'
              - condition: state
                entity_id: light.couch_lamp
                state: 'off'
        sequence:
          - service: light.turn_on
            target:
              entity_id: light.couch_lamp
            data: {}
          - service: switch.turn_on
            target:
              entity_id: switch.bookshelf_lights_switch
            data: {}
          - service: input_boolean.turn_on
            target:
              entity_id: input_boolean.living_room_light_automation
            data: {}
    default: []
mode: single
alias: Automate Living Room Lights

```

The Automation is where the triggers are evaluated. The main trigger condition is when the lux value goes above or below 5lx, based on the Statistic sensor. You'll notice there is one additional trigger I haven't mentioned, when the shade percentage goes above 25%. This is a catch for when the shades are moved from open to close, it will force the light level to evaluate at this moment. The conditions check for shades being open and the sunset condition explained above.

```
alias: Automate Living Room Lights
description: >-
  turn on/off lights in the living room if the shades are open and the light
  level is too low
trigger:
  - platform: numeric_state
    entity_id: sensor.living_room_illuminance
    below: '5'
    for:
      hours: 0
      minutes: 1
      seconds: 0
      milliseconds: 0
  - platform: numeric_state
    entity_id: sensor.living_room_illuminance
    above: '5'
    for:
      hours: 0
      minutes: 1
      seconds: 0
      milliseconds: 0
  - platform: numeric_state
    entity_id: cover.living_room_shades
    attribute: current_position
    above: '25'
    for:
      hours: 0
      minutes: 0
      seconds: 30
      milliseconds: 0
condition:
  - condition: and
    conditions:
      - condition: state
        entity_id: cover.living_room_shades
        state: open
      - condition: sun
        before: sunset
        before_offset: '-00:45:00'
action:
  - service: script.automate_living_room_lights
    data: {}
mode: single
```

One thing not listed here is the Evening Automation that triggers close to sunset. This resets the boolean helper each day to `False` so the lights start out in manual control each day.

## Conclusion

Overall this has worked very well for our family with very few instances where we get undesirable behavior. I purposly didn't seek out other solutions to this as I wanted to come up with things myself. I'm sure there are probably more elegant solutions. In the end this was easily explainable to my family (always a big plus) who often get frustrated when they don't understand why something happened on it's own. Acceptance factor is always a high priority when automating things for me.

## Links

* [Home Assistant Statistics Sensor][statistics-sensor]
* BH1750 Light Sensor
  * [Amazon Link][light-sensor-amazon]
  * [ESP Home Setup][esp-home-light-sensor]

[statistics-sensor]: https://www.home-assistant.io/integrations/statistics/
[esp-home-light-sensor]: https://esphome.io/components/sensor/bh1750.html?highlight=light%20sensor
[light-sensor-amazon]: https://www.amazon.com/gp/product/B01DLG4NZC/
