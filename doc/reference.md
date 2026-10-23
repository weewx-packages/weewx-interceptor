# Device and network reference

Start with the [installation and basic listener setup](../README.md). These
examples cover gateways that need network redirection and protocol-specific
sensor maps. Replace sample addresses and interface names with your own.

Files mentioned under `util/` are source examples, not automatically installed
configuration. Download or clone this repository before using those files.


## Driver options

To configure the driver beyond the default values, set parameters in the
`[Interceptor]` section of the WeeWX configuration file.

For example, to listen on port 8000 instead of the default port 80:

```ini
[Interceptor]
    driver = user.interceptor
    device_type = observer
    port = 8000
```

To provide the hardware name of your weather station:

```ini
[Interceptor]
    driver = user.interceptor
    device_type = observer
    hardware_name = WS-2902A
    port = 8000
```

To listen on port 8080 on the network interface with address 192.168.0.14:

```ini
[Interceptor]
    driver = user.interceptor
    device_type = lw30x
    port = 8080
    address = 192.168.0.14
```

To sniff packets from 192.168.0.14 on network interface eth1:

```ini
[Interceptor]
    driver = user.interceptor
    device_type = acurite-bridge
    mode = sniff
    iface = eth1
    pcap_filter = src 192.168.0.14 and dst port 80
```

For hardware such as the Acurite bridge or LW30x that support multiple remote
sensors, use the sensor_map to map the sensors to database fields.  For
example:

```ini
[Interceptor]
    driver = user.interceptor
    device_type = acurite-bridge
    ...
    [[sensor_map]]
        inTemp = temperature.06022.*      # sensor id 06022
        outTemp = temperature.05961.*     # sensor id 05961
```

```ini
[Interceptor]
    driver = user.interceptor
    device_type = lw30x
    ...
    [[sensor_map]]
        inTemp = ot.2:*.*      # sensor on channel 2
        outTemp = ot.1:*.*     # sensor on channel 1
```

For hardware such as the GW1000 that use the ecowitt-client protocol, use the
sensor_map_extensions to add/modify sensor mappings without having to specify
the entire mapping.  For example, this would use the readings from the first
temperature sensor as the indoor temperature, and the readings from the fourth
temperature sensor as the outdoor temperature.  All of the other sensors would
be mapped using the default sensor mapping for ecowitt-client.

```ini
[Interceptor]
    driver = user.interceptor
    device_type = ecowitt-client
    ...
    [[sensor_map_extensions]]
        inTemp = temperature_1
        inHumidity = humidity_1
        outTemp = temperature_4
        outHumidity = humidity_4
```


## How to run the driver directly

Use the installed module from the WeeWX Python environment. Stop WeeWX first
if it uses the same listening port. Replace `/path/to/station/bin` with the
installed station directory.

```sh
PYTHONPATH=/path/to/station/bin python -m user.interceptor --device=acurite-bridge --mode=sniff --iface=eth0 --debug
```

Sniff mode needs the [packet-capture dependencies](../README.md#capture-existing-http-traffic).
Parser-only diagnostics accept `--data`; use `--help` for all options.


## Example: Ecowitt GW1000 in listen mode on port 8000

1) Using the WSView app, go to the 'customized' page and configure the GW1000
   to send to the weewx computer on port 8000 using the ecowitt protocol

2) Configure the interceptor

```ini
[Interceptor]
    driver = user.interceptor
    device_type = ecowitt-client
    port = 8000
```


## Example: Acurite Bridge in listen mode, traffic redirected by DNS hijack

1) Configure the DNS server for the local network.  Make a DNS entry for one
   (or both) of these names so that they map to the IP address of the computer
   running the interceptor driver:

     hubapi.myacurite.com   # firmware 126 uses this name
     www.acu-link.com       # firmware 224 uses this name

2) Configure the interceptor

```ini
[Interceptor]
    driver = user.interceptor
    device_type = acurite-bridge
    port = 80
```


## Example: LW302 in listen mode, traffic redirected by DNS hijack

1) Configure the DNS server for the local network.  Make a DNS entry for this
   name so that it maps to the IP address of the computer running the
   interceptor driver:

     gateway.weather.oregonscientific.com

2) Configure the interceptor

```ini
[Interceptor]
    driver = user.interceptor
    device_type = lw30x
```


## Example: Observer in listen mode

1) Configure the Observer to send data to the computer on which interceptor
   is running.

2) Set parameters in the weewx configuration file:

```ini
[Interceptor]
    driver = user.interceptor
    device_type = observer
```


## Example: GW1000U

1) Use the GAS utility to set the proxy on the gateway to the computer on which
   interceptor is running.

2) Optionally set parameters in the weewx configuration file:

```ini
[Interceptor]
    driver = user.interceptor
    device_type = lacrosse-bridge
    serial = 7FFFXXXXXXXXXXXX # 16-character registration key
    sensor_interval = 1 # numer of minutes between sensor updates
    history_interval = 3 # integer value in [0-7]
    ping_interval = 120 # number of seconds between gateway pings
```


## Example: sniff mode for any WU client

This example assumes two network interfaces on the computer running weewx.

1) Plug the computer into the local network using the first interface (eth0).
   Plug the device into the second interface (eth1).
   Bridge the two network interfaces.
   If you run any web services on the computer, configure them to listen on
   interface eth0, not eth1.

2) Set parameters in the weewx configuration file.  The device has IP address
   of X.X.X.X

```ini
[Interceptor]
    driver = user.interceptor
    device_type = wu-client
    mode = sniff
    iface = eth1
    pcap_filter = src X.X.X.X and dst port 80
```


## Example: Acurite Bridge in sniff mode

As of Feb2019, sniffing traffic bound for acurite servers no longer works,
   since acurite no longer runs the aculink or myacurite servers.

1) The interceptor must be running on a computer that can see traffic from the
   acurite bridge.  For example, run weewx on a raspberry pi that has both a
   wired network interface and a wifi network interface.  Connect the acurite
   bridge to the wired interface, and connect the rpi wireless to the local
   wireless network.  Configure the rpi to bridge or route traffic between
   wired and wireless interfaces.
    
   Using iptables, routing would look something like this:

```sh
sudo iptables -t nat -A POSTROUTING -o wlan0 -j MASQUERADE
sudo iptables -A FORWARD -i wlan0 -o eth0 -m state --state RELATED,ESTABLISHED -j ACCEPT
sudo iptables -A FORWARD -i eth0 -o wlan0 -j ACCEPT
sudo sh -c "echo 1 > /proc/sys/net/ipv4/ip_forward"
```

   If the acurite bridge is the only device on the wired interface, you must
   also run a DHCP server on the rpi that will give the acurite bridge a valid
   IP address.

2) Configure the interceptor

```ini
[Interceptor]
    driver = user.interceptor
    device_type = acurite-bridge
    mode = sniff
    iface = eth0
    pcap_filter = src X.X.X.X and dst port 80
```

  Where X.X.X.X is the IP address of the acurite bridge on the wired interface.


## Example: LW302 in sniff mode

As of 2018, sniffing traffic bound for oregon scientific servers no longer
   works, since oregon scientific shut down their servers.

1) Plug LW302 into first network interface eth0 on computer running weewx.
   Bridge the two network interfaces eth0 and eth1 on the computer running
   weewx.

2) Set parameters in the weewx configuration file.  The LW302 has IP address
   of X.X.X.X

```ini
[Interceptor]
    driver = user.interceptor
    device_type = lw30x
    mode = sniff
    iface = eth0
    pcap_filter = src X.X.X.X and dst port 80
```


## Example: Acurite Bridge in listen mode, traffic sniffed at router

1) Create a script that captures data from the bridge and sends it to the
   computer on which weewx runs.  The router address is Z.Z.Z.Z, the acurite
   bridge address is X.X.X.X, the computer running the interceptor is Y.Y.Y.Y.

#!/bin/sh
# remote to the router, capture traffic, and send to interceptor
```sh
ssh Z.Z.Z.Z "tcpdump -U -w - -i vr1 src X.X.X.X and port 80" | combine-lines.pl | xargs -n 1 curl http://Y.Y.Y.Y:9999 -s -d
```

2) Configure the interceptor

```ini
[Interceptor]
    driver = user.interceptor
    device_type = acurite-bridge
    port = 9999
```


## Reference: Generic configurations

Here are some example configurations, from simplest to most complicated.

Example 1: weewx is running on host 'pi' and nothing is listening on port 80.
Configure the driver to listen on port 80 and configure the internet bridge to
send to the host 'pi' instead of the cloud.

Example 2: weewx is running on host 'pi' with two bridged network interfaces,
eth0 and eth1.  Plug the internet bridge into one interface and the network
into the other interface.  Configure the driver to sniff traffic from the
internet bridge.

Example 3: weewx is running on host 'pi', nothing is listening on port 80, and
the internet bridge cannot be configured.  Configure the driver to listen on
port 80.  Add a DNS entry so that traffic from the internet bridge is sent to
'pi' instead of the cloud.

Example 4: weewx is running on host 'pi', nothing is listening on port 80, and
the internet bridge cannot be configured.  Configure the driver to listen on
port 80.  Configure the router to redirect traffic from the internet bridge
and send it to 'pi' instead of the cloud.

Example 5: weewx is running on host 'pi', which has a web server on port 80
to display weewx reports.  Configure the driver to listen on port 9999.  Add a
reverse proxy to the web server configuration to direct traffic on port 80 from
the device to port 9999.  Add a DNS entry so that traffic from the device is
sent to 'pi' instead of the cloud.


## Reference: How to intercept data

These are strategies for getting data to the driver when the simple approach
(direct from device to driver) is not possible:

  1) Hijack DNS
     use a local DNS entry to make the internet bridge send directly to weewx
     internet_bridge ---> driver ( ---> web_service )

  2) HTTP proxy/redirect
     configure the internet bridge to send to an HTTP proxy that you control
     internet_bridge ---> proxy ---> driver ( ---> web_service )

  3) Packet capture
     listen to traffic on the network and capture anything from the bridge
     internet_bridge ---> tap ( ---> web_service )
                           \-> driver ( ---> web_service )

Which one you choose depends on your network configuration, network hardware,
and your ability to add and configure devices on the network.

In the examples that follow,
  X.X.X.X is the address of the internet bridge
  Y.Y.Y.Y is the address of the computer on which weewx is running
  PPPP is the port on which the driver is listening


## 1) Hijack DNS

Change the DNS entry so that the internet bridge device sends directly to the
driver.  If you control DNS on the network, you can make the internet bridge
send to the driver by creating a DNS entry for the host to which the internet
bridge tries to send its data.

The examples that follow use hubapi.myacurite.com.  For acurite devices with
firmware 224 you should use www.acu-link.com.  For LW30x devices use
gateway.weather.oregonscientific.com.

1a) If you run pfsense, simply add an entry in the Services -> DNS forwarder

  host: hubapi
  domain: myacurite.com
  ip address: Y.Y.Y.Y

OpenWrt has a similar configuration.

1b) If you run your own nameserver, add an entry to your DNS configuration.
For example, a bind9 configuration looks something like this:

In the file /etc/bind/named.conf.local:

```conf
zone "hubapi.myacurite.com" {
    type master;
    file "/etc/bind/myacurite.com";
};
```

In the file /etc/bind/myacurite.com:

```conf
$TTL    604800
@       IN      SOA     dns.hubapi.myacurite.com. root.hubapi.myacurite.com. (
2016032001 ; Serial
    604800 ; Refresh
     86400 ; Retry
   2419200 ; Expire
    604800 ) ; Negative Cache TTL
;
@       IN      NS      dns.hubapi.myacurite.com.
@       IN      A       Y.Y.Y.Y
*       IN      A       Y.Y.Y.Y
*       IN      AAAA    ::1
```

This will redirect any requests to hubapi.myacurite.com, but it will not
redirect any requests to myacurite.com.

1c) Alternative bind configuration using DNS views.  This configuration sends
the hijacked DNS entry for queries from the weather station, but the standard
DNS entry for wunderground for every other DNS client query.  Thanks to Lee H-W
for this very simple DNS solution.

```conf
view "watson" {
  match-clients { 172.17.1.131/32; }; // weather station
  recursion yes;
  // view statements as required
  zone "rtupdate.wunderground.com" {
    type master;
    file "/etc/bind/override1.zone";
  };
};
```


## 2) HTTP proxy/redirect

Use a proxy to capture HTTP traffic and redirect it to the driver, and use
DNS hijack or direct configuration of the device to make the internet bridge
send to the web proxy.  Then configure the web server as a reverse proxy or
with a redirecting CGI script.

2a) Here is an example of an Apache 'reverse proxy' configuration for the
Acurite internet bridge.  The Apache server sends any requests from the
internet bridge to the driver.

In the file /etc/apache2/conf.d/myacurite.conf:

```conf
RewriteEngine on
RewriteCond %{HTTP_POST} www.myacurite.com
RewriteRule ^/weatherstation(.*)$ http://Y.Y.Y.Y/weatherstation$1
```

2b) Another option is to use an Apache CGI script.  Put a script alias in
the file /etc/apache2/conf.d/myacurite.conf:

```conf
ScriptAlias /weatherstation/updateweatherstation/ /usr/lib/cgi-bin/myacurite
```

and copy the cgi script util/usr/lib/cgi-bin/myacurite to the Apache
cgi-bin directory (nominally /usr/lib/cgi-bin).

2c) Here is a reverse proxy configuration for the nginx web server:

```conf
server {
    location /weatherstation/ {
        proxy_pass http://Y.Y.Y.Y:PPPP/;
    }
}
```


## 3) Packet capture configurations

There are many ways to capture traffic.  In each case, traffic is intercepted
then sent to the driver.  The capture and the driver might run on the same
device, or they can run on separate devices.  The traffic may also be sent on
to its original destination.

One strategy is to use a network capture tool such as tcpdump or ngrep to
capture traffic, then use a tool such as nc to direct the traffic to the
driver.  Another strategy is to use firewall rules to capture and redirect
traffic.

If the source sends each HTTP request in a single line (e.g., the LW30x or the
Observer), then tcpdump, ngrep, or tcpflow will probably work fine.  However,
if the source sends each request in multiple packets (e.g., Acurite bridge
with firmware after July 2016), you will have to use a utility such as
combine-lines.pl (in the util folder) before re-sending the request.

Here are examples of how to capture packets:

option 1: redirect traffic using iptables firewall rules

```sh
iptables -t broute -A BROUTING -p IPv4 --ip-protocol 6 --ip-destination-port 80 -j redirect --redirect-target ACCEPT
iptables -t nat -A PREROUTING -i br0 -p tcp --dport 80 -j REDIRECT --to-port PPPP
```


option 2: capture using tcpdump, redirect using nc

```sh
tcpdump -i eth0 src X.X.X.X and port 80 | nc Y.Y.Y.Y PPPP
tcpdump -i eth0 dst hubapi.myacurite.com and port 80 | nc Y.Y.Y.Y PPPP
```


option 3: capture using ngrep, redirect using nc

```sh
ngrep -l -q -d eth0 'ether src host X.X.X.X && dst port 80' | nc Y.Y.Y.Y PPPP
```


option 4: capture using tcpdump via a secure connection to the router

```sh
ssh Z.Z.Z.Z "tcpdump -U -w - -i vr1 src X.X.X.X and port 80" | nc localhost PPPP
```


option 5: capture using ngrep, filter with sed, forward with curl

```sh
ngrep -l -q -d eth0 'xxxxxxxxxxxx' | sed -u '/mac=/!d' | xargs -n 1 curl http://Y.Y.Y.Y:PPPP -s -d
```


option 6: use stdbuf and strings to extract fragments from tcpdump

```sh
tcpdump -Anpl -s0 -w - -i eth0 src X.X.X.X and dst port 80 | stdbuf -oL strings -n8 | combine-lines.pl | xargs -n 1 curl http://Y.Y.Y.Y:PPPP -s -d
```


option 7: capture using tcpflow

```sh
tcpflow -C -i eth0 -s0 tcp dst port 80 | combine-lines.pl | xargs -n 1 curl http://Y.Y.Y.Y:PPPP -s -d
```


## Here are configurations that use packet capture:

a) Firewall-on-Router

Use firewall rules to redirect traffic from the internet bridge to the driver.

b) Bridge

Configure a computer or other device with two network interfaces to physically
bridge between the internet bridge and the regular network.  Plug the internet
bridge into one port, and connect the other port into the local network.  The
bridge captures traffic and sends it to the driver.

c) Tap-on-Router

Capture traffic on the network's edge router.  This can be done with a script
that captures traffic from the internet bridge and sends it to the driver.

d) Tap-on-Hub

Configure a device connected to the same hub as the internet bridge.  Since
any device on a hub can see the network traffic from any other device on the
hub, simply listen for traffic from the internet bridge then send it to the
driver.

e) Tap-on-Switch

Configure a device that is connected to a managed switch.  Unmanaged switches
will not work.  Configure the switch to mirror the traffic from one port to a
second port.  Configure a device on the second port to capture traffic from
the internet bridge then send it to the driver.

f) Tap-on-wireless

Configure a device that is connected to the same wireless network as the
internet bridge, or on a wireless segment that is between the internet bridge
and the edge router.  This will probably only work on open, unencrypted
wireless networks.
