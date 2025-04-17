# Containers

Ubuntu Core has been developed specifically to leverage the power of snap packages, with their autonomous upgrades and confined sandbox.

However, Ubuntu Core can also be used to run and deploy Docker container images, either from the command line or embedded within an Ubuntu Core image.

## Deploying Docker

Install and run Docker from within the Ubuntu Core environment. Build a snap which embeds the container images you wish to deploy from a device.

- [Deploy Docker containers from a snap](how-to/deploy-docker-from-a-snap)

## Packaging Docker

How Docker images can be bundled inside snaps to be deployed together with other resources within the package.

Ship containerized applications with Ubuntu Core by building a custom image that can then be used to create bootable media and can automate deployment of a production system.


- [Package Docker images in a snap](how-to/package-docker-images-in-a-snap)
- [Build an image for Docker deployment](how-to/build-an-image-for-docker-deployment)


```{toctree}
:hidden:
:titlesonly:
:maxdepth: 2
:glob:

Deploy Docker from a snap <how-to/deploy-docker-from-a-snap>
Package Docker images in a snap <how-to/package-docker-images-in-a-snap>
Build an image for Docker deployment <how-to/build-an-image-for-docker-deployment>