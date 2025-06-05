# Smart home

Smart home applications are an important driver of IoT.
Many smart assistants, security systems, and  smart appliances are targeted at this market, and many use the [Matter standard](https://csa-iot.org/all-solutions/matter/).

The Matter open source connectivity standard allows IoT devices to be connected securely and reliably.
Its wide adoption limits compatibility issues.

Ubuntu offers a rich suite of established developer tools, simplifying the work of building Matter applications. 
With a device running Ubuntu, you can build a Matter device, deploy Home Assistant, set up an OpenThread Border Router for your IoT devices, and more. 


## Get started
Explore the basics of smart homes and Matter using readily available hardware.
- {doc}`Build your first Matter device with a Raspberry Pi<tutorial/pi-gpio-commander>`
- {doc}`Commission and control Matter devices with Chip Tool<how-to/chip-tool-commission-and-control>`
- {doc}`Install Home Assistant on Ubuntu Core<how-to/otbr-ubuntu-core-image>`

## Thread Networking
Set up a Thread network and gateway for your Matter devices. 
- {doc}`Run Matter applications with Thread networking on Ubuntu<how-to/matter-and-thread-on-ubuntu>`
- {doc}`Set up OpenThread Border Router on Ubuntu<how-to/otbr-on-ubuntu>`
- {doc}`Create an OS image with OpenThread Border Router<how-to/otbr-ubuntu-core-image>`


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