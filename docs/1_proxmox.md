# ![](https://www.proxmox.com/images/proxmox/Proxmox_logo_standard_hex_600px.png)

## Installation

Installing [Proxmox](https://www.proxmox.com/en/proxmox-virtual-environment/get-started) is as simple as installing their latest ISO from their [Downloads page](https://www.proxmox.com/en/downloads), burning the image onto a USB Drive, booting into it and following the on-screen instructions. They also have great [documentation](https://pve.proxmox.com/pve-docs/chapter-sysadmin.html) of the whole installation and configuration processes.

## Configuration

The initial Proxmox configuration is relatively simple, but crucial to get right. That is because our Homelab has a unique quirk: as we mentioned before, we'll virtualize our router with pfSense inside of Proxmox. However, Proxmox - being a device in our network -, needs internet access through our router for many important reasons: NTP, updates, web configuration portal, etc., all of which do not work unless our pfSense VM is up and running - a sort of "network bootstrapping". 

To solve that, we assign static IP and network configurations for the Proxmox machine on the `vmbr0` bridge (our LAN bridge) instead of using DHCP, and we will reflect that with a static mapping on our pfSense settings. Thus, while Proxmox itself is booting, it will not have internet connectivity or a DCHP server to request an address lease, but once it spins up the pfSense VM - which we configure to turn on immediately on boot - it will get a connection and have all the access it needs.

This brings us to another point simple but important point: VM boot order. Proxmox has a really nifty feature, where we can choose not only whether to spin up a VM on boot, but the order in which to do so and the delay between each VM spinup. Thus, we configured the pfSense VM to immediately spin up on boot, and only after a 240-second delay to ensure it is fully up, we progressively turn on the other VMs that rely on networking being up to function. We will get to those in the later steps of the project. Here's the configuration for the pfSense VM:

![](../media/proxmox_pfsense_bootorder.png)

### Network

Our Proxmox machine will have 2 wired Ethernet connections: one coming from our ISP's modem - which will be our WAN -, and another which will go to our switch/WAP - which is our LAN. Since our mini-PC only has one built-in Ethernet adapter, we bought a USB-to-Ethernet adapter. We arbitrarily assigned our onboard Ethernet adapter to be our LAN, and the adapter one to be our WAN - perhaps because we care more about LAN stability and performance, rather than WAN, as many of our more network-intensive services are inward-facing.

Then, we make two Linux bridges, one for each adapter - `vmbr0` for LAN and `vmbr1` for LAN. This is what will be passed to the VMs, which provides them with a network connection. Then, as we mentioned above, we solve our "network bootstrapping" problem by statically assigning our network configuration for our Proxmox machine on the `vmbr0` (the LAN bridge) like so:

* IP: 10.10.0.50/16
* DNS: 10.10.0.1 *(our pfSense VM's IP)*
* Gateway: 10.10.0.1 *(our pfSense VM's IP)*
  
Here's a screenshot of our Proxmox network configuration:
![](../media/proxmox_network_config.png)

## Services and VMs



### Template VM

### pfSense

### PiHole

### TrueNAS

### HomeAssistant

### k3s
