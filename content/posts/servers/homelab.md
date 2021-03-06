---
title: "Homelab Overview"
date: 2020-10-16T20:14:55-07:00
hero: /images/posts/servers/homelab-overview/office.jpg
description: A brief overview of my homelab
theme: Toha
author:
  name: Chuck Valenza
  image: /images/avatar.svg
menu:
  sidebar:
    name: Homelab Overview
    identifier: homelab-overview
    parent: servers
    weight: 500
---

I run a networking lab in my house, fondly coined a 'homelab.'

I stand up a variety of services from time to time when experimenting with new
software and networking concepts. Services which the lab constantly provides
to my home are: DNS, SIEM, wireless, file shares, print and scan service, a plex
server, publicly accessible Gitlab and Nextcloud services, and PoE IP cameras.
From time-to-time I stand up various virtual machines to experiment with new
concepts.

## Hardware Summary

{{< vs 3 >}}

{{< img src="/images/posts/servers/homelab-overview/homelab.png" width="900" align="center">}}

{{< vs 3 >}}

Above is an image of my lab. From top to bottom, left to right:

| Item                          | Notes                                                           |
| ----------------------------- | --------------------------------------------------------------  |
| Surfboard SB6190 Modem        | 32x8 DOCSIS 3.0 600Mbps modem                                   |
| Unifi CloudKey Gen 2+         | Unifi Network Management Service and Unifi Protect              |
| Dell PowerEdge R210 II        | pfSense Firewall                                                |
| TP-Link TL-SG108PE            | Powers my Unifi AP and the Unifi CK2                            |
| Netgear GS310TP               | Powers my PoE cameras                                           |
| Cisco SG300-28                | Primary switch for my network                                   |
| Dell PowerEdge R620           | Runs my ESXi server and the majority of my virtualized services |
| Synology Disk Station DS1618+ | Runs a few shares to the network                                |

{{< vs 3 >}}

## Not pictured

I also run a proxmox server on an old desktop machine and a small raspberry pi
fly-away kit to stand up a vpn tunnel back to my home network.

{{< vs 3 >}}

## Future plans

I currently plan on setting up VoIP services in the house. I've also never quite
got Jenkins and Kubernetes to work in my dev environments, so automated
regression testing would be a thing I would like to stand up. I ran a Matrix
chat server for a while, but the server was still young and buggy, I'd like to
host a chat server in the future again.

