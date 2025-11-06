---
layout: post
title:  "Custom Jinja Templates in Home Assistant"
author: robweber
categories: smarthome
tags: home-assistant
---

I have a Home Assistant automation that reads out some information each morning utilizing [text-to-speech][ha-tts] with [Piper][piper]. One of the first things it says is the current date and day of the week. My _big problem_ is that the text to speech engine is not very good at pronouncing dates. What I want is "today is Monday, September 1, 2025" with the date read as "first". What I get is the date read as "September one". The is a real first world dude-with-too-much-time-on-his-hands sort of problem but it does bother me. It's a bit jarring to hear and seems like the sort of thing that should _just work_.

Of course the problem is two fold. The first is that the date parsing code sends the sentence with the number __1__ instead of the written out word __first__ or even __1st__. This is an issue with the date translation working for the written word but not how it's spoken. The second is that the text-to-speech system doesn't know this is a date. It just sees a number and reads it. Utilizing some custom template macros I decided to try and fix this issue in a re-useable way.

<!--more-->

{% include toc %}

## Home Assistant Templates

The nuts and bolts of the whole problem comes down to modifying the text that is sent to the text-to-speech system. Ultimately what I need to do is send a written word, like "first", instead of the numeric value. In Home Assistant this kind of dynamic text is generated using [templates](https://www.home-assistant.io/docs/configuration/templating/). Home Assistant templates are built on [Jinja][jinja] with a lot of custom Home Assistant functions and filters. In my automation the date is created with the following template:

{% raw %}
```
Today is {{ as_timestamp(now()) | timestamp_custom("%A, %B %-d, %Y") }}
```
{% endraw %}

Basically, in order, the following is happening:

1. Get today's date - `now()`
2. Convert it to a Unix timestamp - `as_timestamp()`
3. Pipe the output to convert the timestamp into something readable - `timestamp_custom()`
4. Specify the output format using [strftime][strftime] format codes
5. The output is something like __Today is {{ site.time | date: '%A, %B %-d, %Y' }}__

For more information on each function checkout the [time section on templating](https://www.home-assistant.io/docs/configuration/templating/#time). If things were easy there would be a [strftime format code][strftime] for printing the date as a string, but there isn't.

## Num2Word

In the Python world there is a great little package called [Num2Words](https://pypi.org/project/num2words/) that does exactly what I'm looking for. For good reason, Home Assistant can't just import random Python modules. What I decided to do instead was build similar functionality within a custom Jinja template.

Home Assistant has support for [re-usable template macros](https://www.home-assistant.io/docs/configuration/templating/#enabled-jinja-extensions). These are pieces of Jinja syntax that you can re-use like a built-in function. This allows for some pretty awesome expansions of the existing Home Assistant template features. In fact there are lots of custom template resources in places like the [HACS store](https://www.hacs.xyz/).

### Creating A Template

To start creating my own macro I used the Home Assistant [Developer Tools](https://www.home-assistant.io/docs/tools/dev-tools/) area. This nice little sandbox let's you play with templates so you can fine tune them before putting them in a Script or Automation. I also found a nice [Stackoverflow](num2word) thread that demonstrated how to utilize a dictionary object to convert a number to a word. I used this as my starting point.

Breaking down my original template from above, what I really needed was just the date component. This is easy to extract by changing the format codes. Note the explicit cast to an integer using the `int` filter. This is important for the dictionary lookup later.

{% raw %}
```
{% set day = as_timestamp(now()) | timestamp_custom("%-d") | int %}
```
{% endraw %}

Now I had the date on it's own in a variable called `day`. The next bit was to utilize the dictionary method to convert this to words, but do so entirely within Jinja. This really wasn't that big of a task.

{% raw %}
```
{% set num2words = {1: 'One', 2: 'Two', 3: 'Three', 4: 'Four', 5: 'Five',
             6: 'Six', 7: 'Seven', 8: 'Eight', 9: 'Nine', 10: 'Ten',
            11: 'Eleven', 12: 'Twelve', 13: 'Thirteen', 14: 'Fourteen',
            15: 'Fifteen', 16: 'Sixteen', 17: 'Seventeen', 18: 'Eighteen',
            19: 'Nineteen', 20: 'Twenty', 30: 'Thirty', 40: 'Forty',
            50: 'Fifty', 60: 'Sixty', 70: 'Seventy', 80: 'Eighty',
            90: 'Ninety', 0: 'Zero'} %}
{% if day in num2words %}
  {{ num2words[day] }}
{% else %}
  {{ num2words[day-day%10] }}-{{ num2words[day%10] | lower }}
{% endif %}

```
{% endraw %}

Playing around with different values for `day` this will successfully print out "One", "Four" or "Twenty-one". Since I wanted these in more of a date format (first vs one) I modified the dictionary a bit and reduced it since I didn't need values greater than 31. I also added two special values to handle the fact that dates like 20 read as "twentieth" but 21 is "twenty-first". I ended up with something that looked like this.

{% raw %}
```
{% set day = as_timestamp(now()) | timestamp_custom("%-d") | int %}
{% set num2words = {1: 'First', 2: 'Second', 3: 'Third', 4: 'Fourth', 5: 'Fifth',
             6: 'Sixth', 7: 'Seventh', 8: 'Eighth', 9: 'Nineth', 10: 'Tenth',
            11: 'Eleventh', 12: 'Twelfth', 13: 'Thirteenth', 14: 'Fourteenth',
            15: 'Fifteenth', 16: 'Sixteenth', 17: 'Seventeenth', 18: 'Eighteenth',
            19: 'Nineteenth', 20: 'Twentieth', 30: 'Thirtieth', 120: 'Twenty',
            130: 'Thirty'} %}
{% if day in num2words %}
  {{ num2words[day] }}
{% else %}
  {{ num2words[day-day%10 + 100] }}-{{ num2words[day%10] | lower }}
{% endif %}
```
{% endraw %}

### Converting to Macro

Converting this to a reusable macro is fairly easy following the [instructions](https://www.home-assistant.io/docs/configuration/templating/#enabled-jinja-extensions) but I did have a few more problems to solve. The date value is just one of an entire string of values. To avoid a lot of hackery where I would have to create the day, month, and year values on their own I wanted to get everything working in one macro. This would require inserting my day value as a [strftime format code][strftime].

You can insert arbitrary values into the format code like this:

{% raw %}
```
{% set test = "The date is: "%}
{{ as_timestamp(now()) | timestamp_custom(test + "%-d") }}
```
{% endraw %}

I decided the easiest thing to do was insert my own format code value with my custom date string. Looking at the existing codes most of the letters are already used so I chose `%e` as my code value. Mostly because it was the letter after `d` and that's the one I would have used. Doing the replacement is pretty easy using the `replace` filter.

{% raw %}
```
{% set format = "%A, %B %e, %Y" %}
{% set format = format | replace("%e", num2words[day]) %}
{{ as_timestamp(now()) | timestamp_custom(format) }}
```
{% endraw %}

Using this I could finally wrap the whole thing up as a macro.

### Using in Automation

The final piece of the puzzle was to update my existing automation. You have to manually import your custom Jinja template prior to using it so in automations I ended up with:

{% raw %}
```
{% from 'date2words.jinja' import date2words %}
Today is {{ date2words(now(), "%A, %B %e, %Y") }}
```
{% endraw %}

All in all this was a fairly simple addition to make and dates are now read properly. It did occur to me I could extend this to read other numbers but I'd have to adjust the dictionary mappings and the logic to handle this. Handling any arbitrary number just wasn't a project I wanted to tackle - at least for now. This also gave me some experience in enhancing the Home Assistant template functions. I'll be on the lookout for other bits of template logic I'm repeating between Scripts and Automations to see if I can reduce these to macros as well.

## Links

* [Piper][piper] - Open source text to speech project compatible with Home Assistant
* [Home Assistant Templates](https://www.home-assistant.io/docs/configuration/templating) - documentation on custom HA templating
* [Python strftime][strftime] - docs for strftime behavior related to datetime output formatting

[ha-tts]: https://www.home-assistant.io/integrations/tts/
[piper]: https://github.com/rhasspy/piper/
[jinja]: https://jinja.palletsprojects.com/en/3.1.x/
[num2word]: https://stackoverflow.com/questions/19504350/how-to-convert-numbers-to-words-without-using-num2word-library
[strftime]: https://docs.python.org/3/library/datetime.html#strftime-strptime-behavior
