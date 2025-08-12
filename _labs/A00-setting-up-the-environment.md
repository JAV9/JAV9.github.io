---
title: "Lab 0. Setting up the Environment"
excerpt: "Step-by-step guide to preparing the virtual environment for our custom labs."
layout: single
collection: labs
classes: wide
author_profile: false
toc: true
toc_sticky: false
sidebar:
  nav: "labs"
categories: 
  - custom-labs
  - environment
  - setup
tags: 
  - tutorial
  - virtualbox
  - ubuntu
  - virtual machine
---

## Overview
This post explains how to set up the virtual environments needed for our custom labs.  
We’ll prepare a **base configuration** that will be reused in future exercises, ensuring everyone starts with the same working setup.

While you can adapt the environment to your preferred tools, this guide uses the following:

- **Virtualization platform:** Oracle VirtualBox
- **Base image:** *ubuntu-jammy* (OVA or ISO)

We’ll work with **three virtual machines**. Instead of importing the OVA multiple times, we’ll use **linked clones** to save disk space and speed up setup.

---

## Step 1 – Create the Master VM

1. Import the Ubuntu OVA or install it from ISO.
2. Open the **VM Settings → Network** tab.
3. Enable **two network adapters**.
4. Set both adapters to the same **Host-only network** (name: `local`).
5. Enable **"Generate new MAC address for each network adapter"**.

This "master" VM will serve as the template for all clones.

---

## Step 2 – Create Linked Clones

1. Right-click the master VM → **Clone**.
2. Choose a name (e.g., `VM_1`, `VM_2`, etc.).
3. Select **Linked Clone**.
4. Click **Finish**.
5. Repeat until you have three clones.

You will not need the master VM again unless creating more copies.

---

## Environment A – Basic Local Network

This environment is used for basic connectivity tests between VMs.

**Topology:**

![setting_up](/assets/images/a00_setting_up_a.png)

[VM_1: 192.168.0.1] --- [VM_2: 192.168.0.2] --- [VM_3: 192.168.0.3]

**Configuration:**

VM_1
```bash
sudo ip link set dev enp0s3 up      # Enable interface
sudo ip addr add 192.168.0.1/24 dev enp0s3
```

VM_2

```bash
sudo ip link set dev enp0s3 up
sudo ip addr add 192.168.0.2/24 dev enp0s3
```

VM_3

```bash
sudo ip link set dev enp0s3 up
sudo ip addr add 192.168.0.3/24 dev enp0s3
```

## Environment B – Firewall Lab Topology

This environment is designed for firewall and routing experiments.

**Topology:**

![setting_up](/assets/images/a00_setting_up_b.png)

[VM_1: 192.168.0.1] --- [VM_3: 192.168.0.3 | 172.16.0.3] --- [VM_2: 172.16.0.2]

* VM_3 acts as a **router/firewall** between two networks.
* VM_1 and VM_2 each run a simple Apache HTTP server for testing.

**Configuration:**

VM_1

```bash
sudo ip link set dev enp0s3 up
sudo ip addr add 192.168.0.1/24 dev enp0s3
sudo ip route add default via 192.168.0.3
sudo systemctl start apache2
```

VM_2

```bash
sudo ip link set dev enp0s3 up
sudo ip addr add 172.16.0.2/16 dev enp0s3
sudo ip route add 192.168.0.0/24 via 172.16.0.3
sudo systemctl start apache2
```

VM_3 (router/firewall)

```bash
sudo ip link set dev enp0s3 up
sudo ip link set dev enp0s8 up
sudo ip addr add 192.168.0.3/24 dev enp0s3
sudo ip addr add 172.16.0.3/16 dev enp0s8
sudo sysctl -w net.ipv4.ip_forward=1   # Enable routing
```

----

## Testing Connectivity

You can test the network with `ping`. For example, from **VM_1**:

```bash
ping -c 3 192.168.0.2
```

---

## Notes

- **Linked clones** share the base disk image — changes are stored separately, saving space.
- **IP addressing** is static for reproducibility in lab exercises.
- **Apache2** is only used for basic HTTP tests; it can be replaced with any service you prefer.
- **IP forwarding** is temporary in this setup. To make it persistent, you can edit `/etc/sysctl.conf`.
