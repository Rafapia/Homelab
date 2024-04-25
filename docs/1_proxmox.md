# ![](../media/proxmox_logo.png)

## Installation

Installing [Proxmox](https://www.proxmox.com/en/proxmox-virtual-environment/get-started) is as simple as installing their latest ISO from their [Downloads page](https://www.proxmox.com/en/downloads), burning the image onto a USB Drive, booting into it and following the on-screen instructions. They also have great [documentation](https://pve.proxmox.com/pve-docs/chapter-sysadmin.html) of the whole installation and configuration processes.

## Configuration

The initial Proxmox configuration is relatively simple, but crucial to get right. That is because our Homelab has a unique quirk: as we mentioned before, we'll virtualize our router with pfSense inside of Proxmox. However, Proxmox - being a device in our network -, needs internet access through our router for many important reasons: NTP, updates, web configuration portal, etc., all of which do not work unless our pfSense VM is up and running - a sort of "network bootstrapping". 

To solve that, we assign static IP and network configurations for the Proxmox machine on the `vmbr0` bridge (our LAN bridge) instead of using DHCP, and we will reflect that with a static mapping on our pfSense settings. Thus, while Proxmox itself is booting, it will not have internet connectivity or a DCHP server to request an address lease, but once it spins up the pfSense VM - which we configure to turn on immediately on boot - it will get a connection and have all the access it needs.

This brings us to another point simple but important point: VM boot order. Proxmox has a really nifty feature, where we can choose not only whether to spin up a VM on boot, but the order in which to do so and the delay between each VM spinup. Thus, we configured the pfSense VM to immediately spin up on boot, and only after a 240-second delay to ensure it is fully up, we progressively turn on the other VMs that rely on networking being up to function. We will get to those in the later steps of the project. Here's the boot order configuration for the pfSense VM:

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

To keep all the documentation organized, we'll cover the Proxmox side of the setup of all the following services here, and the remainder of the setup in the service's respective platform in their own documents.

### pfSense

TODO

Pass CPU flags

### Template VM

One of the great practicalities of having a Proxmox server running is how quickly we can spin up a VM - whether that is for deploying some permanent service or just testing. And with each VM completely isolated from the other, we risk very little in terms of breaking anything we've already deployed.  

To make this process even faster and effortless, we can make a template VM: that is, a VM that we can't actually turn on, but can quickly clone into separate VMs which are all pre-configured the way we want. This way, we don't have to go through all the installation and configuration processes of an OS - like Ubuntu - every time we want to create a quick VM.

For this, we'll create a Ubuntu Server template and make use of the built-in [CloudInit](https://cloud-init.io/) functionality on Proxmox. With it, we can pre-configure this template to have our Linux user, SSH keys, and networking configurations, and let CloudInit take care of the rest.

While we did this process manually, we found this bash script that does essentially what we did, which we modified and included below for brevity:

```bash
# Download fresh copy of the image. Delete if already installed.
cd
rm -f jammy-server-cloudimg-amd64.img
wget -q https://cloud-images.ubuntu.com/jammy/current/jammy-server-cloudimg-amd64.img

# Create VM
sudo qm create 9000 --name "ubuntu-2204-cloudinit-template" --memory 2048 --cores 2 --net0 virtio,bridge=vmbr0
sudo qm importdisk 9000 jammy-server-cloudimg-amd64.img local-lvm
sudo qm set 9000 --scsihw virtio-scsi-pci --scsi0 local-lvm:vm-9000-disk-0
sudo qm set 9000 --boot c --bootdisk scsi0
sudo qm set 9000 --ide2 local-lvm:cloudinit
sudo qm set 9000 --serial0 socket --vga serial0
sudo qm set 9000 --agent enabled=1
sudo qm template 9000

# Delete image, not needed anymore.
rm -f jammy-server-cloudimg-amd64.img
```
(Script from [here](https://austinsnerdythings.com/2023/01/10/proxmox-ubuntu-22-04-jammy-lts-cloud-image-script/))

Now, we can finish setting up our CloudInit options on the Proxmox UI under our new VM:

![](../media/proxmox_cloudinit.png)

and every time a new VM is made with this template, it will already have the correct linux user created, with our password, and our SSH keys preinstalled; all we have to do is SSH into the machine and start our new project!

### PiHole

TODO

### TrueNAS

We first created another VM in Proxmox and set it to boot using the ISO image downloaded from the following [link](https://www.truenas.com/download-truenas-scale/).

Since [TrueNAS Core](https://www.truenas.com/truenas-core/), like pfSense, is based on FreeBSD and not on Linux, an important step in the VM setup is to change the *OS Type* to *Other*:

![](../media/proxmox_truenas_OS.png)

Also, we want the TrueNAS VM to have full access to the disk we installed for it - that iss, we need to pass through the disk to the VM on our hypervisor. For that, we followed [these steps](https://pve.proxmox.com/wiki/Passthrough_Physical_Disk_to_Virtual_Machine_(VM)) from Proxmox's documentation to pass through our physical disk to the VM we just created without a PCIe controller. We can confirm that the disk was added by checking the Hardware tab for the VM in Proxmox:

![](../media/proxmox_truenas_disk.png)

Now, we just need to turn the VM on and follow the TrueNAS installation, which we documented [here](4_truenas.md#installation)

### HomeAssistant

To set up HomeAssistane, we first logged in to our Proxmox machine as *root*. Then, we downloaded the `.qcow2` HomeAssistant disk from the [HomeAssistant's guide](https://www.home-assistant.io/installation/alternative#install-home-assistant-operating-system) with:

```bash
cd
wget https://github.com/home-assistant/operating-system/releases/download/12.2/haos_ova-12.2.qcow2.xz
```
Then, we created a new VM with no local disk:

![](../media/proxmox_homeassistant_OS.png)

From there, under the *System* tab, we change the BIOS to UEFI, select the EFI Storage, and **disable** *Pre-enrolled keys*, or otherwise the HomeAssistant OS does not boot:

![](../media/proxmox_homeassistant_System.png)

Then, on the *Disks* tab, we delete the *scsi0* default disk since we downloaded it as we'll be importing the one we downloaded previously from the HomeAssistant guide. After that, we give the VM 2 CPU cores and 2GB of RAM - as these are the minimum recommended specs, as can be seen [here](https://www.home-assistant.io/installation/alternative#create-the-virtual-machine) -, and finally we select `vmbr0` as our network (as we established that this is our LAN bridge).

Optionally, we can also set this VM to turn on on boot, so it will automatically spin up. This is recommended, as we might depend on it for many home functions.

Our VM setup is almost done, all we now have to do is attach our downloaded disk to the VM. We first extract the *.xz* file we downloaded and import it onto the VM with

```bash
unxz haos_ova-12.2.qcow2.xz
qm importdisk <VM_ID> haos_ova-12.2.qcow2.xz local-lvm
```

Now, all we have to do is turn on the VM and follow HomeAssistant's Installation, which we go over [here](5_homeassistant.md#instalaltion).

<!-- ### k3s

TODO -->