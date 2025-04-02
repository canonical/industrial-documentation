# How to synchronise two Raspberry Pi 5s using LinuxPTP

The Raspberry Pi 5 supports hardware timestamping of network packets transmitted and received by its ethernet interface. This timestamping is handled by the PTP Hardware Clock (PHC), a separate clock dedicated to the network interface. This clock is exposed by the network driver to the Linux kernel, and shows up as `/dev/ptp0`.

PTP stands for precision time protocol, which is a standardized protocol to synchronize PHCs over ethernet. LinuxPTP provides the `ptp4l` application to do this synchronization.

As an additional step one can synchronize your system clock to the PHC, or synchronize your system clock from the PHC, using the `phc2sys` utility.

```{figure} images/RPi5-ptp.svg
   :width: 100%
   :alt: Raspberry Pi 5 clock topology

   Raspberry Pi 5 clock topology
```

## Install LinuxPTP

LinuxPTP is available in Ubuntu as either a snap or a deb package. The snap gets more frequent updates, but the deb is more stable. Either of these will work for the rest of this guide.

You can install the snap using:

```bash
sudo snap install linuxptp
```

```{note}
The `linuxptp` snap is not yet available under the stable channel. You can install it from the beta channel by appending `--beta` to the command above. The current version of the snap installs `linuxptp` version 4.4.
```

Alternatively you can install the Debian package:

```bash
sudo snap install linuxptp
```

```{note}
This will give you an older, but stable version of LinuxPTP - currently version 4.2.
```

## Set up your network

LinuxPTP can use either layer 2 or layer 3 networking. We will use layer 3, IP networking, for the two Raspberry Pis to communicate. If you use a switch between the two Raspberry Pis, the switch will also need to support the PTP protocol. As a starting point, do not use a switch, but connect the two Pis directly to each other.

From now on we will refer to *Pi A* and *Pi B* to distinguish between the two devices.

Assign static IP addresses to the two Pis, so that they can communicate on this direct link. We will assign 10.0.0.1 to Pi A, and 10.0.0.2 to Pi B.

On Ubuntu Desktop, ethernet is managed by Network Manager. In this case remove the interface from Network Manager so that we can manually specify the address:

```bash
sudo nmcli dev set eth0 managed no
```

Assign a static IP to the interface, and up the link.

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

On each device ping the other to make sure layer 3 IP networking is functional.

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

PTP in a server-client architecture. One time domain has one server with multiple clients synchronizing against it. One ethernet can have multiple of these time domains.

PTP has a best master clock algorithm (BMCA) to choose which device should be the server. For our experiment we manually configure Pi A to be the server and Pi B to be the client, by respectively specifying the `--serverOnly 1` and `--clientOnly 1` command line arguments.

Start the PTP server on Pi A:

```bash
sudo ptp4l -i eth0 --verbose 1 --use_syslog 0 --hwts_filter full --neighborPropDelayThresh 17000 --serverOnly 1
```

Wait for the server to print out `assuming the grand master role` before starting the client.

We use mostly the default configuration of `ptp4l`, except for these overrides:  
* `-i eth0` specifies the ethernet interface to use  
* `--verbose 1` prints logs to stdout  
* `--use_syslog 0` prevents logs from being written to the syslog  
* `--hwts_filter full` is an override that is required for the Raspberry Pi 5, to tell the network driver to always timestamp network packets. Without this, timestamping does not occur on all network packet types, and synchronization fails.  
* `--neighborPropDelayThresh 17000` is also an override required for the Raspberry Pi 5\. From experimentation we can see that there is a constant path delay of around 16.9 microseconds between two Raspberry Pi 5s. The Pi 5’s ethernet is connected to the RP1 chip, which is a coprocessor that handles IO. The latency between the two Pis is likely increased because this RP1 chip sits inline. This constant delay is above LinuxPTP’s default threshold of 800ns. We increase the threshold to just above the constant delay of around 16.9 microseconds.  
* `--serverOnly 1` tells `ptp4l` to run in server only mode, and not use BMCA to become a client.

On Raspberry Pi B, start the client:

```bash
sudo ptp4l -i eth0 --verbose 1 --use_syslog 0 --hwts_filter full --neighborPropDelayThresh 17000 --clientOnly 1
```

The command for the client is almost identical to the server, except that we change the last argument to `--clientOnly 1` to force `ptp4l` to always run in client mode, and not try and become the server using BMCA.

On the client device you will see many logs being printed. Let’s look at some of the important ones.

If the client found the server on the network, it will print these messages:

```{terminal}
port 1 (eth0): new foreign master 2ccf67.fffe.1cbba1-1
selected best master clock 2ccf67.fffe.1cbba1
```

You will see messages about the internal state transitions of ptp4l:

```{terminal}
ptp4l[24493.428]: port 1 (eth0): INITIALIZING to LISTENING on INIT_COMPLETE
...
ptp4l[24498.921]: port 1 (eth0): LISTENING to UNCALIBRATED on RS_SLAVE
...
ptp4l[24502.922]: port 1 (eth0): UNCALIBRATED to SLAVE on MASTER_CLOCK_SELECTED
```

And most importantly, you will see logs of the time offsets between the client and the server:

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

The time offset will likely start at a very high number, jump rapidly to a smaller number, and then settle on a number around 0±100. Generally two systems are considered synchronized with PTP when their clocks differ by less than 100 nanoseconds. A software Proportional-Integral control system is used by `ptp4l` to tune the frequency of the PHC. Depending on how well the systems are synchronized, it might take tens of seconds to stabilise around 0±100.

## Synchronize the system clock

For this step we assume that Pi A, the server, has an accurate system clock. You can manually synchronize it from the internet using the `ntpdate` tool from the `ntpsec-ntpdate` package:

```{terminal}
   :input: sudo ntpdate pool.ntp.org
   :user: ubuntu
   :host: pi-a
   :dir: ~

2025-03-31 15:09:07.128437 (+0000) +0.000957 +/- 0.009660 pool.ntp.org 196.10.52.57 s2 no-leap
```

On Pi A, start `phc2sys`, synchronising from the system clock to eth0’s PHC:

```bash
sudo phc2sys -s CLOCK_REALTIME -c eth0 -O 0 -m -q
```

* `-s` specifies that the source is the system clock, a.k.a `CLOCK_REALTIME`   
* `-c` specifies the clock that needs to be synchronized. In this case it is either `eth0`, or its alias `/dev/ptp0`  
* `-O 0` tells the synchronizer that it should target a 0 nanosecond offset between the two clocks. You can add an offset here, for example to account for leap seconds  
* `-m` enables printing log messages to stdout  
* `-q` disables printing to the syslog

```{note}
`phc2sys` has many more features to automatically synchronise between clocks, like selecting the best source to synchronize from, and automatically handling leap seconds and daylight savings time. The arguments we use here disable all these features, and use constant offsets. This is to show a minimal setup of what this utility does. In a real world scenario you will highly likely never synchronise from your system clock, as it is usually inaccurate. You would rather use a source clock that is synchronised by a GPS receiver.
```

After running the `phc2sys` command, the output should show the difference between eth0’s clock and the system clock, and over time it should decrease to the range 0±100. 

On Pi B, the client, you will see some jumps in the time offset too, as we are now synchronising from A’s system clock, to A’s PHC, to B’s PHC.

To synchronize B’s system clock from its PHC, run this command:

```bash
sudo phc2sys -s eth0 -c CLOCK_REALTIME -O 0 -m -q
```

The terminal will print the offset statistics.

## Verifying that it works

We will now verify that time is propagating from Pi A to Pi B. Stop `phc2sys` on Pi B.

Change the system clock for Pi B to some date in the past, and verify that it has changed:

On Pi B:
```{terminal}   
   :user: ubuntu
   :host: pi-b
   :dir: ~

:input: sudo date --set="2015-01-01 14:15:16.178"
Thu Jan  1 14:15:16 SAST 2015
:input: date
Thu Jan  1 14:15:17 SAST 2015
```

Start `phc2sys` on Pi B and let it run until the offset is within 0±100:

```{terminal}   
   :user: ubuntu
   :host: pi-b
   :dir: ~

:input: sudo phc2sys -s eth0 -c CLOCK_REALTIME -O 0 -m -q
phc2sys[28916.598]: CLOCK_REALTIME phc offset -299834345657614183 s0 freq   +2010 delay   1037
phc2sys[28917.598]: CLOCK_REALTIME phc offset -299834345657613951 s1 freq   +2242 delay   1037
phc2sys[28918.598]: CLOCK_REALTIME phc offset       -47 s2 freq   +2195 delay   1000
phc2sys[28919.598]: CLOCK_REALTIME phc offset       -94 s2 freq   +2134 delay   1037
phc2sys[28920.598]: CLOCK_REALTIME phc offset       -30 s2 freq   +2170 delay   1055
phc2sys[28921.598]: CLOCK_REALTIME phc offset        78 s2 freq   +2269 delay   1000
phc2sys[28922.599]: CLOCK_REALTIME phc offset        35 s2 freq   +2249 delay   1037
phc2sys[28923.303]: CLOCK_REALTIME phc offset         9 s2 freq   +2234 delay   1037
```

In another terminal run the `date` command to see if it is synchronized. It should print the current wall clock time, and not the time we set it to before:

```{terminal}   
   :user: ubuntu
   :host: pi-b
   :dir: ~

:input: date
Mon Mar 31 17:25:32 SAST 2025
```
