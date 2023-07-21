# zerofeed

Zerofeed solar power control shell script for OpenDTU / Tasmota / OpenWrt with Shelly 3EM

- https://github.com/tbnobody/OpenDTU
- https://github.com/tasmota
- https://openwrt.org/

## About

Zero feed script to make sure the solar modules are reducing their output
to (ideally) not push any energy into the public network.

Inspired by https://github.com/tbnobody/OpenDTU/blob/master/docs/Web-API.md
and
https://github.com/hartkopp/zerofeed/

Needs OpenDTU and the Tasmota smart meter IoT devices in your WLAN and is
intended to be executed on an OpenWrt router (install curl & jq packages).

USE AT YOUR OWN RISK! Especially I don't know if the inverter is fit for
this purpose as all this functionality was reverse engineered by the
fabulous OpenDTU developers. I simply rely on their amazing work here.

## Code

The script `zerofeed-dbg.sh` contains extensive debug output which was used
during the development. The script `zerofeed.sh` should be completely silent
to not bloat any logfiles. `zerofeed-dbg.sh` is the main source code.

Create the 'release' script file without debug output with:

`grep -v echo zerofeed-dbg.sh > zerofeed.sh`

## Configuration

The script has several options to be adapted to your environment:

- SmartMeter IP (Tasmota) (update for your local network setup)<br />
`SMIP=192.168.60.7`

- SmartMeter user (Tasmota)<br />
`SMUSER=admin`

- SmartMeter password (Tasmota)<br />
`SMPASS=password`

- DTU IP (OpenDTU) (update for your local network setup)<br />
`DTUIP=192.168.60.5`

- DTU default admin user access (from OpenDTU installation)<br />
`DTUUSER="admin:openDTU42"`

- DTU serial number (insert your inverter SN here)<br />
`DTUSN=110123456789`

- SmartMeter type adaption:

The code is tested with a Shelly 3EM smart meter<br />
which provides the current house/flat power consumption (in Watt) with a
Shelly 3EM specific Web-API answer that has to be adapted in the `getSMPWR()`
function when a different smart meter is used!

In the current Shelly 3EM environment the full `jq` output looks like this:

```
$ curl -s "http://192.168.60.7/cm?user=admin&password=password&cmnd=status%208" | jd
{
  "StatusSNS": {
    "Time": "2023-07-14T11:03:31",
    "ENERGY": {
      "TotalStartTime": "2022-12-11T15:29:13",
      "Total": 2275.777,
      "Yesterday": 7.685,
      "Today": 2.477,
      "ExportActive": [
        12.617,
        0,
        0
      ],
      "Power": [
        -393,
        179,
        9
      ],
      "ApparentPower": [
        433,
        283,
        29
      ],
      "ReactivePower": [
        180,
        220,
        27
      ],
      "Factor": [
        -0.91,
        0.63,
        0.3
      ],
      "Frequency": [
        50,
        50,
        50
      ],
      "Voltage": [
        238,
        235,
        235
      ],
      "Current": [
        1.804,
        1.196,
        0.121
      ],
      "CurrentNeutral": 0.008
    }
  }
}
```

Which makes the code in `getSMPWR()` produce the power consumption value:

```
$ curl -s "http://192.168.60.7/cm?user=admin&password=password&cmnd=status%208" | jq '.StatusSNS.ENERGY.Power[0]'
509
```

Please check your smart meter with the full `jq` output and find the needed
power consumption in the JSON tree to get the correct string for your smart
meter for the power value (analogue to the '.StatusSNS.ENERGY.Power_curr'
example) - and update this string in `getSMPWR()` for your local setup.
I my setup I have to sum all three phases and save them to SMPWR.

## Installation

The script can be executed without any installation on any Linux system,
e.g. a Linux laptop, Raspberry Pi, OpenWrt or even on WSL2. It might also
run on Windows (when `bash`, `curl` and `jq` are installed), but this was
never tested. To reduce the power consumption and hardware effort the script
is intended to run on an OpenWrt router, which is powered anyway and can
handle this task easily.

To run and automatically start the `zerofeed.sh` script on your OpenWrt
router a start/stop script `zerofeed` for OpenWrt 22.03 has been created
which can be found in the `openwrt` directory of this repository.

The installation process:

1. Copy the `zerofeed` script to `/etc/init.d/zerofeed` on the router
2. Make sure the script is executable: `chmod 755 /etc/init.d/zerofeed`
3. Copy `zerofeed.sh` script to `/usr/bin/zerofeed.sh` on the router
4. Make sure the script is executable: `chmod 755 /usr/bin/zerofeed.sh`
5. Enable the script with: `/etc/init.d/zerofeed enable`

If you want to see the latest state in `/var/run/zerofeed.state` rename
the two '#cho' statements in the `zerofeed.sh` script to 'echo'.

E.g. with

`sed -i 's/\#cho/echo/g' /usr/bin/zerofeed.sh`

To check if everything works as expected start the zerofeed process with

`/etc/init.d/zerofeed start`

Check with `ps` whether the script is running:

```
root@OpenWrt:~# ps | grep zerofeed
 5175 root      1324 S    {zerofeed.sh} /bin/sh /usr/bin/zerofeed.sh
 5410 root      1308 S    grep zerofeed
```

The process id and the latest state can be seen in `/var/run`:

```
root@OpenWrt:~# ls -l /var/run/zerofeed.*
-rw-r--r--    1 root  root       5 Nov 29 22:41 /var/run/zerofeed.pid
-rw-r--r--    1 root  root      35 Nov 30 08:20 /var/run/zerofeed.state
root@OpenWrt:~# cat /var/run/zerofeed.pid
5175
root@OpenWrt:~# cat /var/run/zerofeed.state
30.11.22,08:22:19,0,302,0,,0,10,50
```

The meaning of the `zerofeed.state` content can be seen in the code.

## Adaption

Additionally there are some values to tweak the power control process:

- reduce safety margin from inverter by increasing this value<br />
`ABSLIMITOFFSET=0`

- threshold to trigger the SOLABSLIMIT decrease<br />
`SMPWRTHRESMIN=10`

- threshold to trigger the SOLABSLIMIT increase<br />
`SMPWRTHRESMAX=50`

- the minimum solar power (Watt) before starting the power control is now<br />
calculated based on the provided `DTUMAXPWR` value from the inverter:<br />
`SOLMINPWR = (3% from DTUMAXPWR) + 10 Watt`

These values are estimations and work fine in my environment.

## Power control functionality

The following pictures were rendered (with LibreOffice Calc) from debug
values during the development phase for understanding the general concept
of the zerofeed script. The adapted values and the algorithm have been
improved after getting the raw values for these figures.

* `SOLPWR` = blue (solar power yield)
* `SMPWR` = red (smart meter power consumption from house/flat)
* `SOLLASTLIMIT` = yellow (solar limit)

### Power control example (more solar power than consumption)

This picture shows a day with good solar power (SOLPWR) yield which has to be
limited to the house/flat power consumption (SMPWR). E.g. from 13:00 - 15:00
the solar panels are limited. At 15:30 a relevant power consumer is attached
with sets the SOLLASTLIMT value to DTUMAXPWR (disabled limit).

<img src="https://github.com/hartkopp/zerofeed/blob/main/img/high-solar-yield.png" width="800">

### Power control example (low solar power mostly unlimited)

This picture shows a day with low solar power (SOLPWR) yield where the
house/flat power consumption (SMPWR) is mostly higher than the solar yield
and therefore the SOLLASTLIMT value is set to DTUMAXPWR (disabled limit) for
most of the time. From 15:00 to 15:30 the solar power exceeds the SMPWR value
and the limiter gets enabled to not feed the public network.

<img src="https://github.com/hartkopp/zerofeed/blob/main/img/low-solar-yield.png" width="800">

## Power control algorithm

### Inverter startup phase

The script first waits for `SOLMINPWR` Watts before it starts to control
the solar panel power output. This makes sure that the inverter is working
and can process requests to the power limiter. The OpenDTU always answers
requests to get the current solar power - but in the case the solar power is
zero the inverter is completely shut down and not accessible!<br />
When the inverter switches off and the `SOLPWR` value becomes zero (e.g. over
night) the script gets back to this startup phase.

### Initializing/disabling the solar power limiter

When the inverter is up and running the limit is disabled (set to 100%).
Usually the persistent power limit value is always 100% at power on time until
someone modified this value (with persistence). This script only applies
non-persistent power limitations to the inverter.

### The power control loop

The 'system power' is measured by the Tasmota smart meter interface and
provides the value of the power flow from the public network to your
house/flat (where the solar modules are connected too). This value `SMPWR`
is usually a positive value which indicates the current power consumption
of your house/flat. When the solar modules produce more energy than your
house/flat consumes the `SMPWR` can become a negative value as the power
meter measures your feeding into the public network - which we want to
avoid with this script.

We mainly have two triggers to start a power limit control action to maintain
a low power consumption between `SMPWRTHRESMIN` and `SMPWRTHRESMAX`:<br />

1. `SMPWR` is less than `SMPWRTHRESMIN`:<br />
Concept: Set the power limit for the solar panels to `SMPWR` + `SOLPWR`.
As `SMPWR` was assumed to be negative the calculated limit is less than the
current solar power output `SOLPWR` and should lead to a `SMPWR` value greater
then zero. This action is now triggered when `SMPWR` is less than
`SMPWRTHRESMIN`, therefore the calulated limit results from:
`SMPWR + SOLPWR - SMPWRTHRESMIN`. As the inverter might be conservative and
too cautious with the limit the `ABSLIMITOFFSET` value was introduced to
increase the calculated limit. The `ABSLIMITOFFSET` probably needs to be
adjusted in your setup.

2. `SMPWR` is greater than `SMPWRTHRESMAX`:<br />
The `SMPWRTHRES*` threshold values are used to reduce the power limit control
commands to the OpenDTU and the inverter. In the best case the `SMPWR` value
remains between `SMPWRTHRESMIN` and `SMPWRTHRESMAX`. When `SMPWR` gets
greater than `SMPWRTHRESMAX` the solar power limit can be safely increased by
`SMPWRTHRESMAX`. Depending on `SMPWR` we can increase the solar power limit
faster to finally remove the solar power limit. When the solar power limit
has been increased up to 100% it remains there until `SMPWR` becomes less than
`SMPWRTHRESMIN` again to restart the power limit control process.
