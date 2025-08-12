---
title: "Lab 2. NFTables"
excerpt: "In this lab, you’ll learn to perform and analyze several network protocol attacks across different layers of the OSI model, including ARP spoofing, ICMP flooding, Smurf attacks, TCP SYN floods, and network enumeration with Nmap."
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
  - network-security
  - pentesting
  - ethical-hacking
  - cybersecurity
  - linux-tools
  - nftables

---

## Environment

This lab builds upon the environment setup from [Lab 0. Setting up the environment](/labs/a00-setting-up-environment/). All attacks will be launched from **VM_2**, unless stated otherwise.

> **Warning:** While Wireshark is extremely useful for analyzing network traffic, it’s not recommended to run it during *flood* attacks due to the high volume of packets it will attempt to capture. This can cause performance issues or even crash the interface.

> **Tip:** If you're running Wireshark in VirtualBox and not seeing any traffic, make sure promiscuous mode is enabled:
>
> `Devices → Network → Network Settings → Adapter → Advanced → Promiscuous Mode: Allow All`

## 