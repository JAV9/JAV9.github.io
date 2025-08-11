---
title: "Lab 0. Setting up the Environment"
excerpt: "In this post you can learn how to set up the environment for our custom labs."
layout: single
collection: labs
classes: wide
author_profile: false
toc: false
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

## How-to
This post is about setting up the environment needed for our custom labs. You're free to modify it in future projects however you like — this will simply serve as a **base configuration** we can build upon. In upcoming posts, we’ll discuss any changes to the environment as needed.

You can choose whichever tools you feel most comfortable with. However, since this is a guided project, I'll be showing the ones I’ve chosen:

* **Virtualization platform:** Oracle VirtualBox
* **OVA/.ISO of our chosen distro:** *ubuntu-jammy*

![setting_up](/assets/images/setting_up.png)

As shown in the image above, we'll be working with three machines. There’s no need to import the same OVA multiple times — we can import it once and then create **linked clones**. 

First, configure the original virtual machine, which we’ll refer to as the **"master"** VM from now on.

1. Open the VM settings and go to the **Network** tab.
2. Enable **two network adapters**.
3. Set both to use the **'local'** network name.
4. Make sure **'Generate new MAC address for each network adapter'** is checked.

Next, we’ll create clones of the master VM:

- Right-click the imported VM → **Clone**
- Choose a name (e.g., `VM_1`, `VM_2`, etc.)
- Make sure to check **‘Linked Clone’**
- Click **Finish**

After cloning, **boot up the three cloned virtual machines**. You won’t need the master VM again unless you want to create more copies.

Now we’ll configure each VM’s network interface and IP address manually. Open a terminal in each VM and run the following:

VM_1

```bash
sudo ip link set dev enp0s3 up
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

To verify everything is working, use `ping`. For example, from **VM_1**:

```
ping -c 3 192.168.0.2
```

This will send 3 ICMP packets to **VM_2**. If the packets are received, your basic network setup is working correctly.

Feel free to test connectivity between any of the VMs using `ping`.
