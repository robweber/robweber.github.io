---
layout: post
title:  "Homemade Dynamic DNS"
author: robweber
categories: automation random
tags: home-assistant python scripts
---

In my home lab I have a handful of services that need to be accessed from outside the house via the internet. For me [Home Assistant][home-assistant] is my number one need but there are others. Of course the problem with this is that a residential ISP doesn't give you a static IP so your outside address changes every so often. What usually ends up happening in these situations is that to get a static IP you need to upgrade your internet plan or pay some additional monthly fee - which I of course don't want to do. [Dynamic DNS][dynamic-dns] is supposed to solve this problem. The idea is you setup a static host record and then update the IP it points to when your DHCP address with your ISP changes.

Dynamic DNS is pretty easy to nail down and there are probably hundreds of services that do it. You can even get routers with the ability to do it built in. My specific problem with these services is that you often get a really weird URL like __hostname.dynamicservice.com__. Nothing wrong with that if you're getting this service for free but the second you want to add something additional, like a custom domain or SSL cert, you have to start paying the dynamic DNS service. I don't want to pay them either! I want my cake and I want it for free.

<!--more-->

{% include toc %}

## My Problem

I have my own custom domain that I want to use to access my external services. Additionally I have this domain tied to a [Let's Encrypt][lets-encrypt] free SSL certificate for security. This way I can properly secure my internet facing services. In order to get this to work I need to be able to have a reliable DNS entry for the domain name that matches my SSL certificate and points to the DHCP address on my router given by the service provider.

Making the DNS entry is easy. Domain registrars typically offer a basic DNS service. To both have my cake and get it for free I need to find a way to update the DNS entry when my DHCP address from my ISP changes. To do this I'm going to lean on some web APIs and Python to develop a script that can compare my current external IP address with the one in the DNS record. The goal is that when they don't match to update the record automatically.

## Finding Your Outside IP

Finding your outside IP is as easy as typing "[what's my ip](https://www.google.com/search?q=what%27s+my+ip)" into Google. To automate things though something that can be read in via a web service is a better mechanism.

There are a lot of websites that will read the IP address from your incoming TCP request and just echo it back to you. I decided [Ipify][ipify] would work well for what I wanted. It was easy to use and can return payloads as a JSON object. To try it out just open a browser to [https://api.ipify.org/?format=json](https://api.ipify.org/?format=json).

```json
# this is the cloudflare DNS IP, you'll see your own here
{"ip":"1.1.1.1"}
```

Ipify does have it's own [Python package](https://pypi.org/project/ipify/) to make this a plug-and-play Python solution; however I had trouble getting it working. It threw a bunch of errors and rather than track them all down I decided doing a quick GET request using the `requests` module was way faster and not too hard. I threw together this code to quickly download the address or throw an error if something is up with the website (internet down, etc).

```python
import requests

def get_external_ip():
    """
    try and get the external IP from ipify.org
    raise an error if no response found

    :returns: the external IP as a string
    """
    result = None
    response = requests.get('https://api.ipify.org?format=json')

    if(response.status_code == 200):
        result = response.json()
    else:
        raise Exception("Ipify not available")

    return result['ip']

print(get_external_ip())
```

## Updating DNS

I'll admit I got a bit lucky here. GoDaddy is my domain registrar and I use them to manage DNS records as well. They offer a pretty decent [web API][godaddy-api] for their services, which includes a method to both get and update DNS records. This was really the key to the whole system and without it I'd be stuck triggering a notification and then manually updating the records - no thank you.

To use the GoDaddy API you have to generate some keys, which requires a GoDaddy account. This is pretty easy and outlined in their [Getting Started](https://developer.godaddy.com/getstarted) guide. Once you have the keys you'll just need to add them to the header of every request in the form `Authorization: ssokey {go_daddy_key}:{go_daddy_secret}`.

### Retrieving Records
Looking up existing DNS records is pretty easy using the endpoint [/v1/domains/{domain}/records/{type}/{name}](https://developer.godaddy.com/doc/endpoint/domains#/v1/recordGet). This same endpoint is used to both read and update the DNS record depending on the type of HTTP request. GET requests will read the record while PUT requests will update a record. Breaking down the endpoint syntax a little more you need to supply:

* __{domain}__ - this is the high level domain name
* __{type}__ - this is the type of record, in my case an A record but there are other [DNS record types](https://en.wikipedia.org/wiki/List_of_DNS_record_types)
* __{name}__ - the name of the record. My record is the root of the domain so it's designated with `@` but for a subdomain you could use `subdomain` as the name to reference `subdomain.domain.com`.

As an example, if you wanted to get the CNAME record for `email.domain.com` you could use the following URL and get the resulting payload:

```
# GET URL
https://api.godaddy.com/v1/domains/domain.com/records/CNAME/email

# Response Payload
[
  {
    "data": "@",
    "name": "email",
    "ttl": 3600,
    "type": "CNAME"
  }
]
```

### Updating Records

Updating records is the [reverse of looking them up](https://developer.godaddy.com/doc/endpoint/domains#/v1/recordReplaceTypeName). Using the same endpoint a PUT request with a JSON object payload will update the record's information. Below is an example in Python using the `requests` library.

```python
import json
import requests

# GoDaddy API information
secret = ""
key = ""

# using the email.domain.com example from above
url = 'https://api.godaddy.com/v1/domains/domain.com/records/CNAME/email'

# the data to update, payload expects an array of dicts
data = [{"data": "mail.domain.com", "ttl": 3600}]


response = requests.put(url, data=json.dumps(data),
                        headers={"Authorization": f"sso-key {key}:{secret}",
                                 "Content-type": "application/json"})

# return code should be 200
print(response.return_code)

```

## Final Script

The final workflow is pretty easy. Retrieve both the current external IP, the GoDaddy DNS information, and then compare them. If they differ the GoDaddy entry should be updated. The full script is below where I added some parameters that can be passed in on the command line, or held in a config file. This makes it easy to use the script for more than a single DNS entry if necessary. I have this setup to run once a day on a server on my home network.

__Note:__ There is a lag between the time the DNS entry is updated and when it will propagate through the internet. You can see the time to live (TTL) is defaulted to 3600 seconds (60 min) in the records above. This does mean that there may be times when services are unavailable waiting for a new IP to take effect. In practice I've noticed the only time my IP really changes is when my ISP modem reboots - generally this only happens during a sustained power outage.

<script src="https://gist.github.com/robweber/3e83c29b38a60bf896e20d2924e424f7.js"></script>

## Links

* [Home Assistant][home-assistant] - home automation platform, useful to have exposed to the internet for remote access
* [Dynamic DNS][dynamic-dns] - high level description of dynamic DNS and it's purpose
* [Let's Encrypt][lets-encrypt] - Free SSL Certificate Authority for web certificates
* [Ipify][ipify] - service to get IP information
* [GoDaddy API][godaddy-api] - GoDaddy's domain API documentation

[home-assistant]: https://www.home-assistant.io/
[dynamic-dns]: https://en.wikipedia.org/wiki/Dynamic_DNS
[lets-encrypt]: https://letsencrypt.org/
[ipify]: https://www.ipify.org/
[godaddy-api]: https://developer.godaddy.com/doc/endpoint/domains#/
