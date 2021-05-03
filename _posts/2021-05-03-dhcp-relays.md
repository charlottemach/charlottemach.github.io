---
layout: post
title: "What the heck are DHCP relays?"
date: 2021-05-03
tags: DHCP networking
---

If you're just getting starting with Zero-Touch-Provisioning there are a few networking challenges ahead for you. Luckily there are some great tools to support out there, which make getting started very simple. 

### DHCP

The Dynamic Host Configuration Protocol (DHCP) is used to dynamically assign IP addresses to devices connected via a [layer 2 network](https://en.wikipedia.org/wiki/Data_link_layer). 
That means a DHCP server needs to exist in the same network already, or you're stuck with having to assign static IPs.

DHCP has four phases of operation:

![DHCP workflow](/assets/images/dhcp1.png "DHCP operations")
1. Discovery - Broadcasting an IP address lease request
2. Offer - Offering an IP for lease to the client
3. Request - Requesting to receive the offered IP
4. Acknowledge - Accepting the request and relaying other information such as lease time

If you're just getting started setting up a data center and want to do zero touch provisioning, you're generally gonna want some way of accessing the [Baseboard Management Controllers (BMC)](https://en.wikipedia.org/wiki/Intelligent_Platform_Management_Interface#Baseboard_management_controller) to manage your servers. To access those, you need their IPs. But if there's no DHCP server in the same network, how can you even get started? That's where DHCP relays come in.

### DHCP Relays

DHCP Relay Agents help when having your DHCP server in a different subnet than your DHCP clients. Generally DHCP messages are broadcasted, but broadcast packets aren't routed. DHCP relays take the DHCP messages and turn them from a broadcast one into a unicast message and send them directly to one or more DHCP servers.


![DHCP relay workflow](/assets/images/dhcp2.png "DHCP operations")
Compared to the standard DHCP operations a relay acts as an intermediary basically forwarding DHCP communication between different subnets. This way you can have your DHCP server in a L2 network, but still utilize dynamic IP addressing.
1. Discovery - The relay forwards the broadcasted request to one or multiple DHCP servers that are configured for the relay
2. Offer - The DHCP server responds with an offer, which the DHCP relay either broadcasts to the other network or sends directly to the client
3. Request - The broadcasted request for the offered IP is relayed the same as the discovery message
4. Acknowledge - Same as the offer, the unicasted acknowledge message gets turned into either a unicasted or broadcasted reply

This way you can run DHCP relay agents on your routers, and assign IPs from one or more DHCP servers across multiple networks.

### Install

Installing (e.g. on Ubuntu 20.04) is very simple, just run
```
# apt install isc-dhcp-relay
```
Edit the config-file at `/etc/default/isc-dhcp-relay` so that the relay agent uses the correct DHCP server and interface:
```
SERVERS=<IP to your DHCP server>

INTERFACES=<Interface connecting to your DHCP server>
```
(Re)start the service
```
# systemctl restart isc-dhcp-relay
```
Finally check if your service is up and running correctly and then you can either check your DHCP server or tcpdump (e.g. `tcpdump -i <INTERFACE> -pvn port 67 and port 68` or `tcpdump -i <INTERFACE> -nv '(udp port 546 or port 547)'` for IPv6).
