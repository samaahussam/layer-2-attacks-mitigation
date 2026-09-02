# Mitigating Common Layer 2 Attacks in a LAN

A hands-on Cisco Packet Tracer project that explores five common Layer 2 attacks and the Cisco IOS security features used to reduce their impact.

The project starts with a small multi-VLAN LAN, demonstrates the security weakness behind each attack, applies the appropriate mitigation, and verifies the resulting configuration using Cisco IOS `show` commands.

The main goal of the project is to connect CCNA switching concepts with basic security practices: understanding how a Layer 2 attack works, recognizing what makes the network vulnerable, applying a suitable mitigation, and checking that the mitigation is actually active.

> **Note:** Packet Tracer has limitations when it comes to reproducing real-world attack tools. Therefore, attacks that require tools such as `macof`, `Ettercap`, or `arpspoof` are explained and approximated where possible, while the defensive configurations are implemented and verified directly on the Cisco switches.

---

## Table of Contents

1. [Project Overview](#project-overview)
2. [Objectives](#objectives)
3. [Methodology](#methodology)
4. [Network Topology](#network-topology)
5. [VLAN Design](#vlan-design)
6. [IP Addressing](#ip-addressing)
7. [Attack 1: MAC Flooding](#attack-1-mac-flooding)
8. [Attack 2: DHCP Spoofing and Starvation](#attack-2-dhcp-spoofing-and-starvation)
9. [Attack 3: ARP Spoofing](#attack-3-arp-spoofing)
10. [Attack 4: VLAN Hopping](#attack-4-vlan-hopping)
11. [Attack 5: STP Manipulation](#attack-5-stp-manipulation)
12. [Packet Tracer Limitations](#packet-tracer-limitations)
13. [Summary](#summary)
14. [Tools Used](#tools-used)
15. [Conclusion](#conclusion)

---

## Project Overview

This project focuses on Layer 2 security in a small business LAN.

The network uses multiple VLANs to separate different groups of devices, with two Layer 2 access switches connected to a multilayer switch that performs inter-VLAN routing.

After building the basic network, five common Layer 2 attacks were studied:

- MAC Flooding
- DHCP Spoofing / Starvation
- ARP Spoofing
- VLAN Hopping
- STP Manipulation

For each attack, the project looks at:

- How the attack works
- Why the network is vulnerable
- What the possible impact is
- How the attack can be detected or observed
- Which Cisco security feature can help prevent it
- How to verify the configuration

The project is intended as a practical extension of CCNA switching topics rather than a full penetration-testing environment.

---

## Objectives

- Build a small multi-VLAN LAN using Cisco Packet Tracer.
- Practice VLANs, trunking, inter-VLAN routing, DHCP relay, and STP.
- Understand the basic operation of common Layer 2 attacks.
- Configure common Cisco Layer 2 security features.
- Use `show` commands to inspect and verify the configuration.
- Understand the limitations of simulating attacks inside Packet Tracer.
- Connect networking knowledge with basic security and SOC concepts.

---

## Methodology

The project uses the following approach for each attack:

### 1. Setup

Build the relevant part of the network in an initially less-hardened state.

### 2. Attack / Demonstration

Attempt to reproduce the attack, or demonstrate the relevant behavior when Packet Tracer does not provide the tools required for a complete attack.

### 3. Detection

Use available switch information such as MAC tables, ARP tables, DHCP Snooping information, trunk status, or STP status to identify relevant network behavior.

### 4. Mitigation

Apply the Cisco IOS security feature designed to reduce or prevent the attack.

### 5. Verification

Use `show` commands to confirm that the security feature is enabled and working as expected.

This approach is mainly used as a learning methodology and is not intended to represent a complete enterprise incident-response process.

---

# Network Topology

The network uses a simple two-tier design:

- **Core:** 1x Cisco 3560 Multilayer Switch
- **Access:** 2x Cisco 2960 Layer 2 Switches
- **Server:** 1x server used as the centralized DHCP server
- **End Devices:** 10 PCs distributed across the different VLANs
- **Attacker:** 1 dedicated PC connected to the left access switch

The multilayer switch performs inter-VLAN routing using SVIs.

The access switches connect the end devices and provide the main point where Layer 2 security controls are applied.

### Logical Topology

```text
                         ┌──────────────────────┐
                         │  Multilayer Switch   │
                         │   Cisco 3560         │
                         │   Core / L3          │
                         └──────────┬───────────┘
                                    │
                         DHCP / Server VLAN
                                    │
                              ┌─────┴─────┐
                              │   Server  │
                              └───────────┘

                   ┌────────────────┴────────────────┐
                   │                                 │
            ┌──────┴──────┐                   ┌──────┴──────┐
            │  Access SW1 │                   │  Access SW2 │
            │   Cisco2960 │                   │   Cisco2960 │
            └──────┬──────┘                   └──────┬──────┘
                   │                                 │
          ┌────────┼────────┐              ┌─────────┼─────────┐
          │        │        │              │         │         │
       Business    IT    Attacker       Business     IoT      Guest
          PCs      PCs      PC             PCs        PCs       PCs
```

VLAN Design
VLANID	        Name	                Purpose
10	        Management	              Reserved for management of network devices
20	        Business Operations	      General business users
30	        IT	                      IT and technical users
40	        Servers	                  Server network, including the DHCP server
50	        IoT	                      IoT and low-trust devices
99	        Guest	                    Guest devices with limited access
999	        Native/Unused	            Unused VLAN used as the native VLAN on trunk

Why use multiple VLANs?

VLANs provide logical separation between different groups of devices.

For example, the IT VLAN is separated from normal business users, while Guest and IoT devices are placed in separate networks.

The VLAN structure is mainly used in this project to provide a realistic environment for demonstrating Layer 2 security concepts.

VLAN	Network	Gateway / SVI	       Example         DHCP Range
10	10.10.10.0/24	                 10.10.10.1	     10.10.10.10+
20	10.10.20.0/24	                 10.10.20.1	     10.10.20.10+
30	10.10.30.0/24	                 10.10.30.1	     10.10.30.10+
40	10.10.40.0/24	                 10.10.40.1	     Static
50	10.10.50.0/24	                 10.10.50.1	     10.10.50.10+
99	10.10.99.0/24	                 10.10.99.1	     10.10.99.10+

The multilayer switch performs inter-VLAN routing using SVIs.

The DHCP server is located in VLAN 40. Since DHCP requests are broadcasts, ip helper-address is configured on the required SVIs to forward DHCP requests to the centralized server.

No dynamic routing protocol is required because the lab contains a single Layer 3 switch.

Attack 1: MAC Flooding

What is MAC Flooding?

A switch uses a CAM/MAC address table to associate MAC addresses with switch ports.

Normally, this allows the switch to forward a frame only through the port where the destination device is located.

In a MAC flooding attack, an attacker sends a large number of frames with different source MAC addresses in an attempt to exhaust the switch's MAC address table.

If the table becomes exhausted, traffic for unknown destinations may be flooded to multiple ports, potentially allowing an attacker to observe traffic on the local segment.

Risk

Possible impacts include:

.Increased unknown-unicast flooding
.Network performance issues
.Increased opportunity for traffic sniffing on the local segment
.Packet Tracer Demonstration

Packet Tracer does not provide tools such as macof for generating a large number of randomized MAC addresses.

Instead, the attacker PC was used to demonstrate the basic MAC-learning behavior by changing its MAC address and generating traffic.

This is not a full MAC flooding attack. It is only a simple demonstration of how a switch can learn different MAC addresses on the same physical port.

Mitigation: Port Security

Port Security can limit the number of MAC addresses learned on an access port.

Example:

interface FastEthernet0/X
 switchport mode access
 switchport port-security
 switchport port-security mac-address sticky
 switchport port-security maximum 1
 switchport port-security violation shutdown

Configuration explanation

.switchport mode access
Keeps the port as an access port.
.port-security
Enables Port Security.
.mac-address sticky
Allows the switch to dynamically learn a MAC address and add it as a secure MAC address.
.maximum 1
Allows only one secure MAC address on the port.
.violation shutdown
Places the port into an error-disabled state if a security violation occurs.

Detection / Verification
Useful commands include:

show mac address-table
show port-security
show port-security address

These commands can be used to inspect learned MAC addresses and confirm that Port Security is active.



Attack 2: DHCP Spoofing and Starvation

What are they?

DHCP Starvation

The attacker attempts to consume the available DHCP addresses by sending many DHCP requests using different MAC addresses.
The goal is to prevent legitimate clients from obtaining an IP address.

DHCP Spoofing

The attacker introduces a rogue DHCP server into the network.

If clients accept the rogue server's response, the attacker may provide a malicious:

.Default gateway
.DNS server
.IP configuration

This can potentially help create a Man-in-the-Middle scenario or redirect traffic.

Risk

Possible impacts include:

.Denial of service for new clients
.Incorrect network configuration
.Traffic redirection
.Potential Man-in-the-Middle attacks
.Packet Tracer Demonstration

Packet Tracer does not provide a tool for automatically generating large numbers of DHCP requests with randomized MAC addresses.

A rogue DHCP server can be configured to demonstrate the concept of a second DHCP source.

However, the exact behavior of a real-world DHCP starvation or rogue DHCP attack cannot be reproduced completely using the available Packet Tracer tools.

Mitigation: DHCP Snooping

DHCP Snooping allows the switch to distinguish between trusted and untrusted DHCP sources.

Example:

ip dhcp snooping
ip dhcp snooping vlan 10,20,30,40,50,99

The interface connected toward the legitimate DHCP server should be trusted:

interface GigabitEthernet0/X
 ip dhcp snooping trust

User-facing access ports remain untrusted by default.

DHCP server responses arriving through an untrusted port can therefore be blocked.

DHCP Snooping also creates a binding table containing information such as:

MAC Address
IP Address
VLAN
Port

This information can also be used by Dynamic ARP Inspection.

Verification
show ip dhcp snooping
show ip dhcp snooping binding

These commands can be used to verify that DHCP Snooping is enabled and that bindings are being learned.


Attack 3: ARP Spoofing

What is ARP Spoofing?

ARP is used to map an IP address to a MAC address on a local network.

ARP does not provide authentication, so a device can send a forged ARP message claiming that a particular IP address belongs to its MAC address.

How the Attack Works?

For example, an attacker can attempt to convince a victim that:

Gateway IP → Attacker MAC

The attacker may also attempt to convince the gateway that:

Victim IP → Attacker MAC

If successful, the attacker can place themselves between the victim and the gateway.

This can create a Man-in-the-Middle position.

Risk

Possible impacts include:

.Traffic interception
.Traffic modification
.Session hijacking opportunities
.Redirection of unencrypted traffic
.Packet Tracer Limitation

Packet Tracer's PC command prompt does not provide all of the functionality required to generate arbitrary forged ARP replies like tools used in real environments.

Therefore, a complete ARP poisoning attack was not reproduced in the lab.

The normal ARP behavior and the IP-to-MAC relationship can still be inspected, while the defensive configuration can be implemented and verified.

Mitigation: Dynamic ARP Inspection

Dynamic ARP Inspection (DAI) validates ARP packets received on untrusted ports.

It can use the DHCP Snooping binding table to compare the IP and MAC information of ARP messages against known legitimate bindings.

Example:

ip arp inspection vlan 10,20,30,40,50,99

The trusted uplink can be configured as:

interface GigabitEthernet0/X
 ip arp inspection trust

User-facing access ports remain untrusted.

Verification
show ip arp inspection
show ip arp
show ip dhcp snooping binding

These commands can be used to inspect the ARP inspection configuration and the bindings used for validation.



Attack 4: VLAN Hopping

What is VLAN Hopping?

VLAN hopping is an attempt to access traffic belonging to a VLAN that the attacker should not normally be able to access.

Two common techniques are:

DTP-based VLAN Hopping

If a switch port is allowed to negotiate trunking using DTP, an attacker may attempt to make the port become a trunk.

A trunk could then provide access to multiple VLANs.

Double Tagging

An attacker can construct a frame containing two VLAN tags.

The outer tag can be removed by the first switch, leaving the inner VLAN tag to be processed by another switch.

This technique depends on the native VLAN and the network topology.

Mitigation

Several basic hardening steps can be used together.

1. Explicitly Configure Access Ports

User-facing ports should not negotiate trunking:

interface FastEthernet0/X
 switchport mode access

2. Explicitly Configure Trunk Ports

Uplinks between switches should be configured as trunks:

interface GigabitEthernet0/X
 switchport mode trunk

3. Use an Unused Native VLAN

Instead of using VLAN 1 as the native VLAN:

interface GigabitEthernet0/X
 switchport trunk native vlan 999

VLAN 999 is not assigned to normal end devices.

4. Restrict Allowed VLANs

Only the VLANs required on the trunk should be allowed:

interface GigabitEthernet0/X
 switchport trunk allowed vlan 10,20,30,40,50,99,999
Verification
show interfaces trunk

The output can be checked to confirm:

.The port is operating as a trunk.
.Native VLAN is 999.
.Only the intended VLANs are allowed.




Attack 5: STP Manipulation

What is STP?

Spanning Tree Protocol (STP) prevents Layer 2 switching loops by creating a loop-free topology.

STP elects a Root Bridge based on the lowest Bridge ID.

The Bridge ID is determined using the bridge priority and MAC address.

How the Attack Works

An attacker who manages to connect a rogue switch to the network may attempt to send BPDUs with a better Bridge ID.

If accepted, the rogue switch could become the Root Bridge.

This can cause STP to recalculate the topology and change traffic paths.

Repeated topology changes can also contribute to network instability or denial of service.

Mitigation 1: BPDU Guard

BPDU Guard can be enabled on user-facing ports.

Example:

interface range FastEthernet0/1-4
 spanning-tree portfast
 spanning-tree bpduguard enable

If a BPDU is received on a protected edge port, the switch can place the port into an error-disabled state.

This helps prevent an unauthorized switch from participating in STP through an end-user port.

Mitigation 2: Root Guard

Root Guard can be used on ports where the administrator does not want a downstream switch to become the Root Bridge.

Example:

interface GigabitEthernet0/X
 spanning-tree guard root

If a superior BPDU is received, the port can enter a root-inconsistent state instead of accepting the new Root Bridge.

Verification

Useful commands include:

show spanning-tree
show spanning-tree summary
show running-config

show spanning-tree can be used to identify the current Root Bridge and inspect the STP state of the ports.

Packet Tracer Limitations

Packet Tracer is primarily a network simulation and learning tool. It does not provide all of the offensive tools and packet-generation capabilities available in real security testing environments.

For this reason, the five attacks in this project are not all reproduced in exactly the same way.

Attack	            Simulation in Packet Tracer
MAC Flooding	      Basic MAC-learning behavior demonstrated; no macof-style flooding
DHCP Starvation	    Cannot perform automated large-scale fake-MAC requests
DHCP Spoofing	      Rogue DHCP concept can be demonstrated
ARP Spoofing	      Complete forged-ARP attack not reproduced
VLAN Hopping	      Configuration and trunk behavior can be studied and tested
STP Manipulation    STP behavior and security configuration can be studied and tested

These limitations are part of the reason the project focuses on both attack understanding and defensive configuration.

The project does not claim that Packet Tracer provides the same attack capabilities as a real penetration-testing environment.




Summary
Attack	                          Main Idea	                                   Cisco Mitigation
MAC Flooding	                    Attempt to exhaust the switch MAC table	     Port Security
DHCP Spoofing / Starvation	      Rogue DHCP server or DHCP pool exhaustion	   DHCP Snooping
ARP Spoofing	                    Forged IP-to-MAC mappings	                   Dynamic ARP Inspection
VLAN Hopping	                    Attempt to bypass VLAN separation	           Access/Trunk hardening + Native VLAN + Allowed VLANs
STP Manipulation	                Attempt to influence the STP Root Bridge	   BPDU Guard + Root Guard



Tools Used
.Cisco Packet Tracer — Network design, configuration, and simulation.
.Cisco IOS CLI — Switch configuration and verification.
.draw.io — Network topology documentation.


This project was built as a practical introduction to Layer 2 network security using Cisco Packet Tracer.

It combines basic CCNA switching concepts with several common Layer 2 security threats and their corresponding Cisco security features.

The main configurations practiced in the project include:

.Port Security
.DHCP Snooping
.Dynamic ARP Inspection
.Trunk hardening
.Native VLAN configuration
.Allowed VLAN restrictions
.BPDU Guard
.Root Guard

Because Packet Tracer has limitations when simulating real attack tools, some attacks were demonstrated only at the level supported by the simulator. The project therefore focuses on understanding why the attacks work, what the vulnerable configuration looks like, which mitigation should be used, and how to verify that the mitigation is configured correctly.

This project is intended as a starting point for further networking and security labs. Future work could extend the lab with topics such as ACLs, 802.1X, VTP security, CDP/LLDP security, and more realistic attack simulations using dedicated security lab environments.
