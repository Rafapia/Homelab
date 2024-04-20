# ![](https://www.endpointdev.com/blog/2020/12/pihole-great-holiday-gift/pihole-logo.png)

## Installation

As described in Wikipedia, [PiHole](https://pi-hole.net/) is "a Linux network-level advertisement and Internet tracker blocking application which acts as a DNS sinkhole and optionally a DHCP server, intended for use on a private network. It is designed for low-power embedded devices with network capability, such as the Raspberry Pi, but can be installed on almost any Linux machine." ([source](https://en.wikipedia.org/wiki/Pi-hole)). Since we're already running a much more capable DCHP server on our pfSense VM, we will use PiHole only as a DNS sinkhole for ad and/or telemetry blocking.

While originally designed to run on a RaspberryPi - hence its name - we will run a containerized version of PiHole in a dedicated VM on our Proxmox server. This VM will simply be a Ubuntu Server instance of the VM template we built in the [Proxmox](1_proxmox.md) stage of the project, and will be responsible for running all of our Docker containers in our Homelab.

### Docker

Once we have the VM up and running, we will first need to install Docker. Here, we just followed the official installation instructions available on [Docker's official documentation](https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository).

### Portainer

Then, for ease of use, we will also install [Portainer](https://docs.portainer.io/start/install-ce/server/docker/linux), a "container management software to deploy, troubleshoot, and secure applications across cloud, datacenter, and Industrial IoT use cases". It is just another container that runs in the same Docker environment as we'll run PiHole, which provides a really nice GUI for container management and has support for [Docker Compose](https://docs.docker.com/compose/) through its "[stacks](https://docs.portainer.io/user/docker/stacks)", both of which will make our lives easier.

Once installed, our Portainer GUI looks like this:

![]()

### Docker compose

```yaml
version: "3"

# More info at https://github.com/pi-hole/docker-pi-hole/ and https://docs.pi-hole.net/
services:
  pihole:
    container_name: pihole
    image: pihole/pihole:latest
    # For DHCP it is recommended to remove these ports and instead add: network_mode: "host"
    ports:
      - "53:53/tcp"
      - "53:53/udp"
      - "67:67/udp" # Only required if you are using Pi-hole as your DHCP server
      - "80:80/tcp"
    environment:
      TZ: 'America/Chicago'
      # WEBPASSWORD: 'set a secure password here or it will be random'
    # Volumes store your data between container upgrades
    volumes:
      - './etc-pihole:/etc/pihole'
      - './etc-dnsmasq.d:/etc/dnsmasq.d'
    #   https://github.com/pi-hole/docker-pi-hole#note-on-capabilities
    cap_add:
      - NET_ADMIN # Required if you are using Pi-hole as your DHCP server, else not needed
    restart: unless-stopped
```
*Source: https://github.com/pi-hole/docker-pi-hole*

## Setup

### Lists
