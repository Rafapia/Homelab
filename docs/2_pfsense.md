# ![](../media/pfsense_logo.png)

## Dashboard

Here's a sneak peek of our final pfSense dashboard, after applying all the configurations we go over in this document:

![](../media/pfsense_final_dashboard.png)

## Installation

Installation is made very simple by following Netgate's guide to virtualize pfSense with Proxmox, which can be found [here](https://docs.netgate.com/pfsense/en/latest/recipes/virtualize-proxmox-ve.html). As will not cover the installation steps as we would not do justice to Netgate's resources.

## General Setup

Then, once pfSense is installed, we once again spin up the VM and do some initial configuration through its command line:
![](../media/pfsense_cml.png)

First, as discussed in the [Proxmox](1_proxmox.md) section, we set the `vmbr0/vtnet0` bridge to be the LAN interface and `vmbr1/vtnet1` to be the WAN interface in pfSense. Then, since we want our router to also have a static IP address for obvious reasons, we specify the static IP `10.10.0.1/16` on the LAN interface (`vmbr0/vtnet0`).

Lastly, we reset our *webConfigurator* password, so we don't have the default `admin:pfSense` username and password. This is only temporary, as later we will create another user with admin privileges, and disable the default admin user login.

From there, we just follow the initial [setup wizard](https://docs.netgate.com/pfsense/en/latest/config/setup-wizard.html) to set the basic configurations for pfSense:

![](../media/pfsense_wizard.png)

We kept our hostname as `pfSense` but changed our domain to the one we bought from Cloudflare for the later stages of the project. For documentation purposes, let's refer to our domain name as *homelab.com*. This then makes our pfSense VM have the FQDN of *pfsense.homelab.com*.

Finally, we'll leave the *Primary DNS Server* empty for now, but once we set up our *PiHole* ad-and-traffic-blocking DNS service, we'll put its static IP address here and a public DNS as backup, which will direct all DNS queries from the entire network to PiHole, as long as we set pfSense's IP as the DNS on all of our DHCP configurations - thus achieving a network-wide ad blocking and advertisement tracking blocking. We recognize that the is an argument about whether blocking ads is a good practice or not, but setting up PiHole does not necessarily mean we will block all ads, as all this depends on what block lists we add to our PiHole instance. One can just as well only add lists of known malicious domains, but allow safe ads to be shown.

## Configurations

After going through the wizard, we have yet a few other modifications we'll need.

First, we need to free up port 443, or the default `HTTPS` port. It is currently being used for the *webConfigurator* portal for pfSense, but we will need it to be free for `HAProxy`, which we'll use to host multiple services through the domain we bought through Cloudflare. And since we're here, let's force the *webConfigurator* to work only through HTTPS (which is much safer, even if we're using our self-signed pfSense certificate) and disable the redirect rule. We'll need to remember we're now using `pfsense.homelab.com:10443` as our *webConfigurator* address from now on (we recommend taking notes of these addresses and ports and *highly* recommend using something like [Trotto go/links](https://www.trot.to/) for ease of use, as this will not be the only service that runs on a custom port which needs to be remembered).

![](../media/pfsense_webconfigurator_port.png)

### DHCP

The next step is to set up our DCHP server, which is simple. All we do is go to **Services > DHCP Server > LAN**, and enable it with the following settings:

![](../media/pfsense_dchp.png)

### Static IP reservations

As mentioned many times, we will assign static IP addresses for all of our core devices and services that we will need to address in the future: our Proxmox server, pfSense, PiHole VM, TrueNAS VM, HomeAssistant VM, our PC (for remote access, so we can access all our resources from our laptop from anywhere), etc. All of these static mappings live here.

![](../media/pfsense_dchp_leases.png)

### DNS

While DNS is not strictly speaking a hard concept to understand, or a hard part of the project to implement, it is more often than not the source of many of the problems we faced in our Homelab project, and it is generally very hard to trace.

Here's a beautiful *haiku* we found about this, which helps us always remember to check DNS before we find ourselves too deep into trying to solve a seemingly unrelated and unsolvable problem:

![](https://www.cyberciti.biz/media/new/cms/2017/04/dns.jpg)

#### DNS Resolver

[**DNS Resolver**](https://docs.netgate.com/pfsense/en/latest/services/dns/resolver.html) is one of the services available in pfSense, which "utilizes `unbound`, which is a validating, recursive, caching DNS resolver that supports DNSSEC, DNS over TLS, and a wide variety of options. It can act in either a DNS resolver or forwarder role".

We will use the **DNS Resolver** to make some DNS entries, so that we can have local addresses to our devices/VMs, and so we can resolve hosts by their hostname automatically, either from their DCHP lease, static mapping, or later by their hostname in OpenVPN. As an example, say we have a TV with hostname `rokutv`. Then, in HomeAssistant, for example, we can refer to the TV by its hostname rather than its IP address, without having to manually add a DNS entry for every device - this is done automatically for us.

Strictly speaking, this could be done in PiHole, as it also has a DNS Resolver. However, it is somewhat limited, and we prefer to keep a separation of duties, where PiHole will only handle DNS blocking, and pfSense will handle all the local hostname resolution - so we keep all network-related configurations in one place. Thus, if we think about the path a DNS request makes in our network, it will be:

**Device making request > pfSense's DNS Resolver > PiHole > Upstream DNS (Google's or Quad9)**

As we mentioned in the Wizard section, we also added a public DNS service as a secondary DNS server in the General Configuration in case our PiHole fails, so we don't take down the entire network because one VM is down.

#### PiHole

TODO

#### DDNS

Since we're using Cloudflare as our DNS provider, to keep our "*homelab.com*" record always pointing to our home IP address, all we have to do is navigate to **Services > Dynamic DNS** and click **Add**. From there, we just have to fill out the fields with our Cloudflare information, as pfSense instructs:

![](../media/pfsense_ddns_setup.png)

Since we are hosting all of our services from home, it is easier to make all of our subdomains automatically point to our home IP address with a single DDNS daemon. We can do this by putting a `@` on the *Hostname* field, followed by our *homelab.com* address on our domain field. If we were hosting other services in other places - like AWS, GCP, or anything else outside of our home - we could create one DDNS entry specifically for each subdomain hosted at home, so instead of `@` we could put something like `homeassistant`, if we wanted to access our HomeAssistant service from `homeassistant.homelab.com`.

Another important thing is to enable the **Cloudflare Proxy**, which is supported out-of-the-box by pfSense. As the description states, this makes all of our services' traffic be routed to Cloudflare's servers before going to the requesting client. This has the benefits of hiding our home IP from the public, making use of Cloudflare's DDoS protection, and is fully encrypted between home and Cloudflare, where it will apply our TLS/SSL certificates there before forwarding the connection to the clients. And since we also bought our domains from Cloudflare, they automatically issue trusted TLS/SSL certificates for our "*homelab.com*", which makes our connection encrypted from end to end and very secure.

With that done, we should have a successfully set up DDNS service which appears like so:

![](../media/pfsense_ddns.png)

### VPN

One of the benefits of building a Homelab like we're doing is that we can host our own VPN, which allows us to connect back home and have full access to our devices without exposing or forwarding any ports on our router. For example, as a college student, this has been extremely helpful for working on homework and projects that require more computational power - we could simply connect to our VPN and safely SSH into our PCs. This, combined with our encrypted and locked-down VPN service, makes for a much safer way of connecting back home and greatly reduces the risk of unauthorized parties having access to our devices and data. Another benefit is that we can also forward *all* traffic to our network when connected from somewhere else, which will also forward all DNS traffic to our home router, which will keep filtering and blocking undesired requests.

#### OpenVPN

TODO

#### HAProxy

TODO

#### ACME Certificates

TODO
