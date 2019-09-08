# homebridge-rpi
[![npm](https://img.shields.io/npm/dt/homebridge-rpi.svg)](https://www.npmjs.com/package/homebridge-rpi) [![npm](https://img.shields.io/npm/v/homebridge-rpi.svg)](https://www.npmjs.com/package/homebridge-rpi)
[![JavaScript Style Guide](https://img.shields.io/badge/code_style-standard-brightgreen.svg)](https://standardjs.com)

## Homebridge plugin for Raspberry Pi
Copyright © 2019 Erik Baauw. All rights reserved.

This [Homebridge](https://github.com/nfarina/homebridge) plugin exposes a
Raspberry Pi to HomeKit.
It provides the following features:

- Monitoring from HomeKit the Pi's CPU: temperature, frequency, voltage,
and throttling, incl. [Eve](https://www.evehome.com/en/eve-app) history for
the temperature;
- Monitoring and controlling from HomeKit the Pi's GPIO pins;
- Monitoring and controlling from HomeKit a
[Fan SHIM](https://shop.pimoroni.com/products/fan-shim) installed in the Pi.

Unlike most other Raspberry Pi plugins, homebridge-rpi runs on any regular
Homebridge setup, connecting to the Pi's `pigpiod` daemon over the network.
In particular, homebridge-rpi:
- Exposes multiple Raspberry Pi computers from one Homebridge instance;
- Does _not_ need to run on a Raspberry Pi;
- Does _not_ require any C components;
- Does _not_ require `root` privilege.

### Prerequisites
You need a server to run Homebridge.
This can be anything running [Node.js](https://nodejs.org): a Raspberri Pi,
a NAS system, an always-on PC running Linux, macOS, or Windows, or even a
Docker container.
See the [homebridge Wiki](https://github.com/nfarina/homebridge/wiki) for
details.
I use a Raspberri Pi 3B+, and, occasionally a Mac mini server.
Note that I develop and test against the latest LTS release of Node.js.

The homebridge-rpi plugin connects (locally or remotely) to the
[`pigpiod`](http://abyz.me.uk/rpi/pigpio/pigpiod.html) daemon
on the Raspberry Pi.
It uses the [Socket Interface](http://abyz.me.uk/rpi/pigpio/sif.html),
just as the [`pigs`](http://abyz.me.uk/rpi/pigpio/pigs.html) command.
This daemon is part of the [`pigpio`](https://github.com/joan2937/pigpio)
library, which is included in Raspbian.
While this daemon comes with Raspbian, it needs to be enabled and
configured for use by homebridge-rpi, see [**Installation**](#installation).
<br>If you run Homebridge in a container on the Raspberry Pi, let
homebridge-rpi connect to `pigpiod` running on the host.
Do _not_ try to run `pigpiod` in the container.

To interact with HomeKit, you need Siri or a HomeKit app on an iPhone,
Apple Watch, iPad, iPod Touch, or Apple TV (4th generation or later).
I recommend to use the latest (non-beta) version of the OS.
<br>Please note that Siri and even Apple's
[Home](https://support.apple.com/en-us/HT204893) app still provide only limited
HomeKit support.
To use the full features of homebridge-rpi, you might want to check out some
other HomeKit apps,
like [Eve](https://www.evehome.com/en/eve-app) (free) or
[Home 3](https://hochgatterer.me/home/) (paid).
<br>To interact with HomeKit remotely and for HomeKit automations, you need to
setup an Apple TV (4th generation or later), HomePod, or iPad as
[home hub](https://support.apple.com/en-us/HT207057).

### Installation
To install homebridge-rpi, issue:
```
$ sudo npm -g i homebridge-rpi
```
on the server or container running Homebridge.

Note that you need to execute these steps on each of the Raspberry Pi
computers you want homebridge-rpi to expose.

#### Configure `pigpiod` Service
Raspbian comes with a service definition for `pigpiod`, in
`/lib/systemd/system/pigpiod.service`:
```
[Unit]
Description=Daemon required to control GPIO pins via pigpio
[Service]
ExecStart=/usr/bin/pigpiod -l
ExecStop=/bin/systemctl kill pigpiod
Type=forking
[Install]
WantedBy=multi-user.target
```
By default `pigpiod` won't accept remote connections, due to the `-l` option.
To enable remote connections, edit `pigpiod.service` and remove the `-l` option
from `ExecStart`:
```
ExecStart=/usr/bin/pigpiod
```
If you want, you can limit remote connections to the service running
homebridge-rpi by:
```
ExecStart=/usr/bin/pigpiod -n xx.xx.xx.xx
```
substituting `xx.xx.xx.xx` with the IP address of the server running Homebridge.

#### Enable `pigpiod` Service
To enable and start `pigpiod` as a service, issue:
```
$ sudo systemctl enable pigpiod
$ sudo systemctl start pigpiod
```
To check that the service is running, issue:
```
$ pigs hwver
10494163
```
This returns the Pi's hardware revision (in decimal).

To check that the service is accepting remote connections, run `pigs` on
another Raspberry Pi:
```
$ PIGPIO_ADDR=xx.xx.xx.xx pigs hwver
10494163
```
substituting `xx.xx.xx.xx` with the IP address of the remote Raspberry Pi.

#### Install `vcgencmd` Script
`pigpio` provides a hook to execute a shell command remotely.
homebridge-rpi uses this hook to run a little shell script,
[`vcgencmd`](./opt/pigpio/cgi/vcgencmd), that calls `vcgencmd` to get the
Pi's CPU temperature, frequency, voltage, and throttling information.
This script needs to be installed to `/opt/pigio/cgi` by:
```
$ sudo cp vcgencmd /opt/pigpio/cgi
$ chmod 755 /opt/pigpio/cgi/vcgencmd.sh
```
To check that the script has been installed correctly, issue:
```
$ pigs shell vcgencmd.sh
0
```
The return status `0` indicates success.
This should have created two output files in `/opt/pigpio`:
```
$ ls -l /opt/pigpio
total 12
-rw-r--r-- 1 root root  152 Aug 19 11:51 access
drwxr-xr-x 2 root root 4096 Aug 19 11:53 cgi
-rw-r--r-- 1 root root    0 Aug 19 11:53 vcgencmd.err
-rw-r--r-- 1 root root  114 Aug 19 11:53 vcgencmd.out
```
The `.err` file should be empty.
The `.out` file contains the script's output in JSON:
```
$ json vcgencmd.out
{
  "date": "2019-08-19T09:53:54+00:00",
  "load": 0.23,
  "temp": 42.9,
  "freq": 1400000000,
  "volt": 1.3688,
  "throttled": "0x80000"
}
```

#### File Access
`pigpio` provides a hook to access files remotely.
homebridge-rpi uses this hook to get the Raspberry Pi's serial number from
`/proc/cpuinfo` and to get the output from the `vcgencmd` script.
These files need to be whitelisted, in `/opt/pigpio/access`:
```
$ sudo sh -c 'cat - > /opt/pigpio/access' <<+
/proc/cpuinfo r
/opt/pigpio/vcgencmd.out r
+
```

To check that the files can be read, issue:
```
$ pigs fo /opt/pigpio/vcgencmd.out 1
0
$ pigs -a fr 0 1024
114 {"date":"2019-08-19T09:53:54+00:00","load":0.23,"temp":42.9,"freq":1400000000,"volt":1.3688,"throttled":"0x80000"}
$ pigs fc 0
```
The `fo` command opens the file for reading, returning a file descriptor, `0`
in this example.
This file descriptor is passed to the `fr` and `fc` commands.
The `fr` commands reads up to 1024 bytes from the file,
and prints them as ascii.
The `fc` command closes the file.

#### Automatic Discovery