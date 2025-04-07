# How to use Timemaster

Timemaster is an application from the LinuxPTP package that combines NTP and PTP to provide a synchronized system clock, with redundancy. PTP is the more accurate of the two protocols, but can only synchronize against a single server at a time. NTP is less accurate, but allows synchronization from multiple servers concurrently, providing redundancy. Timemaster combines these two to allow fallback to NTP if the PTP server goes offline.

## Install dependencies

Timemaster and PTP synchronization is provided by the LinuxPTP package. Install the Debian package using APT, as the Snap's confinement currently makes using Timemaster impossible.

We assume you already have a working PTP setup. Have a look at [our how-to](to-be-replaced) to set it up.

On the NTP side, Timemaster can either use `ntpd` or Chrony. The default is to use Chrony. Install it:

```bash
sudo apt install chrony
```

After installation, Chrony will automatically be started as a system service. In our case this is unwanted, as we want Timemaster to start and configure it. Stop and disable the service:

```bash
sudo systemctl stop chronyd
sudo systemctl disable chronyd
```

## Configure Timemaster

The Timemaster config file is located at `/etc/linuxptp/timemaster.conf`. Open this file and make the following changes.

At the top of the config file you will see commented out lines that define an `ntp_server`. Replace this with configuration for multiple NTP servers. At least two, but four is normal. Try to use NTP servers that are close to you, to reduce the latency to them. If you are unsure, using the Ubuntu servers is fine, as it will auto select servers for you.

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
Chrony will also use any NTP servers that are defined system wide.
If this is unwanted, you can remove them from the system by editing the config files under `/etc/chrony`.
Or you can depend only on the system configured servers, and leave the `ntp_server` config option in `timemaster.conf` commented out.
```

The next part of the Timemaster config file is the definition for the PTP domain. The default config might work for you, so uncomment it. Then make sure the interface name is correct. In case your Ethernet interface, that supports PTP, is called `enp0s20f0u1u4`, the config should look like this:

```ini
[ptp_domain 0]
interfaces enp0s20f0u1u4
delay 10e-6
```

The rest of the config file can be left unchanged, unless you need specific PTP or NTP configurations.

If you are running on a Raspberry Pi 5, `ptp4l` will need additional options as described in [the other how-to](to-be-replaced). In this case your configuration should look like this:

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

* `-f /etc/linuxtp/timemaster.conf` \- specifies the path to the Timemaster config file  
* `-l 10` \- makes the Timemaster logs more verbose  
* `-m` \- enables logging to `stdout`
* `-q` \- disables logging to `syslog`

Refer to the [`timemaster` man page](https://manpages.ubuntu.com/manpages/latest/en/man8/timemaster.8.html) for a complete description of all available options.

## Check the logs

Timemaster only prints its own logs to `stdout`. To see the logs for its child processes, look at the system logs:

```bash
journalctl -f -a
```

To make the logs more readable, you can filter for a specific child application:

```bash
journalctl -a -f | grep ptp4l # or
journalctl -a -f | grep phc2sys # or
journalctl -a -f | grep chrony
```

After Timemaster is started, logs for these child processes will appear in the system log. After the startup sequence, there should be a log from `ptp4l` every second, indicating its synchronization state.

Chrony is ultimately responsible to make the decision of the best time source. You can check which time sources are being used by Chrony by running the `chronyc sources` command:

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

Chrony will monitor the NTP servers and the PTP clock. If it detects that the time from one of them is further out than the rest, it will ignore that clock, and print a log message saying it detected a `falseticker`.

As an experiment you can adjust the time of the PTP server. The PTP clock should be detected as false and the log will look similar to this:

```
Apr 04 12:52:28 pi5-2504-desktop chronyd[334944]: Selected source PTP0
# PTP clock manually adjusted...
Apr 04 12:53:36 pi5-2504-desktop chronyd[334944]: Detected falseticker PTP0
Apr 04 12:54:02 pi5-2504-desktop chronyd[334944]: Selected source 2001:4200:7000::6 (tick.sanren.ac.za)
Apr 04 12:54:04 pi5-2504-desktop chronyd[334944]: Detected falseticker PTP0
# PTP clock corrected...
Apr 04 13:17:36 pi5-2504-desktop chronyd[334944]: Selected source PTP0
```

We can query Chrony's status again and see `PTP0` is marked with an `x`, indicating that it is not being used.

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

After the PTP clock is corrected, it takes some time for Chrony's source selection algorithm to choose it again as the best source.

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
