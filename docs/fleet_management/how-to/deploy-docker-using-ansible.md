# Deploy Docker Container on Ubuntu Core using Ansible

In this blog post, we'll explore how to deploy Docker containers on Ubuntu Core using Ansible's snap module. While Ubuntu Core offers a secure and reliable platform ideal for running production environments, particularly for managing deployments across multiple edge devices, the method demonstrated here also works on any other distribution that supports snapd. As an example, we'll demonstrate the deployment of RabbitMQ instances using Docker containers.

## Ansible

Ansible is an open source IT automation engine that automates provisioning, configuration management, application deployment, orchestration, and many others IT processes. It is simple in essence and easy to understand and YAML based, which reads like an English text.

In essence, Ansible has two major components, the control node which is the source of where the commands came from and the managed nodes, which are the devices you want to deploy your automations.

## Ansible Inventory

The Ansible inventory is the collection of machines that you want to control in your deployment. Essentially it needs two things for each control node, the hostname (or the IP address) and the user. Ansible executes it's actions mostly over SSH, but it's not limited to this connection type. The other connections types are called plugins and can be consulted with ansible-doc -t connection -l.

Here, the collection of our inventory will include a Raspberry Pi 4 and a Raspberry Pi 5, that we're going to call them as machines are presented in the following code block:

```yaml
machines:
  hosts:

    # 24.04.1 LTS (Noble Numbat) - Rpi4
    rpi4:
      ansible_host: 192.168.3.57
      ansible_user: locnnil

    # UC24 - RPi5
    rpi5:
      ansible_host: buri.local
      ansible_user: ubuntu
      ansible_python_interpreter: /usr/bin/python3
```

NOTE: Setting ansible_python_interpreter is necessary for Ubuntu Core for avoiding a warning message related to the host is using the discovered Python interpreter at a certain location, but a future installation of another Python interpreter could break this. So we leave the python interpreter path fixed to avoid this sort of problem that may happen.

This tutorial uses as Hardware two Raspberry Pis (4 and 5), both Ubuntu Core compatible hardware products. The Raspberry Pi 5 will run Ubuntu Core 24, while the Raspberry Pi 4 will run Ubuntu Server (Noble Numbat) 24.04.1 LTS. This is to demonstrate how easy it is to switch your containerized runtimes from an Ubuntu server installation like Noble to Ubuntu Core and enhance the security of your system. There are extensive documentation on how to install Ubuntu core on the Get started documentation rather if you want to try a pre-built image or build yourself an Ubuntu Core image.
You can come back here once you have it setup. Or, you can also use a fleet of Multipass VM to follow this section, since Multipass also provides Ubuntu Core as one of its options. Using Multipass you can spin up as many Ubuntu Cores virtual machines as you want and see the magic of your fleet of devices being deployed simultaneously. It's important to reforce that this tutorial will work on any distribution that supports snapd

By defining the inventory.yaml we can already gain from a wide range of features. You can ping the devices to make sure they are reachable, or perform ad hoc operations, such as executing a shell commands like:

```yaml
machines:
  hosts:

    # 24.04.1 LTS (Noble Numbat) - Rpi4
    rpi4:
      ansible_host: 192.168.3.57
      ansible_user: locnnil

    # UC24 - RPi5
    rpi5:
      ansible_host: buri.local
      ansible_user: ubuntu
      ansible_python_interpreter: /usr/bin/python3
```

```{terminal}
$ ansible machines -i inventory.yaml -m shell -a "cat /proc/device-tree/model"
rpi5 | CHANGED | rc=0 >>
Raspberry Pi 5 Model B Rev 1.0
rpi4 | CHANGED | rc=0 >>
Raspberry Pi 4 Model B Rev 1.5
```

```{terminal}
$ ansible machines -i inventory.yaml -m ping
rpi5 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
rpi4 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
```

## Ansible Community Snap Module

In this blog post, our focus is on Ubuntu Core, an operating system that provides all the necessary infrastructure for managing applications. The official packaging format used in Ubuntu Core is snaps, that are secure, reliable packages that employ a robust sandboxing mechanism to ensure applications only access the resources they need. Also, snaps ensure that applications run in an isolated environment, similar to containers, which enhances both security and consistency.

To manage those snaps, we will use the community-maintained Ansible snap module. This module simplifies essential snap management tasks, including channel selection, installation in classic or dangerous modes, and configuring snap options.
Deploy Docker application with an Ansible playbook

Docker snap was mainly created for running container workloads on Ubuntu Core machines, in this case, the advantage in using Docker as a Snap is to have an additional layer of security that is great for production scenarios, which makes it also a great option for Ubuntu Server.
So let's go to business:

Everything that you need to get started writing your playbook.yaml is the name of your hosts that you define in your inventory.yaml, a name for your deployment, then you have the array of tasks, and you're ready to begin:

```yaml
- name: Deploy RabbitMQ container
  hosts: uc_machines
  tasks:
```

A good way to begin is by starting to ping our hosts and check if all machines are responding, to guarantee that we won't have connectivity problems. Ansible has a builtin module for that:

```yaml
- name: Deploy RabbitMQ
  hosts: uc_machines
  tasks:
  - name: Ping my hosts
      ansible.builtin.ping:
```

Since we're demonstrating this deployment targeting Ubuntu Core, the Docker in the format of snap is necessary, since Ubuntu Core doesn't support deb packages. But it's worth to notice that Docker snap is also ideal for Servers when running production workloads. Then we can start by installing the Docker snap, using for that the Ansible snap module for this purpose:

```yaml
    - name: Install Docker snap
      become: true
      community.general.snap:
        name:
          - docker
        channel: latest/stable
        state: present
```

Here an important subject comes: "become: true" which is required for privilege escalation, which is when an operation that requires root privileges or in the scenario of some snap operations, when the user isn't logged with snap login command. Since we use the pre-built image that ships with console-conf, it takes care of login during the first boot, then it wouldn't be necessary for Core, for further information checkout this document. But it's necessary for Noble, since by default isn't logged after the first boot configuration.

The basic configuration needed to spin up a container of RabbitMQ is to configure the user, and it's password.
As there is not always an Ansible module for everything, it's always possible to use the built-in shell module, like used in this case. It's important to note the path used to store the configuration. On Ubuntu Core, most system files and directories are mounted as read-only, with only a few exceptions. Thanks to the snap home interface grant for Docker snap, the home directory will be used to store the configuration and volume data for the deployment. There are other examples of important directories that can are readable and writable on the data locations document.

```yaml
- name: Setup RabbitMQ configurations
      become: true
      ansible.builtin.shell: |
        sudo echo "RABBITMQ_DEFAULT_USER=demo-user" > $HOME/rabbitmq_conf.env
        sudo echo "RABBITMQ_DEFAULT_PASS=SomeReallyStrongPassword" >> $HOME/rabbitmq_conf.env
```

You may realize that we assign the same password for all devices, this isn't the best case scenario, and for avoiding doing that, you may want to divide your inventory into two different ones and create rules and use patterns targeting hosts.

As Docker is a famous tool used for deployment of containers, Ansible also provides a family of Docker modules for doing Docker operations.

Then it's time to start the RabbitMQ container using the Ansible Docker container module:

```yaml
    - name: Start RabbitMQ container
      become: true
      community.docker.docker_container:
        image: rabbitmq:4.0.2-management
        name: message_broker
        hostname: docker_ansible_demo
        env_file: $HOME/rabbitmq_conf.env
        volumes: /var/snap/docker/current/data:/var/lib/rabbitmq
        ports:
          # Publish container port 5672 as host port 17338
          - 17338:5672
          # Publish container port 15672 as host port 18280
          - 18280:15672
```

Here the "become: true" is actually needed because the user on Ubuntu Core isn't part of the docker group and Docker operations need root privileges.
For that, we're using the docker_container module, another module from the family of Ansible Docker modules. The volume is mapped to a place where the docker snap is allowed to write on. You may realize that the port syntax follows the same as the Docker CLI does.

And then, outside the playbook, using the inventory.yaml, you can listen the containers and see that everything is in order, since Ansible doesn't show the output by default but only the result of the command:

```{terminal}
$ ansible machines -i inventory.yaml -m shell -a "sudo docker container ls"
rpi5 | CHANGED | rc=0 >>
CONTAINER ID   IMAGE                       COMMAND                  CREATED       STATUS       PORTS                                                                                                          NAMES
a427d99efd75   rabbitmq:4.0.2-management   "docker-entrypoint.s…"   3 hours ago   Up 3 hours   4369/tcp, 5671/tcp, 15671/tcp, 15691-15692/tcp, 25672/tcp, 0.0.0.0:17338->5672/tcp, 0.0.0.0:18280->15672/tcp   message_broker
rpi4 | CHANGED | rc=0 >>
CONTAINER ID   IMAGE                       COMMAND                  CREATED       STATUS       PORTS                                                                                                          NAMES
74f18efdd315   rabbitmq:4.0.2-management   "docker-entrypoint.s…"   3 hours ago   Up 3 hours   4369/tcp, 5671/tcp, 15671/tcp, 15691-15692/tcp, 25672/tcp, 0.0.0.0:17338->5672/tcp, 0.0.0.0:18280->15672/tcp   message_broker
```

Then, putting everything together and running your playbook.yaml you got:

```{terminal}
$ ansible-playbook -i inventory.yaml playbook.yaml

PLAY [Deploy Docker on Ubuntu Core with Ansible] ***************************

TASK [Gathering Facts] *****************************************************
ok: [rpi5]
ok: [rpi4]

TASK [Ping my hosts] *******************************************************
ok: [rpi5]
ok: [rpi4]

TASK [Install Docker snap] *************************************************
ok: [rpi5]
ok: [rpi4]

TASK [Pull an image] *******************************************************
ok: [rpi5]
ok: [rpi4]

TASK [Setup RabbitMQ configurations] ***************************************
changed: [rpi5]
changed: [rpi4]

TASK [Start RabbitMQ container] ********************************************
ok: [rpi5]
ok: [rpi4]

PLAY RECAP *****************************************************************
rpi4                       : ok=6    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
rpi5                       : ok=6    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

With that, you have a full demonstration of the necessary capabilities to deploy complex systems for your fleet of Ubuntu Core Devices using Ansible and Docker snap.
What’s next?
You may realize that we assign the same password for all devices, this isn't the best case scenario, and for avoiding doing that, you may want to divide your inventory into two different ones and create rules and use patterns targeting hosts.
You may also notice that some values, such as `env_file` or the RabbitMQ image with its tag, are repeated multiple times within the playbook. While this is manageable for a few instances, it can become problematic when multiple locations refer to these values. To avoid duplication, it's recommended to use variables to centralize and manage these values more efficiently.

The next steps would be to expand this example by sending the data from the local brokers deployed in the gateways to a central cloud, like a RabbitMQ federation or a Canonical’s Managed OpenStack.
