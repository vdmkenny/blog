---
title: "Spawncamping Public EV Chargers"
subtitle: "Reverse engineering an API to notify when a charger is free."
date: "2024-06-01"
toc: true # table of contents (only h2 and h3 added to toc)
bold: true # display post title in bold in posts list
math: false # load katex
tags:
  - Python
  - EV
next: true # show link to next post in footer
draft: false
---

## Disclaimer

Using APIs like this is a bit naughty. Use sensible scraping intervals to avoid introducing any unusual load or burden on the API server.

I don’t take any responsibility for the misuse of any APIs, those mentioned in this article or otherwise, by readers of this blog post.

## Background

I have an electric car, but I don't have a driveway or another way to charge my car at home. Because of this, I'm dependent on the very meager public charger offering in my area.
Usually I can charge at work, but sometimes I am unable to or forget, and I end up with a near-dead battery at my house. 

It wouldn’t be so bad since there is a public charger in my street, but unfortunately, these are almost constantly occupied.
In this case, the charger is operated by [Allego](https://www.allego.eu/), a very common sight in Europe.

## Public website

On their [website](https://www.allego.eu/fastcharger/map/ultrafast), they have a map with all of their chargers. When you select a charger, you can see additional information. Most importantly, it has a list of the plugs on this charger (usually 2) and their status.
![Allego\'s website](/blog/allego/allegosite.png)
It has to get this information, somewhere right?

![Inspecting the network requests](/blog/allego/allegonetworktab.png)
Pressing **F12** in Firefox, and opening the Network tab, you can see all the requests the browser is making. 
Filtering by "api" and clicking around, it was easy to find the API call that was doing the lookup.

The API endpoint in this case is: `https://www.allego.eu/api/feature/experienceaccelerator/areas/chargepointmap/getchargepoints/NLALLEGO015053`.

The last part is the ID of the charger. So what happens if we open this URL in a browser directly? It's not even worth it to add a screenshot: the API returns a whole load of nothing.

What's the difference? Well, if we go back to the Network tab, we can see that in the original request, our browser attached a bunch of headers, along with some cookies.

## Coding time

Let's write a simple python script to see if we can make the API reply to us using these headers and cookies. I redacted the specific ones I am using, but it's easy to figure out these values using the method we just used.

```python
import requests

# URL for the API request
url = "https://www.allego.eu/api/feature/experienceaccelerator/areas/chargepointmap/getchargepoints/NLALLEGO015053"

# Headers copied from the browser
headers = {
    "Host": "www.allego.eu",
    "User-Agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10.15; rv:126.0) Gecko/20100101 Firefox/126.0",
    "Accept": "*/*",
    "Accept-Language": "nl,en-US;q=0.7,en;q=0.3",
    "Accept-Encoding": "gzip, deflate, br, zstd",
    "X-Requested-With": "XMLHttpRequest",
    "Connection": "keep-alive",
    "Sec-Fetch-Dest": "empty",
    "Sec-Fetch-Mode": "cors",
    "Sec-Fetch-Site": "same-origin",
    "Priority": "u=1"
}

# Cookies copied from the browser
cookies = {
    "allegoeu#lang": "en",
    "ASP.NET_SessionId": "redacted",
    "SC_ANALYTICS_GLOBAL_COOKIE": "redacted",
    "sxa_site": "AllegoEu",
    "ARRAffinity": "redacted",
    "ARRAffinitySameSite": "redacted",
    "_ga4789": "map",
    "langcode": "en",
    "CookieConsent": "{stamp:'redacted',necessary:true,preferences:true,statistics:true,marketing:true,method:'explicit',ver:2,utc:1717232236568,region:'be'}"
}

# Sending the request
response = requests.get(url, headers=headers, cookies=cookies)
print(response.text)
```

```
➜  python3 allego.py
{
  "address": {
    "addressLine1": "Rijksweg 2A",
    "addressLine2": "",
    "postalCode": "3755 MV",
    "city": "Eemnes",
    "stateProvince": null,
    "country": "NL",
    "fullAddress": "Eemnes, Rijksweg 2A, 3755 MV"
  },
  "capabilities": null,
  "contact": {
    "contactCenter": null,
    "email": "",
    "phone": "+31880333033",
    "website": ""
  },
  "evses": [
    {
      "connectorType": "IEC_62196_T2_COMBO",
      "current": "DC",
      "displayName": "CCS (Combo type 2) 300 kW",
      "id": "1",
      "maxPower": 300.0,
      "status": "Available"
    },
    {
      "connectorType": "IEC_62196_T2_COMBO",
      "current": "DC",
      "displayName": "CCS (Combo type 2) 300 kW",
      "id": "2",
      "maxPower": 300.0,
      "status": "Charging"
    }
  ],
...
```

Bingo! Under the `evses` field, we can see the different connectors, along with their status.

## Full Proof of Concept code

What I want to do now is monitor my local charger by continuously checking the status of the plugs and alert me if one is free.
We can adapt our previous script to add a loop and some configuration options, and make it alert if a plug becomes available.

To alert me, I used [osascript](https://ss64.com/mac/osascript.html) to generate a popup as a proof of concept.

Again, I redacted the specific headers and cookie values I am using, but it's easy enough to figure out.

```python
import requests
import argparse
import time
import sys
import subprocess


class AllegoChargePoint:
    def __init__(self, chargepoint_id):
        self.chargepoint_id = chargepoint_id
        self.url = f"https://www.allego.eu/api/feature/experienceaccelerator/areas/chargepointmap/getchargepoints/{chargepoint_id}"
        self.headers = {
            "Host": "www.allego.eu",
            "User-Agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10.15; rv:126.0) Gecko/20100101 Firefox/126.0",
            "Accept": "*/*",
            "Accept-Language": "nl,en-US;q=0.7,en;q=0.3",
            "Accept-Encoding": "gzip, deflate, br, zstd",
            "X-Requested-With": "XMLHttpRequest",
            "Connection": "keep-alive",
            "Sec-Fetch-Dest": "empty",
            "Sec-Fetch-Mode": "cors",
            "Sec-Fetch-Site": "same-origin",
            "Priority": "u=1"
        }
        self.cookies = {
            "allegoeu#lang": "en",
            "ASP.NET_SessionId": "redacted",
            "SC_ANALYTICS_GLOBAL_COOKIE": "redacted",
            "sxa_site": "AllegoEu",
            "ARRAffinity": "redacted",
            "ARRAffinitySameSite": "redacted",
            "_ga4789": "map",
            "langcode": "en",
            "CookieConsent": "{stamp:'redacted',necessary:true,preferences:true,statistics:true,marketing:true,method:'explicit',ver:2,utc:1717232236568,region:'be'}"
        }
        self.data = None

    def fetch_data(self):
        response = requests.get(self.url, headers=self.headers, cookies=self.cookies)
        if response.status_code == 200 and response.text:
            self.data = response.json()
        else:
            raise Exception("Failed to fetch data or empty response.")

    def parse_data(self):
        if self.data is None:
            raise Exception("No data to parse. Please fetch data first.")

        chargepoint = self.data
        evses = chargepoint.get("evses", [])
        plugs = []
        for evse in evses:
            plug_info = {"id": evse.get("id"), "status": evse.get("status")}
            plugs.append(plug_info)
        return plugs


def print_plugs_info(plugs):
    if not plugs:
        print("No plugs available in the Charge Point.")
        return None
    else:
        print("Plugs in the Charge Point:")
        for plug in plugs:
            status = plug["status"]
            id = plug["id"]
            if status == "Charging":
                status_color = "\033[91m"  # Red
            elif status == "SuspendedEV":
                status_color = "\033[93m"  # Yellow
            elif status == "Available":
                status_color = "\033[92m"  # Green
            else:
                status_color = "\033[0m"  # Reset
            print(f"ID: {id}, Status: {status_color}{status}\033[0m")
        return plugs


def show_popup(chargepoint_id, plug_id):
    message = f"Plug {plug_id} is available in Charge Point {chargepoint_id}!"
    applescript = f'display dialog "{message}" with title "Charge Point Monitor" buttons {{"OK"}}'
    subprocess.call(["osascript", "-e", applescript])


def main(chargepoint_id, interval):
    print(f"Monitoring Charge Point {chargepoint_id} at an interval of {interval} seconds.")
    chargepoint = AllegoChargePoint(chargepoint_id)
    while True:
        try:
            chargepoint.fetch_data()
            plugs = chargepoint.parse_data()
            available_plugs = [plug for plug in plugs if plug["status"] == "Available"]
            print_plugs_info(plugs)
            if available_plugs:
                for plug in available_plugs:
                    show_popup(chargepoint_id, plug["id"])
                    sys.exit(0)
            else:
                print("None of the plugs are available.\n")
            time.sleep(interval)
        except KeyboardInterrupt:
            print("Exiting...")
            sys.exit(0)
        except Exception as e:
            print(f"Error: {e}")


if __name__ == "__main__":
    parser = argparse.ArgumentParser(description="Monitor Allego Charge Point Availability")
    parser.add_argument("chargepoint_id", type=str, help="The ID of the charge point to monitor")
    parser.add_argument(
        "--interval",
        type=int,
        default=60,
        help="The interval in seconds to scrape the charge point status (default: 60 seconds)",
    )
    args = parser.parse_args()

    main(args.chargepoint_id, args.interval)
```

This script takes the Charge Point ID as a parameter, and allows to set the scraping interval. Don't set it below 60 seconds, this is pretty pointless and will only annoy the owner of the API server.
If anything, set it slower when the plugs around you aren't as chronically occupied.

Now when you're waiting for a charger, you can let this script run, and it will alert you when a plug opens up.


![Full script demo](/blog/allego/allegoscript.png)


## Conclusion

It's often pretty easy to figure out how the API behinds a web application works, especially if you don't need to log in.
The tokens we got from the original browser request might expire. Some might not be needed, or some could be randomly generated.

Experiment at your own risk.

This script could be improved, and is only a proof of concept. For a more "production"-grade setup, you could write the status to a time-series database like [InfluxDB](https://www.influxdata.com/) or even write a [Prometheus Exporter](https://prometheus.io/docs/instrumenting/exporters/) if you're feeling fancy.
Then, you could use a visualisation tool like [Grafana](https://grafana.com/) and even use their built-in [alertmanager](https://grafana.com/docs/grafana/latest/alerting/fundamentals/notifications/alertmanager/) to integrate the notifications into a more convenient system.

This is beyond the scope of this quick post. If you do build anything like this, please be aware that the owners of the API usually [don't like this type of thing](https://github.com/Andre0512/hon?tab=readme-ov-file#takedown-story).

Be mindful about what and how many requests you throw at these servers.
