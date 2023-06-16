---
layout: post
title:  "Monitoring SNMP Traps"
author: robweber
categories: coding hardware automation work
tags: python linux yaml
---

In my profession life I recently purchased an [Infrasensing Base Unit][base-unit] along with a [door contact][door-sensor] sensor. These are pretty interesting IoT devices that accept a wide variety of sensors. My intention was to connect this to our network monitoring system, [Icinga][icinga], to monitor the status of a network cabinet door.

Upon playing around with the device I decided the best way to get timely alerts from the sensor was to utilize [SNMP traps][snmp-descrip] instead of polling for the status. While I have a lot of experience polling SNMP systems I'd never dug into traps before. Going down the rabbit hole of getting this going turned in to a pretty neat little project that can be utilized to translate SNMP trap payloads into something Icinga can accept for service checks.

![sensor page](/images/2023-06/default_sensor.png)

<!--more-->

{% include toc %}

## The Setup

In the beginning I had two systems. The Device and The Monitoring System, below are some details on each.

### The Device

The Infrasensing device itself is fairly simple. It is a POE device with a web interface for administration. By default the device has a built in temperature sensor to monitor it's own temp. Others are plugged in via the sensor module ports on the device.

It has some built in ways of getting alerts on the sensor status. The best for integration into our monitoring platform is SNMP.

| ![base unit](/images/2023-06/base_unit.jpg)  | ![door sensor](/images/2023-06/door_sensor.jpg) |

### The Monitoring System

Where I work we utilize the [Icinga Monitoring system][icinga] for network monitoring. Icinga is similiar to [Nagios](https://www.nagios.org/); and in fact check scripts are interoperable between them. Within Icinga you define a device (_host_) and a device can contain many _services_.

For these types of monitoring systems there are active and passive checks. An active check would be the Icinga daemon polling a device for it's status. A passive check is the device, or some external program, sending the service information to Icinga. Both have their use cases, although active checks are the most common.

### Data Flow

The normal data flow is to use active checks from Icinga to the target system. We typically do this in 5 min intervals to capture service statuses. For the door status though I didn't like the idea of polling. Someone could easily tamper with the door and circumvent the sensor within the 5 min polling window. Instead I wanted the Infrasensing device to _push_ notifications to Icinga instead. This would need to be done with passive checks.

## SNMP Traps

To get "instant" notifications from the device to Icinga I would need to use SNMP Traps. This was something I hadn't really had experience with before but knew it could be done. The one thing I knew from past investigations into this is that Icinga couldn't natively accept SNMP trap information as a target. I would need an intermediary to accept the information and forward it to Icinga.

### snmptrapd

The most common way to setup a trap handler is to configure the [SNMP Trap Daemon][trapd]. On most Linux systems this package is available by default by installing the `snmptrapd` package. On Ubuntu I was able to do this easily with:

```bash
sudo apt-get install snmptrapd -y
```

Once installed there are a lot of configuration options. The main thing that needs to happen is to accept incoming SNMP trap information and forward it to a handler. Through some trial and error I was able to land on the below config file to accept SNMP trap information from the door sensor device.

```
# TRAPD BEHAVIOUR (change to your own IP)

snmpTrapdAddr udp:127.0.0.1:162
doNotLogTraps no

################################################################################
# ACCESS CONTROL (change the community to your own)

authCommunity log,execute public
disableAuthorization yes

################################################################################
# NOTIFICATION PROCESSING

traphandle default /usr/bin/python3 /home/rob/Git/snmp-to-icinga/src/snmpicinga.py --config demo.yaml

```

Take note of the `disableAuthorization` flag. One thing I found is that the Infrasensing device was not able to authenticate trap notifications with the daemon. Once I disabled authorization the trap data came through. In some cases this could be a security risk. I plan on doing some firewall controls to filter traffic but word to the wise. The other thing to note is the path to the trap handler. This script doesn't exist (yet) but I'll define it below.

### Icinga Setup

The other piece of this is actually have a service in Icinga that can receive the trap information. This is similar to setting up a normal service but you have to set the check command to `passive` and disable active checks. I set this up in my system ahead of time so I could test processing passive check results.

```

object Service "check_rack_door" {
  import "generic-service"
  display_name = "Rack Door"
  host_name = "rack_door_sensor"
  check_command = "passive"

  enable_active_checks = false
  enable_passive_checks = true

  // send alert after first fail
  max_check_attempts = 1

}

```

The other thing you have to do is setup an [API User]() with permissions to the web API that can trigger the process check result endpoint.

```
object ApiUser "passive" {
    permissions = [ "actions/process-check-result" ]
    password = "WRITE_SOME_GOOD_PASSWORD_HERE"
}
```

## The Bridge

So after all this what I had working was:

1. __snmptrapd__ running to accept the door rack sensor information
2. A host and service in Icinga for the door sensor
3. An API user in Icinga to trigger the web api with the trap information

What I was lacking was the bridge code (`snmpicinga.py` from above) to actually tie everything together. The bridge needed to take the SNMP trap info and transform it into what the Icinga service required. I didn't know what to expect to come across in the device trap payload. Somewhere online I was able to find that the snmptrapd service would send a series of lines to the standard input of the handler script.

### Parsing Trap Information

First, I wrote a simple script to just log everything as a starting point.

```python
# first pass at snmpicinga.py
import sys

trap = sys.stdin.readlines()

with open('/home/rob/snmp.log', 'a') as log_file:
  log_file.write(f"{trap}\n")
```

To trigger it I simply opened the door sensor magnets on the base unit. The result was an array with a series of information.

{% raw %}
```
# appears to be the sender hostname, IP information, some OID info and finally the trap payload
\['sensorgateway.local\n', 'UDP: \[192.168.1.100\]:65534->\[192.168.1.101\]:162\n', 'iso.3.6.1.2.1.1.3.0 0:1:37:20.03\n', 'iso.3.6.1.6.3.1.1.4.1.0 iso.3.6.1.2.1.1.2\n', 'iso.3.6.1.4.1.17095.4.2.0 "Security1,TRIG,DOWN,01 January 2021,01:37:20"\n'\]
```
{% endraw %}

For this to be even remotely usable by Icinga I'd have to split out the info I needed. Namely the IP of the sender, the OID, and the payload. I revised my logging script to look like this:

```python
import re
import sys

# read in the full trap
trap = sys.stdin.readlines()

# network info is index 1 and payload is at the end
network_info = str(trap[1]).strip()
payload = str(trap[len(trap) - 1]).strip()

# get the sender's IP (first match)
sender = re.search('((\d+)\.){3}(\d+)', network_info).group()

# find the oid
oid = re.search('iso(\.([\d]{0,}))+', payload).group()
payload = payload[len(oid) + 1:]

# write the payload
with open('/home/rweber/snmp.log', 'a') as log_file:
  log_file.write(f"[{sender}] OID {oid}: {payload}\n")
```

Don't judge my regular expressions too harshly - I don't write these very often. The end result proved I could extract what I needed and craft some logic to decide what to do with the payload information.

### Sending To Icinga

The next test was to see if I could send the passive check information to Icinga [utilizing the API][passive-check]. The Icinga documentation had a `curl` command that I [quickly turned into a Python code](https://github.com/eau-claire-energy-cooperative/snmp-to-icinga/blob/main/src/snmpicinga.py#L19) utilizing the `requests` module. In a nutshell it requires:

* __host.name__ - the Icinga host
* __service.name__ - the Icinga service
* __exit_status__ - a numeric value (0-3) that corresponds to the service state (OK, WARNING, CRITICAL, UNKNOWN).
* __plugin_output__ - some text information about the service state

The final glue to the whole thing would be using the trap payload information to decide what state the door control was in, and sending it to Icinga with the API.

## Putting It All Together

Admittedly, with all the pieces of the puzzle in place I went full over-achiever and added some bells and whistles. The final project can be [found on Github][snmp-to-icinga]. I ended up with a script that reads in a YAML file with instructions to parse any generic SNMP trap text to determine the service state. For my door sensor the config looks like the following:

{% raw %}
```yaml
traps:
  - name: "Door Sensor"
    snmp:
      host: 192.168.1.100
      oid: iso.3.6.1.4.1.17095.4.2.0
      payload_type: csv
    icinga:
      host: rack_door_sensor
      service: check_rack_door
      return_code:
        ok: "{{ payload[1] == 'OK' }}"
        critical: "{{ payload[1] == 'TRIG' }}"
```
{% endraw %}

I'll break this down a section at a time, starting with the `snmp` node. This is where I define the sender of the payload, and what OID I'm looking for. This is all parsed from the trap using the code above. The script compares it to decide if this is the service we're looking for. You can define multiple traps in a single configuration file.

Assuming the host and OID match, the payload is parsed using the `payload_type` key. This can be one of 3 values:

* __value__- leave it unchanged
* __json__ - parse the string as JSON
* __csv__ - parse the string as a CSV, splitting on the commas

The door control unit sends it's payload as a CSV (`Security1,TRIG,DOWN,01 January 2021,01:37:20` from above). Once the payload is parsed the `icinga` information is loaded. Here is where the return code values are defined. Using the CSV array, column 2 (`payload[1]`) contains the value of either OK or TRIG. The evaluation, using [Jinja](https://palletsprojects.com/p/jinja/), returns either a True or False depending on the state of the door sensor. This is sent as the exit value to the Icinga service.

The end result is the updated state, each time the door sensor is triggered, in Icinga. In testing this happens within 2-3 seconds.

![icinga screenshot](/images/2023-06/rack_door.png)

## Conclusion

I started out with a pretty simple goal and took kind of a circuitous route to get there. I could have easily written a handler just for the door sensor and probably spent 50% less time on it. However, once I had the SNMP Trap information parsed it seemed worthwhile to build a solution I can apply to other traps. Now I can go back through some of our other network devices and see what kind of trap sensors exist on those that may be worth while to monitor. If nothing else I can now say I have experience with SNMP traps and how to deal with them.

## Links

* [snmp-to-icinga][snmp-to-icinga] - the completed project
* [Passive Check Info][passive-check] - information on Icinga passive check processing
* [Infrasensing Unit][base-unit] - the sensor unit I'm using
* [SNMP][snmp-descrip] - High level overview of the SNMP protocol


[base-unit]: https://infrasensing.com/sensors/baseunit.asp
[door-sensor]: https://infrasensing.com/sensors/sensor_sec_doorcontact.asp
[icinga]: https://icinga.com/
[snmp-descrip]: https://en.wikipedia.org/wiki/Simple_Network_Management_Protocol
[trapd]: https://manpages.ubuntu.com/manpages/bionic/man5/snmptrapd.conf.5.html
[passive-check]: https://icinga.com/docs/icinga-2/latest/doc/12-icinga2-api/#process-check-result
[snmp-to-icinga]: https://github.com/eau-claire-energy-cooperative/snmp-to-icinga
