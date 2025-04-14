# How to synchronize two Raspberry Pi 5s using LinuxPTP

The Raspberry Pi 5 supports hardware timestamping of network packets transmitted and received by its Ethernet interface.
This timestamping is handled by the PTP Hardware Clock (PHC), a separate clock dedicated to the network interface.
This clock is exposed by the network driver to the Linux kernel, and shows up as `/dev/ptp0`.

PTP (Precision Time Protocol) is a standardized protocol for synchronizing PHCs over Ethernet.
LinuxPTP provides the `ptp4l` application to do this synchronization.

In addition to synchronizing PHCs, one can synchronize the system clock to the PHC or vice versa using the `phc2sys` utility. In this guide, we will take you through the steps of using LinuxPTP to synchronize the system clocks of two Raspberry Pi 5s.

```{figure} images/RPi5-ptp.svg
   :width: 100%
   :alt: Raspberry Pi 5 clock topology

   Raspberry Pi 5 clock topology
```

## Install LinuxPTP

LinuxPTP is available on Ubuntu as either a deb or a snap package.
You can use either package to work through this guide.

`````{tabs}
````{group-tab} deb

Install the deb package:

```bash
sudo apt install linuxptp
```

````
````{group-tab} snap

Install the snap:

```bash
sudo snap install linuxptp
```

Add [aliases](https://snapcraft.io/docs/commands-and-aliases) for the snap's apps:

```bash
sudo snap alias linuxptp.hwstamp-ctl hwstamp_ctl
sudo snap alias linuxptp.nsm nsm
sudo snap alias linuxptp.phc-ctl phc_ctl
sudo snap alias linuxptp.phc2sys phc2sys
sudo snap alias linuxptp.pmc pmc
sudo snap alias linuxptp.ptp4l ptp4l
sudo snap alias linuxptp.timemaster timemaster
sudo snap alias linuxptp.ts2phc ts2phc
sudo snap alias linuxptp.tz2alt tz2alt
```

````
`````

## Set up your network

LinuxPTP uses either layer 2 or layer 3 networking.
In this case, layer 3, IP networking is used to handle communication between the two Raspberry Pis which are connected directly to each other.
You can use a switch between the two devices, but only if it supports the PTP protocol.

```{note}
In the rest of this guide, the two devices are referred to as *Pi A* and *Pi B*.
```

Assign static IP addresses to the two Pis, so they can communicate using the direct link.
You can assign 10.0.0.1 to Pi A, and 10.0.0.2 to Pi B.

On Ubuntu Desktop, Ethernet is managed by Network Manager.
You'll need to remove the interface from Network Manager so you can manually specify the address. To do this, Run:

```bash
sudo nmcli dev set eth0 managed no
```

Assign each device a static IP and bring up the interfaces.

On Pi A:
```bash
sudo ip addr add 10.0.0.1/24 dev eth0
sudo ip link set dev eth0 up
```

On Pi B:
```bash
sudo ip addr add 10.0.0.2/24 dev eth0
sudo ip link set dev eth0 up
```

On each device, ping the other to make sure layer 3 IP networking is functional.

On Pi A:
```{terminal}
   :input: ping 10.0.0.2
   :user: ubuntu
   :host: pi-a
   :dir: ~

PING 10.0.0.2 (10.0.0.2) 56(84) bytes of data.
64 bytes from 10.0.0.2: icmp_seq=1 ttl=64 time=0.275 ms
64 bytes from 10.0.0.2: icmp_seq=2 ttl=64 time=0.224 ms
64 bytes from 10.0.0.2: icmp_seq=3 ttl=64 time=0.222 ms
^C
--- 10.0.0.2 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2046ms
rtt min/avg/max/mdev = 0.222/0.240/0.275/0.024 ms
```

On Pi B:
```{terminal}
   :input: ping 10.0.0.1
   :user: ubuntu
   :host: pi-b
   :dir: ~

PING 10.0.0.1 (10.0.0.1) 56(84) bytes of data.
64 bytes from 10.0.0.1: icmp_seq=1 ttl=64 time=0.235 ms
64 bytes from 10.0.0.1: icmp_seq=2 ttl=64 time=0.216 ms
64 bytes from 10.0.0.1: icmp_seq=3 ttl=64 time=0.225 ms
^C
--- 10.0.0.1 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2039ms
rtt min/avg/max/mdev = 0.216/0.225/0.235/0.007 ms
```

## Start PTP synchronization

PTP operates in a server-client architecture.
A server with one or more clients synchronizing against it, is called a time domain.
One Ethernet can have multiple time domains.
If the time domain is not specified, LinuxPTP will default to time domain number 0.

By default PTP will use the best master clock algorithm (BMCA) to choose which device should be the server for the time domain.
For this how-to we manually configure Pi A to be the server and Pi B to be the client, by respectively specifying the `--serverOnly 1` and `--clientOnly 1` command line arguments.

### Start server

Start the PTP server on Pi A:

```bash
sudo ptp4l -i eth0 --verbose 1 --use_syslog 0 --step_threshold 1 --hwts_filter full --neighborPropDelayThresh 17000 --serverOnly 1
```

Wait for the server to print out `assuming the grand master role` before starting the client.

We mostly use the default configuration of `ptp4l`, except for these overrides:  
* `-i eth0` specifies the Ethernet interface to use.
* `--verbose 1` prints logs to stdout.
* `--use_syslog 0` prevents logs from being written to the syslog.
* `--step_threshold 1` tells `ptp4l` to step the clock when large time jumps occur.
  This is optional, but useful during testing to see if two devices keep pace.
* `--hwts_filter full` is an override that is required for the Raspberry Pi 5, to tell the network driver to always timestamp network packets.
  Without this, timestamping does not occur on all network packet types, and synchronization fails.
* `--neighborPropDelayThresh 17000` is also an override required for the Raspberry Pi 5.
  From experimentation we can see that there is a constant path delay of around 16.9 microseconds between two Raspberry Pi 5s.
  The Pi 5's Ethernet is connected to the RP1 chip, which is a coprocessor that handles IO.
  The latency between the two Pis is likely increased because this RP1 chip sits inline.
  This constant delay is above LinuxPTP's default threshold of 800 nanoseconds.
  We increase the threshold to just above the constant delay of around 16.9 microseconds.
* `--serverOnly 1` tells `ptp4l` to run in server mode, and not use BMCA to become a client.

For a detailed description of all configuration options, refer to the [`ptp4l` man page](https://manpages.ubuntu.com/manpages/latest/en/man8/ptp4l.8.html) .

### Start client

On Pi B, start the client:

```bash
sudo ptp4l -i eth0 --verbose 1 --use_syslog 0 --step_threshold 1 --hwts_filter full --neighborPropDelayThresh 17000 --clientOnly 1
```

The configurations for the client are identical to the server's, except for the last argument which changes to `--clientOnly 1` to force `ptp4l` to always run in client mode.

Many logs will be printed on the client device once it's running. A few of these are discussed below.

If the client found the server on the network, it will print:

```{terminal}
port 1 (eth0): new foreign master 2ccf67.fffe.1cbba1-1
selected best master clock 2ccf67.fffe.1cbba1
```

You will also see messages about the internal state transitions of `ptp4l`:

```{terminal}
ptp4l[24493.428]: port 1 (eth0): INITIALIZING to LISTENING on INIT_COMPLETE
...
ptp4l[24498.921]: port 1 (eth0): LISTENING to UNCALIBRATED on RS_SLAVE
...
ptp4l[24502.922]: port 1 (eth0): UNCALIBRATED to SLAVE on MASTER_CLOCK_SELECTED
```

Crucially, you will also see logs of the time offsets between the client and the server:

```{terminal}
ptp4l[24500.921]: master offset      21663 s0 freq   +7336 path delay     16925
ptp4l[24501.921]: master offset      21594 s1 freq   +7267 path delay     16925
ptp4l[24502.922]: master offset         -1 s2 freq   +7266 path delay     16925
ptp4l[24503.922]: master offset          4 s2 freq   +7270 path delay     16922
ptp4l[24504.922]: master offset          2 s2 freq   +7270 path delay     16922
ptp4l[24505.922]: master offset          0 s2 freq   +7268 path delay     16920
ptp4l[24506.922]: master offset        -18 s2 freq   +7250 path delay     16917
ptp4l[24507.922]: master offset         27 s2 freq   +7290 path delay     16909
```

The time offset will likely start at a very high number, jump rapidly to a smaller number, and then settle on a number around 0±100.
Two systems are generally considered synchronized with PTP when their clocks differ by less than 100 nanoseconds. It may take tens of seconds before the offset settles in this range.

## Synchronize system clock

On Pi A, start `phc2sys`, synchronizing from the system clock to eth0's PHC:

```bash
sudo phc2sys -s CLOCK_REALTIME -c eth0 -O 0 --step_threshold 1 -m -q
```

* `-s` specifies that the source is the system clock, a.k.a `CLOCK_REALTIME`.
* `-c` specifies the clock that needs to be synchronized.
  In this case it is either `eth0`, or its device node `/dev/ptp0`.
* `-O 0` tells the synchronizer that it should target a zero second offset between the two clocks.
  You can add an offset here, for example to account for leap seconds.
* `--step_threshold 1` (optional) tells `phc2sys` to step the clock when large time jumps occur.
* `-m` enables printing log messages to stdout.
* `-q` disables printing to the syslog.

For a detail description of all available configurations, refer to the [`phc2sys` man page](https://manpages.ubuntu.com/manpages/latest/en/man8/phc2sys.8.html).

```{note}
`phc2sys` has many features to automatically synchronize between clocks such as selecting the best source to synchronize from, and automatically handling leap seconds and daylight savings time.
The arguments we use here disable all these features, and use constant offsets.
This is to show a minimal setup of what this utility does.
```

After running the `phc2sys` command, the output should show the difference between eth0's clock and the system clock, and over time it should decrease to the range 0±100.

You will also notice time jumps being reported by `ptp4l` on Pi B.
This is because synchronization is being done from A's system clock, to A's PHC, to B's PHC.
Therefore, any clock adjustments made by `phc2sys` on Pi A are propagated to Pi B.

To synchronize Pi B's system clock from its PHC, run this command:

```bash
sudo phc2sys -s eth0 -c CLOCK_REALTIME -O 0 --step_threshold 1 -m -q
```

The only difference between this command and the `phc2sys` command on Pi A is the source clock and destination clock are swapped around.

After running the command, the terminal will print the offset statistics.

## Verify time synchronization

With the setup above, Pi A's system clock is synchronized to PI A's PHC which is then synchronized over the network to Pi B's PHC.
Finally, Pi B's PHC is synchronized to Pi B's system clock.

Therefore, Pi A's and Pi B's system clocks should be synchronized.
You can verify this by running the `date` command on both devices and checking if their clocks report the same time.
If the devices are set up in different time zones, they will still report the same time, offset by the number of hours between the two time zones.

On Pi A:
```{terminal}
   :user: ubuntu
   :host: pi-a
   :dir: ~

:input: date
Mon Apr  7 14:42:36 UTC 2025
```

On Pi B:

```{terminal}
   :user: ubuntu
   :host: pi-b
   :dir: ~

:input: date
Mon Apr  7 16:42:36 SAST 2025
```

If Pi A's system clock is incorrect, you can manually adjust it:

```{terminal}   
   :user: ubuntu
   :host: pi-a
   :dir: ~

:input: sudo date --set="2025-01-01 00:00:00"
Wed Jan  1 00:00:00 UTC 2025
```

Large time jumps will take longer to synchronize.
However, if you use `--step_threshold 1`, it should only take a few seconds for all the systems in the PTP time domain to be synchronized again.

You can also manually synchronize the time from the internet using the `ntpdate` tool from the `chrony` package:

```{terminal}
   :input: sudo ntpdate pool.ntp.org
   :user: ubuntu
   :host: pi-a
   :dir: ~

2025-04-07 14:51:36.570167 (+0000) +8347851.831618 +/- 0.050685 pool.ntp.org 196.10.98.182 s1 no-leap
CLOCK: time stepped by 8347851.831618
CLOCK: time changed from 2025-01-01 to 2025-04-07
```

<!-- TODO replace link to timemaster how-to -->
Alternatively, you can use Chrony or `ntpd` to continuously synchronize your system clock from the internet, and then use PTP to distribute the time and synchronize via Ethernet.
You can also follow our guide {doc}`rpi5-ptp-time-sync` to use a combination of PTP and NTP for redundancy.