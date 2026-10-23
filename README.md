# WeeWX Interceptor

Receive weather observations from network bridges and consoles by listening for
HTTP uploads or capturing their existing HTTP traffic. The driver runs locally
and supplies LOOP packets to WeeWX.

## Supported devices

Choose the `device_type` that matches the device and its upload protocol.

| Device or protocol | `device_type` | Connection |
|---|---|---|
| Acurite Internet Bridge and SmartHub | `acurite-bridge` | Legacy HTTP uploads; network redirection may be needed |
| Acurite Access, Ambient consoles and other Weather Underground clients | `wu-client` | Weather Underground HTTP uploads |
| Fine Offset GW1000 and other Ecowitt upload clients | `ecowitt-client` | Ecowitt HTTP uploads |
| ObserverIP, Fine Offset HP1000 and WH2600 | `observer` | Observer HTTP uploads |
| Oregon Scientific LW301 and LW302 | `lw30x` | HTTP uploads redirected to the local receiver |
| LaCrosse GW1000U | `lacrosse-bridge` | Local HTTP listener configured as the gateway proxy |

Firmware determines the available upload settings. A device with a custom-server
setting can send directly to the listener. Other bridges need DNS redirection,
a proxy or a network interface that can capture their traffic. See the
[device and network reference](doc/reference.md) for those setups.

## Install

The catalog package is version **1.0.2**, for WeeWX `>=5.5,<6` and Python
`>=3.10,<3.15`. Listening for uploads needs no additional Python packages beyond
WeeWX. Packet sniffing has [additional dependencies](#capture-existing-http-traffic).

Use the `weectl` command from the environment that runs WeeWX. Replace
`/path/to/weewx.conf` with your station configuration before running:

```sh
weectl extension install https://github.com/weewx-packages/catalog/releases/download/interceptor/v1.0.2/interceptor-1.0.2.tar.gz --config /path/to/weewx.conf
weectl station reconfigure --driver=user.interceptor --config /path/to/weewx.conf
```

Select the matching device type when prompted. These are stock WeeWX commands;
no repository client is required.

## Listen for uploads

For a gateway using the Ecowitt custom-server protocol, configure the following
sections in `weewx.conf`:

```ini
[Station]
    station_type = Interceptor

[Interceptor]
    driver = user.interceptor
    device_type = ecowitt-client
    mode = listen
    address = 0.0.0.0
    port = 8000
```

Set the gateway's custom server to the WeeWX computer's LAN address and port
`8000`, using the Ecowitt protocol. Allow that connection through the host's
firewall. Restart WeeWX after saving the configuration. On a systemd installation:

```sh
sudo systemctl restart weewx
```

For Weather Underground uploads, use `device_type = wu-client` and select that
protocol in the sending device. Use the device table for other protocols.
Port `8000` avoids the elevated privileges usually required for port `80`.

## Map additional sensors

The driver supplies a default sensor map for each protocol. Use
`[[sensor_map_extensions]]` to change individual mappings while retaining the
other defaults. For an Ecowitt client:

```ini
[Interceptor]
    driver = user.interceptor
    device_type = ecowitt-client
    mode = listen
    port = 8000
    [[sensor_map_extensions]]
        inTemp = temperature_1
        inHumidity = humidity_1
        outTemp = temperature_4
        outHumidity = humidity_4
```

A complete `[[sensor_map]]` replaces the default map. Source field names vary
by protocol; use the diagnostic output and the
[mapping examples](doc/reference.md#driver-options) to identify them. Additional
observations need suitable archive schema columns if you want to store them.

## Diagnose incoming data

Stop WeeWX before starting a diagnostic listener on the same port. Use its
Python environment and replace `/path/to/station/bin` with the installed
station's `bin` directory:

```sh
PYTHONPATH=/path/to/station/bin python -m user.interceptor --device=ecowitt-client --mode=listen --port=8000 --debug
```

The diagnostic prints incoming data and parsed sensor values. It also supports
`--help`, `--data` for parser input, and packet-capture options.

## Capture existing HTTP traffic

Sniff mode needs system `libpcap` and a compatible Python `pcap` binding in the
WeeWX environment. The driver supports `pypcap` and `pylibpcap`. Building a
binding from source also requires a compiler, libpcap development headers and
the headers for that Python interpreter. Follow the
[pypcap installation instructions](https://github.com/pynetwork/pypcap#installation)
for your platform before enabling sniff mode.

The capture interface must see the station's traffic and have packet-capture
permission. This mode parses HTTP traffic, not encrypted HTTPS. LaCrosse
GW1000U supports listen mode only. See the
[network examples](doc/reference.md) for interface, filter, DNS and proxy setup.

## Source and license

This repository mirrors [tfilo/weewx-interceptor](https://github.com/tfilo/weewx-interceptor),
which continues [Matthew Wall's original driver](https://github.com/matthewwall/weewx-interceptor).
The [`master` branch](https://github.com/weewx-packages/weewx-interceptor/tree/master)
follows upstream; this documentation branch adds current setup instructions.
The original copyright notices and [GPLv3 license](license) remain in the source.
Compatibility checks use protocol fixtures; they do not replace testing with
your bridge and firmware.

Copyright 2016-2020 Matthew Wall. Distributed under the terms of the GPLv3.
