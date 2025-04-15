# How to set up LoRa Basics™ Station on Ubuntu Core
The LoRa Basics™ Station snap is a [companion snap](https://ubuntu.com/core/docs/docker-companion-snap) for [LoRa Basics™ Station for Docker](https://github.com/xoseperez/basicstation-docker). It allows you to run and configure a LoRa gateway on Ubuntu Core devices. This guide will take you through the steps of setting up, running, and debugging the snap.

## Installing the snap
To install the snap from the Snap Store, run:

```shell
sudo snap install lora-basicstation
```
This automatically installs the [Docker](https://snapcraft.io/docker) snap as well if not present on the host. 

The LoRa Basics station snap needs to interact with the Docker Engine on the host. To allow that, connect the required docker interfaces by running:

```shell
sudo snap connect lora-basicstation:docker docker:docker-daemon
sudo snap connect lora-basicstation:docker-executables docker:docker-executables
```

## Configuration
Configuration of the gateway is done similar to the Docker container using the same configuration options as listed in the [upstream documentation](https://github.com/xoseperez/basicstation-docker?tab=readme-ov-file#configure-the-gateway). The key difference is that the snap configuration options are used in place of environment variables. 

```{note}

It's possible to create an environment variable file and pass that to the Docker container, but using the standard snap configuration options will enable you to leverage the the snap ecosystem to maintain and update your Ubuntu Core devices.

```

To translate from environment variables to snap options, you only need to change the name or key of the configuration option. The values remain the same. The variable name is changed to lower case and underscores are replaced with hyphens. An `env.` prefix is also added.

Example: `GATEWAY_EUI=DEADFFFEBEEF` becomes `env.gateway-eui=DEADFFFEBEEF`

Once the configurations are in this format, you can set them one by one using multiple `snap set` commands, e.g.:

```shell
sudo snap set lora-basicstation env.tc-uri="wss://eu1.cloud.thethings.network:8887"
sudo snap set lora-basicstation env.tc-key=NNSXS...
```

Or all at once with a single command:

```shell
sudo snap set lora-basicstation \
    env.gateway-eui=DEADFFFEBEEF \
    env.model=SX1301 \
    env.tc-key=NNSXS... \
    env.tc-uri=wss://eu1.cloud.thethings.network:8887 \
    env.device=/dev/spidev0.0 \
    env.interface=SPI \
    env.spi-speed=200000 \
    env.use-libgpiod=1 \
    env.gpio-chip=gpiochip4 \
    env.reset-gpio=22
```

An option can be cleared with the `snap unset` command, e.g.:

```shell
sudo snap unset lora-basicstation env.gateway-eui
```

You can also verify options using `snap get`. For example, you can check which defaults are already set immediately after installation by running:

```shell
$ sudo snap get lora-basicstation env
Key            Value
env.device     /dev/spidev0.0
env.gpio-chip  gpiochip0
```

These configurations are required to forward the correct devices to the Docker container. If you are using a Raspberry Pi 5 you must change the `env.gpio-chip` value to `gpiochip4` to allow the Docker container to reset the LoRa concentrator.

## Running the snap
To start the snap and enable the daemon to start automatically in the future, run:

```shell
sudo snap start --enable lora-basicstation
```
To stop the gateway software:

```shell
sudo snap stop lora-basicstation
```
To view the log output:

```shell
sudo snap logs -f lora-basicstation
```

## Verifying the environment
The snap configurations are translated to an environment file when it is started. This environment file is then passed to the Docker container when it is created.

To verify the contents of the environment file, run:

```shell
cat /var/snap/lora-basicstation/common/conf.env
```
## Verifying that the Docker container is running
To check if the Docker container is running, run:

```
sudo docker ps
```
If it does not list any containers with the name `lora-basicstation`, the container is not running.

## Finding the Gateway EUI

This snap provides a tool to generate an EUI from the device MAC address.
Refer to the [upstream documentation](https://github.com/xoseperez/basicstation-docker?tab=readme-ov-file#get-the-eui-of-the-gateway) for details on its usage. Run the following command to identify the Gateway EUI:

```shell
$ lora-basicstation.gateway-eui
Gateway EUI: 2CCF67FFFE1CBBA1 (based on interface eth0)
```

If the `env.gateway-eui` snap option is set, this tool will output the following:

```shell
$ lora-basicstation.gateway-eui
Gateway EUI: 0242ACFFFE110002 (based on environment variable)
```

## Finding the attached concentrators

An application to scan for connected LoRa concentrators is also included.
To learn more about its limitations, please see the [upstream documentation](https://github.com/xoseperez/basicstation-docker?tab=readme-ov-file#find-the-concentrator).
You can execute this application by running:

```shell
$ lora-basicstation.find-concentrator
```
This will output a list of devices, if any:

```
Looking for devices, this might take some time...

DEVICE             DESIGN      ID
---------------------------------------------------

0 device(s) found!
```







<!-- Links -->
