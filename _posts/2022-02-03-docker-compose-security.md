---
layout: post
title:  "Docker Compose security controls"
date:   2022-02-03 16:31:19 +0200
categories: ['docker','container-security']
---
Docker Compose is used to define and deploy multiple containers. Several security controls can de defined within the *docker-compose.yaml* file which hardens the security state of the containers. This post explores which security controls are available, what risks they mitigate and how they can be defined in the *docker-compose.yaml* and *Dockerfile* to codify secure Docker containers. Docker implements some security controls by default, but this can be overwritten which could weaken the container as well, these configs are also documented. Most of these controls can also be set as parameters with the *docker* *run* command, but deploying them with Docker Compose helps with replicating secure deployment and code review in CI/CD. We start with a simple PHP web server container and progressively deploy and test controls. 
<!--more-->
## Base container
We start with a very basic PHP web server that mounts a volume with a simple homepage. We have a *Dockerfile* and *docker-compose.yaml* file.

*Dockerfile*
```docker
FROM php:8.0-apache
WORKDIR /var/www/html

```

**docker-compose.yaml**
```yaml
version: '3.8'
services:
    web:
        container_name: php-webserver
        build:
            context: ./
            dockerfile: Dockerfile
        volumes:
            - ./src/:/var/www/html
        ports:
            - 8000:80
```
We run ``sudo docker-compose up -d`` to build and run the container. Then we run the command below to check the status of the container.  
```bash
docker exec -it php-webserver /bin/bash
root@a6e966cb1f98:/var/www/html# 
``` 

## Docker security controls
This section runs through all the controls that we progressively introduce into the config.

### Non-Root User
Docker runs containers using the ``root`` user by default. This user maps to the same user on the host. If a container is compromised and an attacker could breakout of the Docker container this could give them root privileges on the underlying host. By declaring a low privileged user on the container this risk is mitigated.

We can create a low privileged user by adding the lines below to the the *Dockerfile*.
```Dockerfile
RUN useradd -u 8877 notroot
USER notroot
```
Run `` docker-compose up -d --build `` to rebuild the image before creating the container.

### No New Privileges
This control prevents processes from running inside a container from gaining new privileges. This is useful if some executables may have privilege escalation paths. For example this control would disallow a user to run a binary that has the setuid bit set with owner root to run as root.

To add this control, modify the *docker-compose.yaml* file and the ``no-new-privileges`` directive:
```yaml
version: '3.8'
services:
    web:
        container_name: php-webserver
        build:
            context: ./
            dockerfile: Dockerfile
        volumes:
            - ./src/:/var/www/html
        ports:
            - 8000:80
        security_opt:
            - no-new-privileges:true

```
We can check which security options have been applied by running the command below:
```bash
docker inspect php-webserver --format '{{ .Id }}: SecurityOpt={{ .HostConfig.SecurityOpt }}'
5603e1c81da6067298bdf3fcc009efaba80bc2edf22bba17a4ef679075a15ca1: SecurityOpt=[no-new-privileges:true]
```

### Limit Capabilities
Linux capabilities extend the traditional root/non-root binary permissions model into more granular permissions. With capabilities executables and processes can be assigned limited privileges to execute kernel calls only to the degree that their functionality requires it. Docker containers are prefconfigured with a limited set of capabilities but not all of them are always necessary and which contributes to a larger attack surface. We can explicitly disable these capabilities in the container.

Modify the *docker-compose.yaml* file to drop all cababilities using the ``cap_drop`` directive:
```yaml
#removed for brevity
        security_opt:
            - no-new-privileges:true
        cap_drop:
            - all
```
To check which capabilities have been dropped or added we can use ``docker inspect``:
```bash
docker inspect php-webserver --format '{{ .Id }}: CapAdd={{ .HostConfig.CapAdd }}'
2bb0750e5bce067e2994a72788c3210e5be0afa407ebd7a2dc9e2fbe155193f4: CapAdd=<no value>
docker inspect php-webserver --format '{{ .Id }}: CapDrop={{ .HostConfig.CapDrop }}'
2bb0750e5bce067e2994a72788c3210e5be0afa407ebd7a2dc9e2fbe155193f4: CapDrop=[all]

```

In some cases removing capabilities might hinder legitimate functioning of applications. In that case it is possible to add specific capabilities using ``cap_add`` together with ``cap_drop``. Another possibility is using ``cap_drop`` with specific capabilities instead of all.

This is demonstrated in the example below where all capabilities are dropped and the ``NET_RAW`` capability is added.

```yaml
#removed for brevity
        cap_drop:
            - all
        cap_add:
            - NET_RAW
```

### Resource Quotas and Limits  
Docker containers can consume all processing and memory availability on host unless constraints are specifically specified. This is risky as certain processes can overload the host effectively causing a DoS for services on the host. We can limit the amount of resources that a container can consume using ``limits`` and reserve CP and memory using ``reservations``.

We can confirm the current resource consumption of a service using docker stats.
```bash
docker stats php-webserver --no-stream
CONTAINER ID   NAME            CPU %     MEM USAGE / LIMIT     MEM %     NET I/O           BLOCK I/O     PIDS
08b26c81353e   php-webserver   0.02%     12.54MiB / 3.834GiB   0.32%     5.96kB / 1.31kB   3.61MB / 0B   7

```

Modify the *docker-compose.yaml* file. This will ensure that the container does not use up more then 50% of 1 core on the host CPU and 512MB memory of RAM. Furthermore, 25% of CPU power and 128Mb memory will always be dedicated to the container.
```yaml
#removed for brevity
cap_add:
    - NET_RAW
deploy:
    resources:
        limits:
            cpus: 0.50
            memory: 512M
        reservations:
            cpus: 0.25
            memory: 128M
```


Limits and reservations are not supported for V3 Docker without using Swarm, so in order to use it you have to run the ``--compatibility`` flag when using docker-compose:
```bash
docker-compose --compatibility up --build -d
```

### Privileged Mode
Containers can be configured to allow privileged access on the host. This is **not** recommended as it increases the attack surface on the underlying host and weakens the sandboxing. By default privileged access is disabled so this key value pair does **not** have to be explicitly set in the config. However, if there is a legit usecase for this it is recommended to lockdown the user using ``user namespace remapping``. This is to limit the privileges that a user on the container will have on the underlying host. This is configured by setting the ``privileged directive``.

```yaml
privileged: true
```

### Set Filesystem and Volume to read-only
Restricting write-access on containers and volumes will prevent malicious files from being written to ephemeral and persistent storage. This reduces the attack surface of the container. Careful consideration should be given to the context of the container when doing this to ensure all services can still function as normal. 

We make a few changes:
1. Volume Read-Only - We append ``:ro`` to the volume path use the short-hand syntax to mount the volume with readonly access.
2. Container Read-Only - Folders outside of the volume path are still write-able so we need to configure read only access for the whole container using the ``read_only`` directive.
3. Whitelist writeable directories - The apache service needs write access to specific directories, if it cannot write the service will not start. We use ``tmpfs`` to define directories that are writeable.

*docker-compose.yaml* file modifications
```
#removed for brevity
read_only: true
tmpfs:
    - /tmp
    - /run/apache2
    - /run/lock
volumes:
    - ./src/:/var/www/html:ro
#removed for brevity
```

### Environment Variables

Environment variables are shared with any container linked to the container and visible from docker inspect, accessible by any process in the container. This makes them risky to store sensitive data such as secrets or tokens. There are a few workarounds for it discribed below.

1. **docker secrets** - Docker has built-in support for secrets but this requires a swarm cluster. 
2. **Fake docker secrets using bind mount and docker-compose**-
This approach requires creating a seperate file for each secret. When running docker-compose the secret will be stored on the container and can be accessed during runtime at ``/run/secrets/<secret-name>``. Applications can be configured to read the file content in that directory. If the *Dockerfile* is commited to the repo, make sure to exclude these secrets using gitignore.
This approach is only *slightly* better then setting them plainly as environment variables, as they can no longer be seen with docker inspect. However anyone that can do docker exec or view /proc/ can still see the secrets. 
```yaml
version: '3.8'
services:
    web:
        container_name: php-webserver
        build:
            context: ./
            dockerfile: Dockerfile
#removed for brevity
        secrets:
            - super_secret_password
#removed for brevity
secrets:
    super_secret_password:
        file: ./password
```
3. **Fetch secrets from secure external location** -
Store secrets in a key management system like Azure KeyVault or AWS KMS or HashiCorp Vault. This requires a lot more planning and testing and settting up permissions from the orchestrator to the secret storage location. Its a bit out of scope for this article. However I found good resource that explains this in more depth and how HashiCorp Vault solves the problem. This involves always encrypting the secret and only making it plaintext accessible in memory. - [Secrets with vault and Nomad](https://www.hashicorp.com/resources/securing-container-secrets-vault)

### Disable Inter Container Communication
Containers that are created with docker run are configured by default in the default bridge network. These containers can communicate by default on default bridge network. By setting icc=false we disable this capability. Historically, explicit links had to be configured to explicitly allow communication between specific containers. Links are now considered legacy functionality and deprecated. User defined networks are the prefered way to define network trust between containers. However, docker-compose creates a seperate network from the default bridge network to launch containers in. This is already a good start for segmentation of networks, but under certain circumstance it might still be good to follow a user defined approach. Especially for deployments consisting of a large number of distinct service that don't all need to communicate with each other.

### Seccomp
Similar to capabilities in the sense that it limits access to kernel functionality. Its different because instead of removing capabilities entirely, it acts as a firewall for syscalls, limiting which requests can be made to the kernel. Docker sets a default seccomp profile but this can be replaced by a custom profile setting ``seccomp`` in ``security_opt``.

First we check if docker was built with seccomp and the kernel is configured correctly:
```bash
grep CONFIG_SECCOMP= /boot/config-$(uname -r)
CONFIG_SECCOMP=y
```

*docker-compose.yaml*
```yaml
#redacted for brevity
security_opt:
    - no-new-privileges:true
    - seccomp:./seccompcustom.json
#redacted for brevity
```

From the docs [Seccomp actions](https://man7.org/linux/man-pages/man3/seccomp_init.3.html) its explains ``SCMP_ACT_LOG`` will not block filtered rules but only log the syscalls made that are not whitelisted. This is different to ``SCMP_ACT_ERRNO`` which will actively block syscalls.

An example profile that logs syscalls used - this could be useful to investigate which calls are needed for legitimate operation when setting up a block list. 

*seccompcustom.json*
```json
{
    "defaultAction": "SCMP_ACT_LOG"
}
```
A full example profile that blocks syscalls used that is not in the whitelist.

*seccompcustom.json*
<!--Why ping works - https://unix.stackexchange.com/questions/617927/why-ping-works-without-capability-and-setuid-->
```json
{
    "defaultAction": "SCMP_ACT_ERRNO",
    "architectures": [
        "SCMP_ARCH_X86_64",
        "SCMP_ARCH_X86",
        "SCMP_ARCH_X32"
    ],
    "syscalls": [
        {
            "names": [
                "accept4",
                "epoll_wait",
                "pselect6",
                "futex"
            ],
            "action": "SCMP_ACT_ALLOW"
        }
    ]
}
```

It is also possible to remove the standard profile which is **not** recommended but can be identified in the *docker-compose.yaml* file as
```yaml
security_opt:
    - seccomp:unconfined
```

### AppArmor
Can be used to limit kernel access on per process basis. Docker has a default configuration profile that can be referenced using ``docker-default``. 

We can specify a apparmor profile under security_opt. Here we are referencing the default apparmor profile that ships with Docker installation.

```yaml
security_opt:
    - no-new-privileges:true
    - seccomp:./seccompcustom.json
    - apparmor:docker-default
```

Contents of custom-profile that blocks ping by denying icpm and raw packets.
```
#include <tunables/global>

profile custom-profile flags=(attach_disconnected,mediate_deleted) {
  #include <abstractions/base>

  network inet tcp,
  network inet udp,
  
  deny network icmp,
  deny network raw,
  deny network packet,
  file,
  mount,
}

```
When a custom profile is used it must first be loaded into AppArmor before it can be referenced in the *security_opt* setting:
```bash
 apparmor_parser -r -W ./custom-profile
```
We can confirm that the profile is loaded with
```bash
apparmor_status | grep custom-profile
```


Then we can reference it using:
```yaml
security_opt:
    - apparmor:custom-profile
```
## Complete files
Here is an example of all the security controls we configured for the deployment.

*Dockerfile*
```docker
FROM php:8.0-apache
WORKDIR /var/www/html
RUN apt-get update
RUN apt-get install iputils-ping -y
RUN useradd -u 1337 notroot
USER notroot
```

*docker-compose.yaml*
```yaml
version: '3.8'
services:
    web:
        container_name: php-webserver
        build:
            context: ./
            dockerfile: Dockerfile
        read_only: true
        tmpfs:
            - /tmp
            - /run/apache2
            - /run/lock
        volumes:
            - ./src/:/var/www/html:ro
        secrets:
            - super_secret_password
        ports:
            - 8000:80
        security_opt:
            - no-new-privileges:true
            - seccomp:./seccompcustom.json
            - apparmor:custom-profile
        cap_drop: #this is a comment
            - all
        deploy:
            resources:
                limits:
                    cpus: 1
                    memory: 512M
                reservations:
                    cpus: 0.25
                    memory: 128M
secrets:
    super_secret_password:
        file: ./password
```
## Closing
We explored different security options that can be configured in a Docker Compose. It is not always possible to combine add all controls, some require more in-depth investigation to determine how to facilitate proper functioning of applications. This serves as a reference guide and starting point for implementing Docker security when deploying with Docker Compose.