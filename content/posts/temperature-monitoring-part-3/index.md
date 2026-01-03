+++
title = "Temperature Monitoring - Part 3"
description = "Improving the temperature monitoring with multicast, prometheus and grafana"
date = 2025-12-16
[extra]
toc = true
[extra.comments]
id = "115831105808128754"
[taxonomies]
tags = ["rpi-pico","pico","raspberry-pi","rpi","temperature","monitoring","multicast","micropython"]
+++

# Introduction

For the moment, I'm still sending updates into IOT Plotter, but I wanted to have more flexibility (and I wanted to tinker some more, honestly).

# Pico Changes

So there have been a number of changes, additions, tidying and refactoring to the code running on the pico.

## Multicast

Now, on each polling run, for each temperature sensor, the pico sends a UDP multicast packet onto the network containing a JSON string. The string contains the sensor name and the current temperature.

One UDP packet for each sensor for each poll of the sensors.

What this means is that anything else around the network that is interested in the various temperatures, can subscribe to the multicast group and do as it wishes with what it receives.

This involves two bits of code, first a subroutine to send a multicast packet:

```python
def multicast(message_dict):
    # ...and encode it into JSON
    message = json.dumps(message_dict)

    # create the UDP socket
    sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)

    # ...and turn it into a multicast packet
    if hasattr(socket, "IP_MULTICAST_TTL"):
        sock.setsockopt(socket.IPPROTO_IP, socket.IP_MULTICAST_TTL, 1)

    # send the packet to the network
    sock.sendto(message, (config.MCAST_GROUP, config.UDP_PORT))
    Logger.print(f"sent multicast to {config.MCAST_GROUP}:{config.UDP_PORT} :: {message}")

    # ...and close the socket
    sock.close()
    # end of multicast
```

...and then later, we build a dictionary to pass in and then pass it into that subroutine...

```python
    # create the UDP message dictionary
    message_dict = {"type": "temperature", "name": sensor_name, "sensor": sensor, "temperature": temperature, "offset": sensor_offset}

    # and multicast it
    multicast(message_dict)
```

## Onboard LED

I had the onboard LED come on at the start of a polling cycle and go off at the end, just before the `sleep`. This gives at least a little bit of status without needing to hook up to the computer.

First import the library and intitialise the pin...

```python
from machine import Pin
led = Pin("LED", Pin.OUT)
```

...and then in the main loop we can switch the LED on and off...

```python
while True:
    led.on()

    # other polling, etc

    led.off()
    sleep(config.SLEEP)
```

## API Call Update

I also updated the bit of code that makes the API call, and had it send a multicast packet with a status update in it so there's also an ability to do some remote debug without the need to hook up Thonny being the first port of call.

This builds the `message_dict` within `try` or `except` and then passes it to the `multicast` subroutine in `finally` to send it.

# Prometheus & Grafana

I already use Prometheus elsewhere for collecting metrics from various things, but in all cases I'm using existing exporters and predefined Grafana dashboards. This was an opportunity to have a go from scratch. Learn. Understand.

## Prometheus Exporter

I wrote a basic exporter using the Python `prometheus_client` library.

On my Debian box[^1] this is as simple as:

`sudo apt install python3-prometheus-client`

The idea being that the exporter would:

* Run a HTTP server on a port so that Prometheus can scrape the metric(s)
* Subscribe to the multicast group and listen for the UDP packets
* Update the metric(s) with the current temperature

The full code can be seen in the [Github repository](https://github.com/karldyson/prometheus_temperature_exporter).

Whilst the code only processes the UDP packets containing packets containing temperature updates, it does log the status updates as well, and so the log file for the running service can be used to debug the current state of the IOT Plotter API calls too.

It'd be fairly trivial to alter both the pico code and the Prometheus exporter to talk unicast one to the other, in the event that your network hampers multicast, for example. I may add this to the TODO list.

### Initialise the socket

First, we set up the socket and subscribe to the multicast group

```python
mbin = bytes(map(int, config.multicast_group.split("."))) + bytes(4)
sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM, socket.IPPROTO_UDP)
sock.bind((config.bind_to_ip, config.multicast_port))
sock.setsockopt(socket.IPPROTO_IP, socket.IP_ADD_MEMBERSHIP, mbin)
```

### Start the HTTP server

Then we start the HTTP server that Prometheus will talk to

```python
start_http_server(config.server_port)
print(f"HTTP Server Started on tcp/{config.server_port}, listening for multicast on {config.multicast_group}:{config.multicast_port}")
```

### Initialise the metrics

Then initialise the metrics to be scraped

```python
temp_gauge = Gauge('temperature', "Temperatures", ['friendly_name', 'sensor_name'])
temp_gauge_timestamp = Gauge('temperature_last_seen_timestamp', "Temperatures Last Seen Timestamp", ['friendly_name', 'sensor_name'])
```

### Wait for packets

...before looping round handling the packets

```python,hl_lines=18
while True:
    """grab the packet from the network"""
    data, address = sock.recvfrom(1024)

    """get the current time and output the received packet for the log/console"""
    current_time = time.strftime("%Y-%m-%d %H:%M:%S", time.localtime())
    print(f"{current_time} {address}: {data.decode()}")

    """decode the packet and grab the sensor data from the json"""
    sensor = json.loads(data.decode())

    """we are only interested if the packet type is temperature data"""
    if sensor['type'] != 'temperature': continue

    """work out some details, the sensor name, make it a friendly name, grab the temperature"""
    sensor_name = sensor["name"]
    stripped_name = "".join(sensor_name.split())
    friendly_name = " ".join(re.sub(r"([A-Z])", r" \1", stripped_name).split())
    sensor_temperature = sensor["temperature"]

    """...and finally update the metrics"""
    temp_gauge.labels(friendly_name=friendly_name, sensor_name=stripped_name).set(sensor_temperature)
    temp_gauge_timestamp.labels(friendly_name=friendly_name, sensor_name=stripped_name).set(time.time())
```



### Friendlier Sensor Names

Each temperature update includes the sensor name from the configuration file on the pico. I had ensured that all of the names were formed with a capital for each new word, so <kbd>Loft</kbd>, <kbd>LivingRoom</kbd>, <kbd>ColdWaterTank</kbd> etc.

In the exporter, I convert these into "friendly names" that are included in the updates in a friendly_name label, so the above become Loft, "Living Room", "Cold Water Tank".

You can see the line that acheives this in the highlighted line of code above.

## Prometheus

Configuring Prometheus is the usual addition of a job to scrape the target. I'm running Debian, so my config file is in `/etc/prometheus/prometheus.yml` and I added the following:

```yaml
- job_name: 'temperature'
  static_configs:
    - targets: ['192.0.2.100:8000']
```

## Grafana

In Grafana I created a new dashboard, and a variable picker. This picker uses the `friendly_name` field introduced above:

<figure>
{{ image(url="grafana-friendly-name-variable.jpg") }}
<figcaption>Screenshot from Grafana for friendly_name variable definition</figcaption>
</figure>

The main graph then uses this, defaulting to "All".

In the following example you can see:

* when the heating comes on, whether it's feeding the hot water, radiators or both
* the temperature in the loft, and the water in the cold water tank

<figure>
{{ image(url="temperature-graph.jpg") }}
<figcaption>Screenshot from Grafana of temperature graphs</figcaption>
</figure>

### Alerts

I created an alert in Grafana that will tell me if the water in the cold water tank gets to 3&deg; celsius so I know if it's approaching freezing (although its a reasonable volume so won't freeze quickly)

I also wanted to be alerted as to whether updates had stopped arriving for a sensor.

I looked at a combination of `last_over_time` and `changes`, but, particularly with the cold water tank, the temperature could be very stable (and not change in value) for hours on end.

So I decided to add a timestamp metric that is updated at the same time as the temperature. So, even if the temperature value doesn't change, the timestamp, which is the unixtime value at the time of the update, will always change.

That allowed me to create an alert for a sensor on each pico (bearing in mind that some picos have multiple sensors, I don't want to be alerted multiple times for each pico)

```
time() - last_over_time(
    timestamp(
        changes(temperature_last_seen_timestamp{sensor_name="ColdWaterTank"}[1m]) > 0
    )[1h:]
)
```

# Summary

A nice thing about the design is that if you configure and plug in a new sensor:

* It'll start multicasting temperature updates to the network.
* The prometheus exporter will pick the updates up automatically and include them for scraping.
* Prometheus will pick them up automatically the next time it scrapes.
* Finally, Grafana will see them automatically, and at least for the main graph on the dashboard, they'll appear automatically.

# Footnotes

[^1]: ok, this is actually a Raspberry Pi 5 running Raspbian, so it's the ARM64 (aarch64) version of Bookworm at the time of writing
