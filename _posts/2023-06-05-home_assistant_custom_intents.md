---
layout: post
title:  "Home Assistant Voice Custom Intents"
author: robweber
categories: smarthome
tags: home-assistant yaml
---

In 2023 Home Assistant is dedicated to making their smart home platform more capable of text and voice interaction through the [Year of the Voice][year-of-voice-1]. Essentially this is expanding on the capabilities of Home Assistant to interpret human text and speech into smart home actions. This provides more possibilities for existing voice assistants like Alexa and Google Home; while also providing an alternative through Home Assistant's own [Assist][assist] platform.

While a number of built-in intent phrases have been added, I've been experimenting with adding my own trigger phrases and intents. For the most part this was just following [the documentation][assist-sentences] however there were a few things that weren't clear, or just didn't work as expected. I'm documenting these here for future reference, and for anyone else struggling through the same thing. Hopefully as this part of Home Assistant matures adding these custom actions will become more clear.

![Assist Chat](/images/2023-06/assist_1.PNG)

<!--more-->

{% include toc %}

## Pipeline

Just for some background, Home Assistant has created a pipeline feature that takes human written sentences and interprets them into a Home Assistant command (intent). The core of this is a custom tool called [Hassil][hassil]. In a nutshell Hassil has two components. The first is a YAML based system for describing phrases that could be used to trigger an action. There could be several phrases for the same action, and Hassil allows you to define as many variations as possible. Within a sentence there is also syntax for specifying variables (slots in Hassil parlance) where you expect a user provided information. As an example:

```yaml
sentences:
  - "turn on the {device} in the {area}"
  - "turn off the {device} in the {area}"
```

These could match either _"turn on the lights in the living room"_ or _"turn off the outlet in the bathroom"_. The second part of Hassil, the interpreter, is what evaluates the user input against these rules at runtime. When a match is found the Home Assistant Intent that matches is executed. These sentences can get much more complex than I'm showing here. You can have optional word phrases, skip words, and even add contexts. These are all detailed in the [Template Syntax][assist-sentences] documentation.

Already built in to Home Assistant are dozens of sentence templates to trigger common actions. These have been transcribed into many languages and allow you to utilize the Assist tool within Home Assistant to interact with your smart home. Home Assistant is further expanding this system with [text to speech][wyoming] and [speech to text systems][wyoming], creating true virtual assistants.

## Customizing Intents

So the Hassil tool and built-in intents are all super exciting. Of course the first time I tried these out I asked my Home Assistant a common voice assistant question: _"what is it like outside?"_. The result was a failure. Home Assistant didn't have a built in way to give me this information. The reason is no fault of Hassil, or Home Assistant itself. There is simply too much variation to have a common built-in sentence to capture this. For example, what is _outside_? Is that an area I've defined or the actual outside? Even simplifying to _"what is the weather"_ is difficult. What service should I pull the weather info from? Home Assistant has dozens of weather integrations. What if I have my own weather station, should it use that? Obviously I need to create my own sentence and intent here.

The folks at Home Assistant have of course accounted for this and provided tools to add your own sentence templates. Asking for the weather proved a good example for starting to create custom actions.

### Sentence Templates

The documentation for all of this is pretty good. Basically I had to create two things; a custom sentence template for Hassil, and an [intent script][intent-scripts] for Home Assistant to execute. For the sentence template I first created a file within a specific folder of my HA config `config/custom_sentences/en/weather.yaml`. __EN__ is the language code, you can create sentence templates to match any language.

{% raw %}
```yaml
language: "en"
intents:
  OutsideTemperature:
    data:
      - sentences:
        - "what is the (temperature | weather) [outside]"
        - "what is it like outside"
```
{% endraw %}

For this example there really weren't any variables (slots) to worry about and the sentences were pretty basic. With this done I next created my custom intent script. These are added to your `configuration.yaml` file. Notice that the `OutsideTemperature` intent name is what ties these together. Hassil will tell Home Assistant to run this action when the sentences match the definitions. You can also have the intent trigger HA service calls but that wasn't necessary for this example. You'll notice here that I'm pulling my temperature from a personal weather station but the conditions come from a weather service. The power of the custom intent is I can mix and match the results however I want.

{% raw %}
```yaml
intent_script:
  OutsideTemperature:
    speech:
      text: "The weather outside is currently {{ states('sensor.gw2000b_v2_2_0_outdoor_temperature') | int }} degrees and {{ states('weather.keau_daynight') }}."
```
{% endraw %}

After reloading my config I could now ask Assist to give me the current weather conditions.

## More Complicated Intents

This all worked so great I wanted more custom phrases. For these examples I won't go through each iteration of what I tried but instead will post my working examples and where the gotchas occured. I spent _a lot_ more time on these than expected but once I knew where the problems were it got easier. Just as an aside, when testing these a quick reload of the config will refresh any sentence templates for Hassil but when configuring the intent scripts you need a full HA restart to reload any changes.

### Checking Presence

Something else that's useful to know is who is at home. Again, seems simple enough but there is currently no built-in sentence template for this type of question. I ended up with:

```yaml
language: "en"
intents:
  IsHome:
    data:
      - sentences:
        - "is <name> [at] home [right now]"
        - "is <name> in the house"
        requires_context:
          domain: 'person'
```

Picking this apart the `<name>` slot is where the name of the person will go. I'm utilizing one of the two built in Home Assistant slot lists, `<name>`, the other is `<area>`. These are available in any sentence template to match an entity name or area name and pass it along to the intent. If you want others you have to define your own lists. The other thing I added is a context that filters only on entities within the person domain. This means that _"is living room light in the house"_ won't match since that entity is not a person. A full list of domains can be found by running the following template in your Home Assistant's Developer Tools area. I like this better than linking to an online list since this will pull an up to date list.

{% raw %}
```jinja

{{ states | groupby('domain') | map(attribute='0') | list | join('\n') }}

```
{% endraw %}

When testing this with an intent script I noticed a problem right away. The `<name>` slot is populated by the entity name. Imagine the input is something like _"is Rob at home_". When calling the intent you end up with `Rob` in the name slot. While this is correct, I can't use the entity's friendly name to lookup any info about the entity. For that I'd need the entity ID, something like `person.rob`. For built in Home Assistant intents some kind of HA magic must happen to turn the name value into an entity ID, but this wasn't happening for my custom intent scripts. I turned to the Home Assistant Forums to see if anyone else had the same issue. Fortunately I was able to [get some help](https://community.home-assistant.io/t/custom-intents-using-name/571831/2) in the form of an HA template designed to turn a friendly name back into an entity id. This isn't the most ideal solution, since you'll have to add this to every intent script you make, but it is better than creating custom slot list for every possible value as part of the sentence. If I add a person to my Home Assistant install I don't have to circle back and update my sentence templates. Using this work around my intent script ended up looking like this:

{% raw %}
```yaml
IsHome:
  description: "determines if a specific person is home or away"
  speech:
    text: >-
      # this is where the entity id is found from the name value
      {% set person = states.person | selectattr('name', 'in', name) | map(attribute='entity_id') | list | first | default %}
      {% if has_value(person) %}
        {% if states(person) == 'home' %}
          Yes, {{ state_attr(person, 'friendly_name') }} is home
        {% else %}
          {{ state_attr(person, 'friendly_name') }} is not home
        {% endif %}  
      {% else %}
        Sorry, {{ name }} doesn't exist
      {% endif %}
  slots:
    name:
      description: "Name of the person"
      required: true
```
{% endraw %}

### Running A Script

Comparing to mapping the entity above this a lot simpler but still took a bit of thinking. I have a number of scripts that would be helpful to run based on different phrases. Think stuff like _"it's dinner time"_ or _"I'm heading out the door"_. I wanted to make the intent script generic so it could be easily re-used.

```yaml
TriggerScript:
  description: "triggers a script"
  action:
    service: "{{ script }}"
    data: {}
  slots:
    script:
      description: "the name of the script ie: script.script_name"
      required: true
```

This is a little different than the other examples in that there is no `speech` information given but there is an `action`. The action in this case is just a call to the script we're going to pass in as a slot from our template sentence. For the response I utilized a custom [response key](https://developers.home-assistant.io/docs/voice/intent-recognition/template-sentence-syntax#responses) for the different template sentences.

```yaml
language: "en"
# define sentence triggers for the script intent
intents:
  TriggerScript:
    data:
      - sentences:
        - "I'm heading out the door"
        - "I'm leaving now"
        - "Goodbye"
        slots:
          script: "script.goodbye"
        response: "goodbye"
      - sentences:
        - "[Tell everyone] it's dinner time"
        slots:
          script: script.notify_dinner
        response: "dinner_time"

# define responses for the script intent
responses:
  intents:
    TriggerScript:
      goodbye: "Sounds good, see you later"
      dinner_time: "Sending the notification"
```

The YAML above has two parts. The first part is defining the trigger sentences. For each group of sentences the same intent script `TriggerScript` is going to be triggered. What is different is that the `script` slot value is being hardcoded to the name of the script I want to trigger. This corresponds to the script slot in the intent script above. The final `response` key in each group of sentences tells Hassil to use that response for this trigger.

The responses are easier, you just define a different response for each given. [Per the documentation](https://developers.home-assistant.io/docs/voice/intent-recognition/template-sentence-syntax#responses) you can also reference slots within the response if you want.

![Assist Chat 2](/images/2023-06/assist_2.PNG)

## Wrapping It Up

The built in sentences and actions bundled with Home Assistant are pretty flexible on their own; but it's nice knowing you can always customize things more. It's actually pretty amazing to see the progress this has made in just the first half of the year. As a big fan of letting things run locally I'm looking forward to watching this mature to the point where we could drop our Google Home devices and use HA for everything. Integrating these custom actions with the [Voice over IP](https://www.home-assistant.io/voice_control/worlds-most-private-voice-assistant/) integration also makes for a pretty cool [bat phone](https://en.wikipedia.org/wiki/Bat_phone) style gadget.

## Links

* [Home Assistant Assist][assist]
* [Hassil][hassil]
* [Intent Scripts][intent-scripts]
* [Assist Sentence Template Documentation][assist-sentences]

[assist]: https://www.home-assistant.io/docs/assist/
[hassil]: https://github.com/home-assistant/hassil
[year-of-voice-1]: https://www.home-assistant.io/blog/2023/01/26/year-of-the-voice-chapter-1/
[year-of-voice-2]: https://www.home-assistant.io/blog/2023/04/27/year-of-the-voice-chapter-2/
[assist-sentences]: https://developers.home-assistant.io/docs/voice/intent-recognition/template-sentence-syntax/
[intent-scripts]: https://www.home-assistant.io/integrations/intent_script/
[wyoming]: https://www.home-assistant.io/integrations/wyoming/
