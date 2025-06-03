# Communication Protocols
Communications in IoT networks are managed by different protocols, often with the goal of achieving fast, reliable, and secure transmission of data. Different protocols can be implemented depending on characteristics such as location, security, size of the network, and expected level of responsiveness.

Beyond the choice of protocol, when setting up IoT infrastructure, there are additional considerations such as features to be implemented to support your applications and achieve the performance you want, setting up devices for efficient management, and how niche features can be implemented.

With Ubuntu Core devices you can set up your IoT network to use industry-leading protocols, ensuring peak performance, long-term network reliability, and device compatibility.

## LoRa on Ubuntu Core
LoRaWAN(Long range wide area network) is a networking protocol. It has characteristics that are well suited to IoT applications such as low power consumption, an open standard, and large area coverage. With an Ubuntu Core device, you can set up and manage your own LoRa gateway. 

- {doc}`Set up LoRa Basics™ Station on Ubuntu Core<how-to/setup-lora-on-ubuntu-core>`
- {doc}`Manage a fleet of LoRa gateways using Ubuntu Core with Landscape<how-to/manage-fleet-uc-gateways-landscape>`

## Time Synchronization
In IoT applications such as industrial automation, financial trading, and telecommunications, the system clocks in all connected devices must be synchronized to ensure they operate in tandem. Clocks in devices running Ubuntu Core can be synchronized in different ways including using a single time protocol or combining two different protocols. 

- {doc}`How to synchronize two Raspberry Pi 5s using LinuxPTP <how-to/rpi5-ptp-time-sync>`
- {doc}`Combine NTP and PTP for redundant time synchronization<how-to/timemaster>`







```{toctree}
:maxdepth: 1
:hidden:
how-to/rpi5-ptp-time-sync
how-to/timemaster
how-to/setup-lora-on-ubuntu-core
how-to/manage-fleet-uc-gateways-landscape
```