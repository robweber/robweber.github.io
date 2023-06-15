---
layout: post
title:  "Home Assistant Update Notifications"
author: robweber
categories: automation smarthome
tags: home-assistant yaml
---

Starting with version [2022.4](https://www.home-assistant.io/blog/2022/04/06/release-20224/#introducing-update-entities), Home Assistant has support for [Update entity types][update-entity]. Using this entity type, devices (or Home Assistant itself) can let the user know of updates to software and firmware. At it's most basic level this acts as a binary sensor to tell you if an update is available (when on). The update entity also has lots of information such as the currently installed version, the new version available, release notes URL, additional service calls and much more.

The documentation for the Update entity includes a basic automation for how to receive notifications for when an update is available. The problem with this example is it only works for a __specific__ Update entity (like Home Assistant Core updates). I wanted a solution to let me know when an update to __any__ update entity was available. I found this can be done with a bit of thought and a [Template sensor][template-sensor].

<p align="center">

<img src="/images/2022-09/update_details.png" />

</p>
<!--more-->

{% include toc %}

## The Problem

Below is the example automation for creating an update notification. This works great for a specific update entity (`update.my_light_bulb`). Your instance of Home Assistant may have dozens of update entities, which means you'd have to repeat this for each one. To create a generic update notification for any update you need to somehow "roll up" all the entities into a single yes/no value that triggers when any of them have an update available.

```yaml
automation:
  - alias: "Send notification when update available"
    trigger:
      platform: state
      entity_id: update.my_light_bulb
      to: "on"
    action:
      alias: "Send notification to my phone about the update"
      service: notify.iphone
      data:
        title: "New update available"
        message: "New update available for my_light_bulb!"
```

## Template Sensor

The solution I came up with was to create a [Template binary sensor][template-sensor] that contains all of the update entities currently in the `on` state. Playing around in the Developer Tools I came upon a Jinja filter that was able to generate a list of updates all in one line.

{% raw %}
```yaml
states.update | selectattr('state', 'equalto', 'on') | list
```
{% endraw %}

The `states.update` object contains all entities in the Update entity domain. Right out of the gate this is a filtered list of all the entities needed. The next filter `selectattr` performs a test on the attributes of each entity. In this case it's checking if the state of the entity is equal to `on`. The selectattr filter returns an iterator, so the last filter converts this to a list. This now contains a list of all update entities currently in the `on` state.

Using this selection criteria a binary template sensor can be defined as part of the Home Assistant config. This will roll up all the update entities into a single sensor to be used within the notification automation (or anywhere else you want it). For readability here I've set the list to a variable, `t` and then output a true/false value for if the length of the array is greater than 0. The `names` attribute will output a list of all entity names from the list. That will be useful later.

{% raw %}
```yaml
template:
  - binary_sensor:
     - name: "Update Available"
       state: >-
         {%set t = states.update | selectattr('state', 'equalto', 'on') | list %}
         {{ t | length  > 0 }}
       attributes:
         names: >-
           {% set t = states.update | selectattr('state', 'equalto', 'on') | list %}
           {{ t | map(attribute='name') | list }}
```
{% endraw %}

## Notification Automation

The resulting automation with this sensor will trigger any time an update entity finds it needs an update. Using the `names` attribute you can print a list of the entity names as well.

{% raw %}
```yaml
automation:
  - alias: "Send notification when update available"
    trigger:
      platform: state
      entity_id: binary_sensor.update_available
      to: "on"
    action:
      alias: "Send notification to my phone about the update"
      service: notify.iphone
      data:
        title: "New update available"
        message: >-
          "There is an update available in Home Assistant for the following integrations: {{ state_attr('sensor.update_available', 'names') | join(', ') }}"
```
{% endraw %}

## Limitations

One major problem here is that this is acting as a binary sensor. Once triggered to the `on` state you won't get additional notifications. If you wanted to make sure you got a notification on each and every update you could change this to a sensor type template. The state of the sensor would be the total count of available updates. Your automation trigger would be for when the state of this sensor changes (up or down) and then in the automation conditions you can check if the number is increasing or decreasing. When decreasing ignore it and when increasing send the notification.

I didn't need this level of granularity but using the `trigger.from_state` and `trigger.to_state` [template objects](https://www.home-assistant.io/docs/automation/templating/) you could figure it out.

## Bonus - Display In Lovelace

Using the very useful [Auto Entities][auto-entities] card in Lovelace you can also get a list of updates in your UI. Installing the card is out of scope for this but assuming you have it working you can create a card like the one below. This uses a conditional display to only show the card when there is an update for at least one entity. From here the Auto Entities card is loaded to do the state filtering and display the updates in a simple entity list. You can make this look a lot more custom to fit your theme as well with some tweaking.

```yaml
type: conditional
conditions:
  - entity: binary_sensor.update_available
    state: 'on'
card:
  type: custom:auto-entities
  filter:
    include:
      - entity_id: update.*
        state: 'on'
        options: {}
    exclude: []
  sort:
    method: friendly_name
  show_empty: true
  card_param: entities
  card:
    type: entities
    show_header_toggle: false
```

<p align="center">

<img src="/images/2022-09/updates_list.png" />

</p>

## Links

* [Update Entity Documentation][update-entity]
* [Template Sensor Documentation][template-sensor]
* [Built-In Jinja Filters][jinja-filters]
* [Auto Entities Lovelace Card][auto-entities]


[update-entity]: https://www.home-assistant.io/integrations/update/
[template-sensor]: https://www.home-assistant.io/integrations/template/
[jinja-filters]: https://jinja.palletsprojects.com/en/3.1.x/templates/#list-of-builtin-filters
[auto-entities]: https://github.com/thomasloven/lovelace-auto-entities
