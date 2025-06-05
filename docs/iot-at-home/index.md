# Home automation

Home automation applications are an important driver of IoT.
Many smart assistants, security systems, and  smart appliances are targeted at this market, and many use the [Matter standard](https://csa-iot.org/all-solutions/matter/).

The Matter open source connectivity standard allows IoT devices to be connected securely and reliably.
Its wide adoption limits compatibility issues.

Ubuntu offers a rich suite of established developer tools, simplifying the work of building Matter applications.

With a device running Ubuntu, you can build a Matter device, deploy Home Assistant, set up an OpenThread Border Router for your IoT devices, and more. 


### Get started
Explore the basics of Smart home and Matter using easily-available hardware.
- {doc}`Build your first Matter device with a Raspberry Pi<tutorial/pi-gpio-commander>`
- {doc}`Commission and control Matter devices with Chip Tool<how-to/chip-tool-commission-and-control>`
- {doc}`Install Home Assistant on Ubuntu Core<how-to/otbr-ubuntu-core-image>`

### Thread Networking
Set up a Thread network and gateway for your Matter devices. 
- {doc}`Run Matter applications with Thread networking on Ubuntu<how-to/matter-and-thread-on-ubuntu>`
- {doc}`Set up OpenThread Border Router on Ubuntu<how-to/otbr-on-ubuntu>`
- {doc}`Create an OS image with OpenThread Border Router<how-to/otbr-ubuntu-core-image>`


## Project and Community
Matter on Ubuntu references existing open source implementations.
The Matter standard and SDK are supported by the Connectivity Standards Alliance (CSA).



Canonical is responsible for Snap packages presented in this documentation:
- [Chip Tool]
- [OpenThread Border Router]
- [Matter Pi GPIO Commander]


Other resources:
- [Matter Standard]
- [Matter SDK]

<!-- Links -->

[Chip Tool]: https://snapcraft.io/chip-tool 
[OpenThread Border Router]: https://snapcraft.io/openthread-border-router
[Matter Pi GPIO Commander]: https://snapcraft.io/matter-pi-gpio-commander
[Matter Standard]: https://csa-iot.org/all-solutions/matter/
[Matter SDK]: https://github.com/project-chip/connectedhomeip

```{toctree}
:maxdepth: 1
:hidden:

tutorial/pi-gpio-commander
how-to/chip-tool-commission-and-control
how-to/home-assistant-ubuntu-core
how-to/matter-and-thread-on-ubuntu
how-to/otbr-on-ubuntu
how-to/otbr-ubuntu-core-image
```