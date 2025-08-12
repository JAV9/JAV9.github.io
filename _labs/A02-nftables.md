---
title: "Lab 2. Stateful Packet Inspection and Web Proxy with nftables & Tinyproxy"
excerpt: "In this lab, you'll configure a Linux firewall using nftables for stateful packet inspection, apply default drop policies, allow specific traffic between private and public networks, and deploy Tinyproxy as both a direct and transparent proxy."
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
  - nftables
  - tinyproxy
  - firewall
  - proxy
  - linux
  - security
  - stateful-inspection
  - tutorial
  - cybersecurity
---

## Environment

This lab builds upon the **environment B** setup from [Lab 0. Setting up the environment](/labs/a00-setting-up-environment/).

We have:

- **VM_1** – Internal host (`192.168.0.0/24`)
- **VM_2** – External host (`172.16.0.0/24`)
- **VM_3_fw** – Firewall with two NICs:
  - `enp0s3` → internal network
  - `enp0s8` → external network

---

## 1. Introduction & Objectives

Firewalls don’t just block or allow packets — they can **track connection states** to make decisions smarter and safer. In this lab we will:

1. Configure **nftables** to act as a stateful firewall.
2. Apply **default drop policies** to block everything unless explicitly allowed.
3. Allow only certain traffic between internal and external networks.
4. Install and configure **Tinyproxy** as:
   - A **direct HTTP proxy** (clients must configure it).
   - A **transparent proxy** (clients don’t even know it’s there).
5. Log rejected packets for troubleshooting and auditing.

By the end, you’ll have a firewall that can:
- Filter based on **connection state**.
- Serve HTTP requests **through a proxy**.
- Transparently redirect web traffic.

---



## 2. Enabling nftables and Setting a Default Policy

> Default **ACCEPT** policies are dangerous — you’re allowing traffic unless you *remember* to block it. We flip this: **deny everything by default, then explicitly allow only what’s needed**.

First, ensure `nftables` is installed and active:

```bash
sudo systemctl start nftables
sudo systemctl enable nftables
```

Edit `/etc/nftables.conf` to apply **default drop policies** and allow loopback traffic:

```bash
table inet filter {
    chain input {
        type filter hook input priority 0;
        policy drop;
        iif "lo" accept
    }
    chain output {
        type filter hook output priority 0;
        policy drop;
        oif "lo" accept
    }
    chain forward {
        type filter hook forward priority 0;
        policy drop;
    }
}
```

Apply the configuration:

```bash
sudo nft -f /etc/nftables.conf
```

> **Why loopback is allowed:** Without it, even internal OS services communicating with themselves would break.



## 3. Allowing Stateful Forwarding Rules

> We don’t want to blindly forward packets. Instead, we:
>
> - Allow internal hosts to start connections outwards.
> - Allow only *established* traffic back in.
> - Open specific services for external access.

Now we add rules for **stateful packet inspection**:

```bash
# Allow internal → anywhere
sudo nft add rule inet filter forward iif "enp0s3" accept

# Allow established/related traffic from outside → inside
sudo nft add rule inet filter forward iif "enp0s8" ct state established,related accept

# Allow HTTP & SSH from external → 172.16.0.1
sudo nft add rule inet filter forward iif "enp0s8" tcp dport {80, 22} ip daddr 172.16.0.1 accept

# Log and drop everything else
sudo nft add rule inet filter forward log prefix "DROP-FWD: " flags all
```



## 4. Testing Stateful Inspection

From **VM_1** (internal):

```bash
wget 172.16.0.2
ssh user@172.16.0.2
```

From the firewall, check connection tracking:

```bash
sudo conntrack -L
```

Example output:

```nginx
tcp      6 431999 ESTABLISHED src=192.168.0.2 dst=172.16.0.2 sport=50322 dport=80
tcp      6 431999 ESTABLISHED src=172.16.0.2 dst=192.168.0.2 sport=80 dport=50322
```

> **Why this matters:** The firewall *remembers* which connections are valid, so it can allow return traffic without opening unnecessary holes.



## 5. Anti-Spoofing Rules

> Attackers could forge IP headers to pretend they’re from a trusted network. Anti-spoofing rules stop this.

To prevent IP spoofing:

```bash
# Block external packets claiming to be from internal network
sudo nft add rule inet filter forward iif "enp0s8" ip saddr 192.168.0.0/24 log prefix "SPOOF-IN: " drop

# Block internal packets with unexpected source IP
sudo nft add rule inet filter forward iif "enp0s3" ip saddr != 192.168.0.0/24 log prefix "SPOOF-OUT: " drop
```



## 6. Direct Proxy Mode with Tinyproxy

> A **direct proxy** lets you control and log HTTP traffic from clients who configure it manually. This is the simplest deployment mode.

Install Tinyproxy:

```bash
sudo apt install tinyproxy
```

Edit `/etc/tinyproxy/tinyproxy.conf`:

```conf
Allow 192.168.0.0/24
```

Change log directory permissions:

```bash
sudo chown tinyproxy:tinyproxy /var/log/tinyproxy
```

Start Tinyproxy:

```bash
sudo systemctl start tinyproxy
```

Update firewall rules:

```bash
# Allow clients to connect to Tinyproxy
sudo nft add rule inet filter input iif "enp0s3" tcp dport 8888 accept

# Allow responses from proxy to clients
sudo nft add rule inet filter output oif "enp0s3" ct state established accept

# Allow proxy to fetch HTTP content externally
sudo nft add rule inet filter output oif "enp0s8" tcp dport 80 accept

# Allow responses from web servers
sudo nft add rule inet filter input iif "enp0s8" ct state established accept

# Log and drop anything else
sudo nft add rule inet filter input log prefix "DROP-IN: " drop
sudo nft add rule inet filter output log prefix "DROP-OUT: " drop
```

## 7. Testing Direct Proxy

From VM_1:

```bash
env http_proxy="http://192.168.0.3:8888" wget 172.16.0.2
```

Example Tinyproxy log (`/var/log/tinyproxy/tinyproxy.log`):

```bash
CONNECT   May 12 11:45:23 [1234]: Connect (file descriptor 6): 192.168.0.2
INFO      May 12 11:45:23 [1234]: Request (file descriptor 6): GET http://172.16.0.2/
```

## 8. Transparent Proxy Mode

> In transparent mode, clients don’t need proxy settings — the firewall silently redirects HTTP traffic to the proxy.

Add NAT prerouting chain:

```bash
sudo nft add table inet nat
sudo nft add chain inet nat prerouting { type nat hook prerouting priority -100; }
sudo nft add rule inet nat prerouting iifname "enp0s3" tcp dport 80 redirect to :8888
```

Now VM_1 can connect without proxy settings:

```bash
wget 172.16.0.2
```

## 9. View Final Ruleset

```bash
sudo nft list ruleset
```

Example output:

```python
table inet filter {
    chain input {
        type filter hook input priority filter; policy drop;
        iif "lo" accept
        iif "enp0s3" tcp dport 8888 accept
        ct state established accept
    }
    ...
}
```