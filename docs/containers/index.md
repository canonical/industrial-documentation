# Containers

A container is a software package consisting of an application and all the dependencies it needs to run.
Applications packaged as containers can run reliably in different computing environments.

Ubuntu Core has been developed specifically to leverage the power of snap packages, with their autonomous upgrades and confined sandbox. 
However, Ubuntu Core can also be used to run Docker containers.

There are several ways to deploy Docker containers on Ubuntu Core. 
For basic use cases, you can run a Docker container [from the command line](https://documentation.ubuntu.com/core/how-to-guides/container-deployment/run-a-docker-container/). Where scalable solutions are required, you can deploy Docker containers via snaps or Ansible. 

## Deploying Docker via snaps

You can create a snap and use it to deploy a Docker container.
You can also package an existing Docker image into a snap or build an Ubuntu Core image with containerized Docker applications.

- {doc}`Deploy Docker Containers from a snap <how-to/deploy-docker-container-from-a-snap>`
- {doc}`Package Docker images in a snap <how-to/package-docker-images-in-a-snap>`
- {doc}`Build an Ubuntu Core image containing Docker <how-to/build-ubuntu-core-image-containing-docker>`
- {doc}`Docker companion snap <explanation/docker-companion-snap>`


## Deploying Docker via Ansible

You can use Ansible to deploy Docker containers to a fleet of devices.

- {doc}`Deploy Docker containers on Ubuntu Core using Ansible <how-to/deploy-docker-containers-uc-ansible>`


```{toctree}
:hidden:
:titlesonly:
:maxdepth: 2

how-to/deploy-docker-container-from-a-snap
how-to/package-docker-images-in-a-snap
how-to/build-ubuntu-core-image-containing-docker
explanation/docker-companion-snap
how-to/deploy-docker-containers-uc-ansible
```
