# How to use Timemaster

Timemaster is an application from the [Linux PTP project](https://linuxptp.sourceforge.net/) that combines Network Time Protocol (NTP) and Precision Time Protocol (PTP) to provide a synchronized system clock, with redundancy.
PTP is the more accurate of the two protocols, but can only synchronize against a single server at a time.
NTP is less accurate, but allows synchronization from multiple servers concurrently, providing redundancy.
Timemaster combines the two protocols to allow the system to fall back to NTP if the PTP server's time drifts too far, or if it goes offline.

This guide will show you how to set up Timemaster, how to monitor it, and finally show how you can verify that fallback to NTP is working.

## Install dependencies

To proceed with this guide, you'll need a working PTP setup using Linux PTP.
On Ubuntu you can install it as a deb package:
<!-- Currently this has to be the deb package, but that will likely change in the next cycle. -->
```bash
sudo apt install linuxptp
```

For more details on setting up PTP on the Raspberry Pi 5, you can follow {doc}`rpi5-ptp-time-sync`. <!-- How to synchronize two Raspberry Pi 5s using LinuxPTP -->

On the NTP side, Timemaster can either use `ntpd` or Chrony.
In this guide we will be using Chrony, which is considered the [more modern and accurate](https://chrony-project.org/faq.html#_how_does_chrony_compare_to_ntpd) implementation of the two.
Install it:

```bash
sudo apt install chrony
```

After installation, Chrony will be automatically started as a system service.
Stop and disable it so Chrony can be started and configured by Timemaster:

```bash
sudo systemctl stop chronyd
sudo systemctl disable chronyd
```

## Configure Timemaster

The Timemaster config file is located at `/etc/linuxptp/timemaster.conf`.
Open this file and make the following changes.

At the top of the config file, replace the commented out lines that define an `ntp_server` with configurations for two or more ntp servers.
Choose NTP servers close to your location to reduce latency.
You can also use the Ubuntu servers as these will auto select appropriate servers for you.

```ini
[ntp_server 0.ubuntu.pool.ntp.org]
minpoll 4
maxpoll 4

[ntp_server 1.ubuntu.pool.ntp.org]
minpoll 4
maxpoll 4

[ntp_server 2.ubuntu.pool.ntp.org]
minpoll 4
maxpoll 4

[ntp_server 3.ubuntu.pool.ntp.org]
minpoll 4
maxpoll 4
```

```{note}
By default Chrony will also use any NTP servers that are defined system wide.
This is set in Chrony's configuration via the `chrony.conf` option in `timemaster.conf`.
```

Next, define the PTP domain.
The default configuration can be used by uncommenting it.
Confirm that the Ethernet interface name is correct.
If your interface is called `enp0s20f0u1u4`, the config should look this:

```ini
[ptp_domain 0]
interfaces enp0s20f0u1u4
delay 10e-6
```

The rest of the config file can be left unchanged unless you need specific PTP or NTP configurations.

On a Raspberry Pi 5, `ptp4l` will need additional options as described in {doc}`rpi5-ptp-time-sync`. <!-- How to synchronize two Raspberry Pi 5s using LinuxPTP -->
In that case the additional configuration should look like this:

```ini
[ptp4l]
path /usr/sbin/ptp4l
options --hwts_filter full --neighborPropDelayThresh 17000
```

Refer to the [`timemaster` man page](https://manpages.ubuntu.com/manpages/latest/en/man8/timemaster.8.html) for more details on configuration options.

## Run Timemaster

Start Timemaster by running this command:

```bash
sudo timemaster -f /etc/linuxptp/timemaster.conf -l 10 -m -q
```

* `-f /etc/linuxtp/timemaster.conf` specifies the path to the Timemaster config file  
* `-l 10` makes the Timemaster logs more verbose  
* `-m` enables logging to `stdout`
* `-q` disables logging to `syslog`

Refer to the [`timemaster` man page](https://manpages.ubuntu.com/manpages/latest/en/man8/timemaster.8.html) for a complete description of all available options.

## Check logs

Timemaster prints its logs to `stdout`, but the logs for its child processes go to the system log.
To see these system logs:

```bash
journalctl -f
```

To make the logs more readable, you can filter for a specific child application:

```bash
journalctl -f | grep ptp4l # or
journalctl -f | grep phc2sys # or
journalctl -f | grep chrony
```

Once Timemaster is up and running you will see `chrony` and `phc2sys` logged their configuration and startup sequences.
`ptp4l` will start logging its synchronization state.

Chrony automatically selects the best time source.
You can check which time sources are being used using the `chronyc` utility:

```{terminal}
   :user: ubuntu
   :host: ubuntu
   :dir: /home/ubuntu/

:input: chronyc sources -v
  .-- Source mode  '^' = server, '=' = peer, '#' = local clock.
 / .- Source state '*' = current best, '+' = combined, '-' = not combined,
| /             'x' = may be in error, '~' = too variable, '?' = unusable.
||                                                 .- xxxx [ yyyy ] +/- zzzz
||      Reachability register (octal) -.           |  xxxx = adjusted offset,
||      Log2(Polling interval) --.      |          |  yyyy = measured offset,
||                                \     |          |  zzzz = estimated error.
||                                 |    |           \
MS Name/IP address         Stratum Poll Reach LastRx Last sample               
===============================================================================
#* PTP0                          0   2   377     3    -52ns[ -174ns] +/- 3555ns
^- ntp-nts-1.ps6.canonical.>     2   4   377     5  -2780us[-2780us] +/-   39ms
^- ntp-nts-2.ps5.canonical.>     1   4   377    10   -350us[ -350us] +/-   13ms
^- ntp-nts-3.ps5.canonical.>     3  10   377   697  -1435us[-1547us] +/-   54ms
```

## Check failover

Chrony monitors the NTP servers and the PTP clock.
If it detects that the time from one clock is not aligned with the rest, it will ignore it and print a log message saying it detected a `falseticker`. 

You can test this by adjusting the system clock of the PTP server.
The PTP clock should be detected as false and the log will look similar to this:

```{terminal}
   :user: ubuntu
   :host: ubuntu
   :dir: /home/ubuntu/

:input: journalctl -f | grep chrony
chronyd[334944]: Selected source PTP0
... (PTP server system clock manually adjusted)
chronyd[334944]: Detected falseticker PTP0
chronyd[334944]: Selected source 2620:2d:4000:1::2123 (ntp-nts-2.ps5.canonical.com)
chronyd[334944]: Detected falseticker PTP0
```

If you query Chrony's status again, `PTP0` will be marked with an `x`, indicating that it is not being used as it may be in error.

```{terminal}
   :user: ubuntu
   :host: ubuntu
   :dir: /home/ubuntu/

:input: chronyc sources
MS Name/IP address         Stratum Poll Reach LastRx Last sample               
===============================================================================
#x PTP0                          0   2   377     2   +37.0s[ +37.0s] +/-  219ms
^- ntp-nts-1.ps6.canonical.>     2   4   377     3  +1059us[ +783us] +/-   48ms
^* ntp-nts-2.ps5.canonical.>     1   4   377     2  -1810us[-2087us] +/-   18ms
^- ntp-nts-3.ps5.canonical.>     3   6    77    62  +2439us[+1881us] +/-   67ms
```

After a clock has been corrected, Chrony's source selection algorithm will choose it again.

```{terminal}
   :user: ubuntu
   :host: ubuntu
   :dir: /home/ubuntu/

:input: journalctl -f | grep chrony
chronyd[334944]: Detected falseticker PTP0
... (PTP server system clock corrected)
chronyd[334944]: Selected source PTP0
```

```{terminal}
   :user: ubuntu
   :host: ubuntu
   :dir: /home/ubuntu/

:input: chronyc sources
MS Name/IP address         Stratum Poll Reach LastRx Last sample               
===============================================================================
#* PTP0                          0   2   377     3    +73us[  +73us] +/-   11ms
^- ntp-nts-1.ps6.canonical.>     2   4   377    15   +251us[ +278us] +/-   69ms
^+ ntp-nts-2.ps5.canonical.>     1   4   377    14   +395us[ +421us] +/-   15ms
^- ntp-nts-3.ps5.canonical.>     3   7   377    88   +848us[ +435us] +/-   65ms
```

```{note}
There will be a noticeable delay between the time a clock is corrected and the time the algorithm chooses it again.
```
