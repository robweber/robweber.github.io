---
layout: post
title:  "Parsing Nagios Performance Data"
author: robweber
categories: coding
tags: python
---

Recently I was adding features to [Trash Panda][trash-panda], a local monitoring system project [I probably never should have started](/coding/trash-panda/). I thought it might be interesting to do something with the performance data many Nagios plugins send back as part of service check results. Performance data is additional data about the service that can be useful as history for how the service is performing. Stuff like disk space metrics, TCP return times, and other types of numeric values. The format is standardized and could then be easily piped into a database or other storage area for later use. This post documents my efforts from loops within loops to something bordering on efficient.

I do want to note __I did__ try and find a tool for this before building something from scratch. There are some command line tools but nothing in a library format. Since the Trash Panda project is written in Python that's really what I needed.

<!--more-->

{% include toc %}

## Performance Data format

To set the stage a little bit - what is what exactly is performance data and how is it formatted?

Nagios has [guidelines](https://nagios-plugins.org/doc/guidelines.html) for how service checks should be written. This includes stuff like common argument syntax and program output. This maintains consistency for the many hundreds of custom service checks developed for Nagios, and other programs that follow similar conventions. Trash Panda utilizes Nagios type service checks specifically because of how many are available. Part of these guidelines allow for the inclusion of [performance data][perf-data-format] in the output. Simply put performance data is a list of key/value pairs that contain relevant information about the service check. Here is an example from a disk space check and an http check.

```
# disk check output
DISK OK| /=112521MB;134087;141536;0;148986 /dev=0MB;1708;1803;0;1898 /media/mythtv=232633MB;271969;287078;0;302188 /tmp=112521MB;134087;141536;0;148986 /var/tmp=112521MB;134087;141536;0;148986

# http check output
HTTP OK: HTTP/1.1 200 OK - 16499 bytes in 0.019 second response time |time=0.019284s;;;0.000000;10.000000 size=16499B;;;0

```

According to the guidelines everything in the output after the pipe __`|`__ symbol is performance data. You can see for the disk check there is data about the various mount points and their space utilization. For the http check it includes the response time for the packets sent and the size of the packet. Other types of service checks will send different information, but most importantly they will all utilize this format. Breaking the format down further each data point should follow this convention:

```
'label'=value[UOM];[warn];[crit];[min];[max]
```

You can read more about the guidelines on the [Nagios site][perf-data-format] but below is a quick summary. Each data point should include:

* __label__ - the value key
* __value__ - the value and it's unit of measure (__UOM__)
* __warn__ - the value when this data point is considered at a warning level (_optional_)
* __crit__ - the value when this data point is considered at a critical level (_optional_)
* __min__ - the minimum allowable value (_optional_)
* __max__ - the max allowable value (_optional_)

Also worth noting is that you can include any of the optional values and skip others. Looking at the __time__ key value from the http check above you'd have the following metrics:

```
key: time
value: 0.019284
unit_of_measurement: s
warn: "absent"
critical: "absent"
min: 0.0
max: 10.0
```

## Parsing Performance Data

The format is easy enough. Here is my first super crude attempt at parsing it.

{% gist 4df52fbfc44cdfe46137a6ac21cfcfd1 attempt1.py %}

This code is offensive just to look at but it gets the job done - sort of. One major issue is that the value part of the response is not a number since it contains the unit of measurement. This means it can't easily be cast to a number value and used in any meaningful way. To separate the value from the unit of measure there is a bit more processing that needs to happen. Accounting for that I had some additional code like this:

```
# go through each character and determine if it's a number or decimal
value = ''.join(list(filter(lambda c: c.isdigit() or c == '.', key_value_array[1])))

# subtract length of value from string to leave UOM
result['uom'] = key_value_array[1][len(value):]

result['value'] = float(value)

```

What is all that nonsense? Essentially it is taking the value `"0.019284s"` and iterating over every character to filter out non-decimal or number characters. What is left is joined back into a string (`"0.019284"`). For the unit of measure the length of this string is removed from the whole to leave the unit of measure behind - in this case `s`. The string is then cast to a decimal to get the value as a number.

### Parsing Efficiency

The method above is a start but is not efficient. Right out of the gate mapping the optional values could be made much easier. Using a list to represent the values means a loop can be added to reduce redundant code.

```
perf_order = ["value", "warning", "critical", "min", "max"]
  for i in range(1, len(p_values)):
      if(p_values[i].strip() != ''):
          result[perf_order[i]] = float(p_values[i])
```

This looks better but could be simplified this even more using [list comprehension](https://docs.python.org/3/tutorial/datastructures.html#list-comprehensions) and filtering. Here is basically the same code but converting the `for` loop down to one line.

```
perf_order = ["value", "warning", "critical", "min", "max"]
# create dict of values, converting to a float where the value is not blank
result = {perf_order[i]: float(p_values[i]) for i in range(1, len(p_values)) if p_values[i].strip() != ''}
```

### Regular Expression Parsing

This was all getting better but I still wasn't satisfied. It occurred to me that what I'm really looking for is a maximum of 5 values, along with the label and unit of measurement. Instead of all the `.split()` statements could I find the numeric values in a different way and utilize the one-line statement above to get everything into a dictionary object? I jumped on to [Regex Pal][regex-pal] to see if I could get a [regular expression](https://en.wikipedia.org/wiki/Regular_expression) to do the job.

```
import re

# find all the numeric values
  values = []
  for t in re.finditer("(=|;)[+-]?((\\d+(\\.\\d+)?)|(\\.\\d+))|;", p):
      values.append(t.group()[1:])

  print(values)
```

The above expression finds any numeric value that is either preceded by an equals sign or semicolon, or followed by a semicolon. This fits the format of the performance data string. Parsing the example http statement gives an array of values for each data point. _Note the blank values for missing optional values._

```
['0.019284', '', '', '0.000000', '10.000000']
['16499', '', '', '0']
```

Combining everything together the entire function now looked like this:

{% gist 4df52fbfc44cdfe46137a6ac21cfcfd1 attempt2.py %}

Looking through this it's more or less a combination of the methods above with a few small tweaks. The first is converting the string values to decimals after finding the unit of measure. I didn't want to do this, adding another list comprehension, however it was necessary. The decimal conversion turned values like `123` into `123.0`. This made the length check not work properly and as a result units of measure were being cut off. Doing this later helped with the __UOM__ slicing.

Another tweak was needed for the unit of measure check as well. Instead of dealing with another `.split(';')` operation I instead opted to slice the string based on two indexes I could find quickly. The first is the length of the numeric value. The second is the index of the first semicolon (;) in the string. If a unit of measure exists it should be bookended between them like this - `16499B;`

### Dealing With Spaces

All in all I was feeling pretty good about this so I ran it on a bunch of performance data outputs to make sure it was working. Everything went great until I ran into this one: `'Disk Status 1'=1`. Doesn't really look like that much of a problem until you realize the very first delimiter the function looks for is a space to split the data points on. This label contains a space, totally messing up the parsing of everything else after.

The specification _does_ allow for a label to be encased in single quotes. What I needed to do was ignore this type of spacing while still finding the space delimiters between different data points. Enter some regular expression parsing to the rescue again. This one took a fair amount of Googling as well as guess/check/revise to nail down. When all was said and done this was the final parsing function which handled every example I could throw at it.

{% gist 4df52fbfc44cdfe46137a6ac21cfcfd1 final.py %}

## Wrapping It Up

Going back to the original version of this it's worth pointing out that it did basically work. Eventually some of the edge cases would have shown themselves (like the label spacing) but overall it was functional. All of the index checking and repetitive code just looked inelegant to me though. I don't think my final version really performs any faster, however it is a lot easier for me to follow and handles the edge cases more gracefully. Sometimes it's just fun to really sift through the process and see how well you can fine tune the algorithm (or at least it's fun for me).

## Links

* [Trash Panda][trash-panda] - local monitoring solution I built this parser for
* [Performance Data Format][perf-data-format]  - the Nagios Performance Data format guidelines
* [Regex Pal][regex-pal] - website to test regular expressions

[trash-panda]: https://github.com/robweber/trash-panda
[perf-data-format]: https://nagios-plugins.org/doc/guidelines.html#AEN200
[regex-pal]: https://www.regexpal.com/
