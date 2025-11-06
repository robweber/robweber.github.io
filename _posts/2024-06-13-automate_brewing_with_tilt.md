---
layout: post
title:  "Automate Brewing With Tilt"
author: robweber
categories: automation hardware smarthome
tags: home-assistant python
---

As I hobby I like to homebrew beer. Nothing too crazy but just a few cases a couple of times a year. It's fun to do and makes for a nice gift option for friends and family as well. Recently I starting thinking more about gathering data during the brewing process. Mostly I was interested in temperature as different stages of the brew process require specific temperature ranges. These vary depending on the type of beer but it's important so you don't kill the yeast during fermentation. In the past I've used a manual thermometer but wanted something more hands off.

I didn't want to build something myself as there is a great consumer market and DIY space for this type of stuff already. My only real requirement was that I get the information into [Home Assistant][home-assistant] since that is where I try and keep all my dashboards and data for ease of use. As I spiraled down the rabbit hole of possible options I ended up having to do some on-the-fly thinking to get it all working.

<!--more-->

{% include toc %}

## Hardware

There are lots of options for home brew off-the-shelf and DIY hardware. It's likely anyone involved in home brewing has heard of projects like the [BrewPi](https://www.brewpi.com/) for monitoring and control. I wasn't concerned with controlling temperature so much as monitoring it so ultimately what I needed was just a thermometer.

I ran across this cool little device called [Tilt][tilt]. It's a combination [hydrometer](https://en.wikipedia.org/wiki/Hydrometer) and thermometer that floats in your brew bucket (or carboy) so you can get readings during the process. I liked this idea right away since it's dead simple, affordable, and the hardware was pre-made - meaning I didn't have to go down the route of making something with an ESP32 or Raspberry Pi. To further sweeten the deal there is [Home Assistant integration](https://www.home-assistant.io/integrations/tilt_ble/) for it as well. It uses Bluetooth, which I am generally not a fan of for home automation, but in reading more about the device it uses [low energy Bluetooth packets](https://en.wikipedia.org/wiki/Bluetooth_Low_Energy) instead of pairing directly with something. This extends the battery life and makes it easier for another device to capture the data.

My general thoughts for getting an integration going were pretty simple:

1. Buy a [Tilt][tilt]
2. Utilize [ESPHome BLE proxies][esp-proxy] to gather information
3. Integrate with Home assistant
4. Make a Dashboard.

I thought I had it all figured out so I pulled the trigger on ordering one for my next batch.

## The First Attempt

Once the Tilt arrived in the mail I threw it in some water to test it out. Right away I ran into a problem. The [Tilt App](https://play.google.com/store/apps/details?id=com.baronbrew.tilthydrometerf7) wouldn't work on my Android phone. The Android OS on my device was too new for the app in the App Store. I grabbed my daughter's iPad and was happy to see the [iOS version](https://apps.apple.com/us/app/tilt-2/id1446662555) would install. The app loaded and found the device quickly. It's a pretty simple interface but got the job done.

![Tilt iPad App - from App Store](/images/2024-06/tilt_app.png)

Since I wanted to capture and store the data the iPad wouldn't work long term. I needed a device that was always listening for the broadcast packets. My first thought was to use an [ESPHome BLE proxy][esp-proxy]. This is a really awesome feature of ESPHome that allows compatible ESP32 hardware to act as a "local" Bluetooth device for Home Assistant integration purposes. In a nutshell instead of pairing a Bluetooth device to your phone or tablet you pair it to Home Assistant and the ESPHome proxy devices capture and send the data back to Home Assistant. The real power is you can have multiple instances of these around your home so if the device moves (a common thing with Bluetooth devices) a different proxy can find the device and get the data.

I have 6 ESP32 devices around my house so I figured one of them would support the Bluetooth Proxy component. I was wrong. In the end I found that I have only one device with an integrated Bluetooth radio. One was all I needed but then I hit another road block. The device in question is attached to a Neopixel ring. The ring lights up in different colors to notify of different events (garage door opening, door bell, etc). The Neopixel component of ESPHome is incompatible with the requirements needed to also run the Bluetooth Proxy component. I couldn't repurpose the device so I ran in to a dead end here.

## The Second Attempt

Realizing I had to re-think my process I started looking in to other options. Surely someone had tried to think up an answer to this type of issue before? As it turns out the people behind Tilt do offer their own solution known as a [TiltPi](https://tilthydrometer.com/products/tilt-pi-v2-bullseye-may23-raspberry-pi-sd-card-image-download). This is a local web service that uses the Bluetooth radio in a Raspberry Pi to collected the Tilt readings. I have some Raspberry Pi devices in service doing other things so thought I could layer this in.

While searching for the source code for the TiltPi I also stumbled upon [Pitch][tilt-pitch]. In the description of the project it states "_The Tilt hardware is impressive, but the mobile apps and TiltPi are confusing and buggy._". My experience with the Android app earlier confirmed this might be the case. Rather than going down the road of the TiltPi I shifted gears to giving this a try. The Pitch project had multiple integrations but one in particular caught my eye - [the webhook](https://github.com/linjmeyer/tilt-pitch#Webhook). This would send the Tilt readings, via an HTTP request, to any configured web URL. I refined my integration strategy from above:

1. ~~Buy a [Tilt][tilt]~~.
2. Install [Pitch][tilt-pitch] on a Pi W I had already
3. Configure Pitch to send a webhook to Home Assistant
4. Create a [Home Assistant Automation][ha-webhook] to capture the webhook and set some MQTT topics
5. Utilize an [MQTT Sensor][ha-mqtt] to expose the information in Home Assistant.
6. Make a Dashboard.

Definitely more involved than the first idea but this would work.

### Getting Pitch Working

For the most part Pitch ran exactly as specified in the [Github Repo][tilt-pitch] so I'll bypass that part of the setup. The only modification I had to make was to fix an error I had the first time running it. The project states it works with __Python 3.7 or greater__ but the required libraries don't have pinned versions. As a result a library called `pyfiglet` downloaded it's latest version, which didn't work on Python 3.8. My Rpi does some other things and I didn't want to mess with the base Python version so I instead pinned the library to a version known to work with Python 3.8. In the end I modified the requirements file to read `pyfiglet<1` to get a version that would work. __Note:__ this fix was later integrated to the main repo via a [pull request](https://github.com/linjmeyer/tilt-pitch/pull/38).

Included in the `examples` directory is a systemd file to run the whole thing as a service. The instructions aren't super clear but you have to modify it for paths on your system and then move it to the right folders on your system.

```
sudo mv pitch.service /etc/systemd/system
sudo systemctl enable pitch
sudo systemctl start pitch
```

I also modified the Pitch setup file a bit so that the webhooks don't fire as often. The Tilt is pretty chatty and will send updates basically once a second. This slows things down to every 30 seconds in Home Assistant. __Note the Home Assistant URL__ - this is setup as part of the automation below.

```
{
  "queue_empty_sleep_seconds": 2,
  "queue_size": 2,
  "green_name": "My Beer",
  "green_original_gravity": 1.050,
  "prometheus_enabled": false,
  "webhook_limit_period": 30,
  "webhook_urls": [
    "http://ip:8123/api/webhook/-C1XgmLYl0jCWyazXyV-8lNS9"
  ]
}
```

## Home Assistant Setup

### Setup The Webhook

Since Pitch is sending the data to Home Assistant it needs a web hook endpoint to capture the data. Home Assistant supports [webhook triggers][ha-webhook] for automations. The instructions are fairly clear; I simply selected webhook as the Trigger type. Home Assistant spits out a randomly generated endpoint ID. Clicking the copy button to get the full URL I just pasted this into the `pitch.json` file as shown above.

Tilt sends a JSON payload so for the rest of the Automation I figured the easiest thing to do was to retain it in an MQTT topic. This way I could set it up as an [MQTT Sensor][ha-mqtt]. To do this I used the `mqtt.publish` service. Note that I'm using the color of the Tilt to set the topic. Just a bit of future proofing as Pitch could get information from multiple Tilt devices, each with a different color. __Future Enhancement:__ Pitch integrates with a lot of services, if it could send to MQTT directly the webhook in Home Assistant could be cut out completely. I might tackle this later to reduce complexity.

Full Automation:

{% raw %}
```
alias: Brewing - Tilt
description: update tilt sensor information
trigger:
  - platform: webhook
    allowed_methods:
      - POST
    local_only: true
    webhook_id: "-XNHMrTWaErtLzUy-8lNS9"
condition: []
action:
  - service: mqtt.publish
    data_template:
      qos: "1"
      retain: true
      topic: tilt/{{ trigger.json.color }}
      payload: "{{ trigger.json | to_json }}"
mode: single
```
{% endraw %}

### Setup Sensors

Once the information was being sent to MQTT I needed a [few sensors][ha-mqtt] to be able to use everything in a dashboard. I decided to create a __gravity__ and __temperature__ sensor since those were the two main values collected. I dumped the rest of the values to the sensor attributes just in case I wanted them later.

{% raw %}
```
mqtt:
  sensor:
    - name: Tilt Green Temperature
      state_topic: "tilt/green"
      value_template: " {{ value_json.temp_fahrenheit }}"
      icon: mdi:thermometer
      device_class: temperature
      unit_of_measurement: "°F"
      json_attributes_topic: "tilt/green"
      json_attributes_template: "{{ value_json | tojson }}"
    - name: Tilt Green Gravity
      state_topic: "tilt/green"
      value_template: " {{ value_json.gravity }}"
      icon: mdi:gauge
      json_attributes_topic: "tilt/green"
      json_attributes_template: "{{ value_json | tojson }}"
```
{% endraw %}

### Dashboard

Finally I could get to the part I wanted - a dashboard. I decided I wanted a few graphs, utilizing the excellent [Mini Graph Card][graph-card] Lovelace plugin. I'm graphing both the Beer Gravity and Temperature over a 24 period. For fun I also added the temp sensor from another device in that same room to compare the room to the beer temp.

To make sure the entire system is working I also used a template to show the last update time for the data. This was pretty simple using a `relative_time` Jinja function.

![Home Assistant Graphs](/images/2024-06/tilt_graphs.png)

{% raw %}
```
{{ relative_time(as_datetime(state_attr(entity, "timestamp"))) }}
```
{% endraw %}

Overall I'm pretty happy with how it turned out. With the data in Home Assistant I can pull out any metrics I want since the data is all stored. The flow of data ended up being a bit more complicated than I expected but the additional dependency on the Pitch project didn't make it too bad. If I purchase any more ESP32 devices in the future I'll make sure they support Bluetooth so I can always migrate to my original BLE Proxy based solution later.

## Links

* [Tilt][tilt] - Tilt bluetooth hydrometer
* [Home Assistant][home-assistant] - Open source home automation platform
* [ESPHome Bluetooth Proxy][esp-proxy] - use ESP32 device to act as Bluetooth proxy
* [Pitch][tilt-pitch] - Python program to grab Tilt data and push it to upstream services
* [Mini Graph Card][graph-card] - Popular Home Assistant graph card to display multiple values

[tilt]: https://tilthydrometer.com/
[home-assistant]: https://www.home-assistant.io/
[esp-proxy]: https://esphome.io/components/bluetooth_proxy.html
[tilt-pitch]: https://github.com/linjmeyer/tilt-pitch
[ha-webhook]: https://www.home-assistant.io/docs/automation/trigger/#webhook-trigger
[ha-mqtt]: https://www.home-assistant.io/integrations/sensor.mqtt/
[graph-card]: https://github.com/kalkih/mini-graph-card
