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

We can then deploy this stack and access our Pi-hole server using the address we set for it in pfSense [here](2_pfsense.md#pihole). 

## Pi-hole Setup

Our Pi-hole Dashboard looks like this (after adding several Adlists, which we'll go over below):

![](https://github.com/Rafapia/Homelab/assets/36646488/b5cd1012-0953-49fa-9625-0cb4b59240bd)

### Lists

For now, all we did was simply add several URLs/domains to our Adlists, which we quickly did by pasting the following list of space-separated addresses:

https://raw.githubusercontent.com/StevenBlack/hosts/master/hosts https://raw.githubusercontent.com/PolishFiltersTeam/KADhosts/master/KADhosts.txt https://raw.githubusercontent.com/FadeMind/hosts.extras/master/add.Spam/hosts https://v.firebog.net/hosts/static/w3kbl.txt https://adaway.org/hosts.txt https://v.firebog.net/hosts/AdguardDNS.txt https://v.firebog.net/hosts/Admiral.txt https://raw.githubusercontent.com/anudeepND/blacklist/master/adservers.txt https://s3.amazonaws.com/lists.disconnect.me/simple_ad.txt https://v.firebog.net/hosts/Easylist.txt https://v.firebog.net/hosts/Easyprivacy.txt https://v.firebog.net/hosts/Prigent-Ads.txt https://raw.githubusercontent.com/FadeMind/hosts.extras/master/add.2o7Net/hosts https://raw.githubusercontent.com/crazy-max/WindowsSpyBlocker/master/data/hosts/spy.txt https://hostfiles.frogeye.fr/firstparty-trackers-hosts.txt https://raw.githubusercontent.com/DandelionSprout/adfilt/master/Alternate%20versions%20Anti-Malware%20List/AntiMalwareHosts.txt https://osint.digitalside.it/Threat-Intel/lists/latestdomains.txt https://s3.amazonaws.com/lists.disconnect.me/simple_malvertising.txt https://v.firebog.net/hosts/Prigent-Crypto.txt https://raw.githubusercontent.com/FadeMind/hosts.extras/master/add.Risk/hosts https://bitbucket.org/ethanr/dns-blacklists/raw/8575c9f96e5b4a1308f2f12394abd86d0927a4a0/bad_lists/Mandiant_APT1_Report_Appendix_D.txt https://phishing.army/download/phishing_army_blocklist_extended.txt https://gitlab.com/quidsup/notrack-blocklists/raw/master/notrack-malware.txt https://v.firebog.net/hosts/RPiList-Malware.txt https://v.firebog.net/hosts/RPiList-Phishing.txt https://raw.githubusercontent.com/Spam404/lists/master/main-blacklist.txt https://raw.githubusercontent.com/AssoEchap/stalkerware-indicators/master/generated/hosts https://urlhaus.abuse.ch/downloads/hostfile/ https://zerodot1.gitlab.io/CoinBlockerLists/hosts_browser https://pgl.yoyo.org/adservers/serverlist.php?hostformat=hosts&showintro=0&mimetype=plaintext https://raw.githubusercontent.com/FadeMind/hosts.extras/master/UncheckyAds/hosts https://raw.githubusercontent.com/bigdargon/hostsVN/master/hosts

## Test

After adding all of the above URLs, we can see that we now block over 800,000 domains and can test whether Pi-hole is working using an [Ad Block Test](https://d3ward.github.io/toolz/adblock). 

Without Pi-hole, we get the following results:

![](https://github.com/Rafapia/Homelab/assets/36646488/7f85e635-4df0-4d3a-82dd-ed2c5a5f3d8a)

And with Pi-hole, we get:

#### TODO
![]()

As you can see, most ads are being blocked and we can add any additional list of domains to our AdLists as we see fit.

