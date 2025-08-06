---
title: "Lab 1. Network protocol attacks"
layout: single
classes: toc-right
author_profile: false
toc: true
toc_sticky: true
sidebar:
  nav: "labs"
collection: labs
categories: 
  - custom-labs
tags: 
  - tutorial
  - arp
  - spoofing
  - nmap
---
## Enviroment

This lab builds upon the environment setup from [Lab 0. Environment setup](/labs/a00-setting-up-enviroment/). All attacks will be launched from **VM_2**.

> ⚠️ **Warning:** While Wireshark is extremely useful for analyzing network traffic, it’s not recommended to run it during *flood* attacks due to the high volume of packets it will attempt to capture. This can cause performance issues or even crash the interface.

> 💡 **Tip:** If you're running Wireshark in VirtualBox and not seeing any traffic, make sure promiscuous mode is enabled:
>
> `Devices → Network → Network Settings → Adapter → Advanced → Promiscuous Mode: Allow All`

## Link layer

### ARP Spoofing

#### What is ARP Spoofing?

The Address Resolution Protocol (ARP) maps IP addresses to MAC (hardware) addresses in local networks. ARP spoofing exploits the lack of authentication in this process: an attacker can send forged ARP responses to mislead devices about the MAC address of other machines on the network. This allows the attacker to intercept, modify, or block traffic—effectively performing a **Man-in-the-Middle (MitM)** attack.

#### Objective

We’ll trick two machines on the network (typically a victim and its default gateway) into thinking that **VM_2** is the other party. This allows us to intercept all traffic between them.

#### Step 1: Enable IP Forwarding on VM_2

To avoid breaking communication between the victim and the gateway (which would turn this into a DoS attack), we need to enable IP forwarding so VM_2 can relay packets:

```bash
sudo sysctl -w net.ipv4.ip_forward=1
```

This makes VM_2 behave like a router, forwarding packets between interfaces.

#### Step 2: Launch ARP Spoofing Attack with `arpspoof`

We’ll use the `arpspoof` tool from the `dsniff` package to carry out the attack. You may need to install it first:

```bash
sudo apt install dsniff
```

Now launch the attack from **VM_2** with:

```bash
sudo arpspoof -t 192.168.0.1 -r 192.168.0.3
```

* `-t 192.168.0.1`: This tells `arpspoof` to target the system at IP `192.168.0.1` (e.g., the router).

* `-r 192.168.0.3`: This option tells it to **also** poison the other system (e.g., the victim machine).

Using both `-t` and `-r` enables **bidirectional poisoning**, allowing you to intercept traffic flowing **both ways**.

> By default, if you omit `-t`, `arpspoof` will poison *all* machines on the network, which could be useful (or risky) in larger scenarios.

#### Step 3: Observe ARP Table Changes

On both **VM_1** and **VM_3**, run the following to inspect the ARP table:

```bash
ip neigh
```

You should notice that the MAC address for the other party (router or victim) has been replaced by VM_2’s MAC address—proof that the ARP poisoning is working.

#### Step 4: Monitor Intercepted Traffic with Wireshark

With ARP spoofing in place, run Wireshark on VM_2 and start capturing packets on the correct network interface (`enp0s3`).

Apply the following display filter to focus on traffic between the victim and the router:

```ini
ip.addr == 192.168.0.1 && ip.addr == 192.168.0.3
```

You should now see packets flowing *through* VM_2, even though VM_2 is not the intended destination.

## Network Layer

### ICMP/ping flood

## Transport Layer

### TCP SYN flood

## Network enum

### System discovery

### Port scanning

### Service detection

## SO detection
