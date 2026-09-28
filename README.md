# VLSM Growing Company Network

## Overview

This project is the second lab in my networking project series, which is focused on designing and implementing an efficient IP addressing scheme for a growing company using **Variable Length Subnet Masking (VLSM)**.

The company was allocated the `172.16.0.0/16` network and required separate networks for six departments with different host requirements.

Instead of assigning the same subnet size to every department, I used VLSM to allocate address space based on each department's actual requirements, then implemented the design in **Cisco Packet Tracer** using VLANs, trunking, router-on-a-stick, DHCP, and static addressing.

---

## Network Requirements

| Department | Required Hosts | VLAN | Allocated Network |
|---|---:|---:|---|
| Operations | 200 | 10 | `172.16.0.0/24` |
| Sales | 100 | 20 | `172.16.1.0/25` |
| Engineering | 60 | 30 | `172.16.1.128/26` |
| Finance | 30 | 40 | `172.16.1.192/27` |
| IT | 20 | 50 | `172.16.1.224/27` |
| Servers | 10 | 60 | `172.16.2.0/28` |

---

## Network Topology

The network was built using:

- 1 router
- 3 Cisco 2960 switches
- 6 VLANs
- Multiple end devices representing each department

The central switch connects the two access switches to the router.

```text
                       Router
                          |
                       Trunk
                          |
                         S1
                    _____|_____
                   |           |
                 Trunk       Trunk
                   |           |
                  S2           S3
             Operations       Sales
             Servers          Engineering
                              Finance
                              IT
```

![Complete Network Topology](https://github.com/maryjane-ccc/vlsm-growing-company-network/blob/291ca0f505ff62dd71630d6700f05e0605a44154/Complete%20Network%20Topology.png)

---

## VLSM Addressing Plan

I calculated the required subnet size for each department using:

```text
2^h - 2 >= Required Hosts
```

where `h` represents the number of host bits.

The final addressing plan was:

| Department | Network | Subnet Mask | Default Gateway | Broadcast |
|---|---|---|---|---|
| Operations | `172.16.0.0/24` | `255.255.255.0` | `172.16.0.1` | `172.16.0.255` |
| Sales | `172.16.1.0/25` | `255.255.255.128` | `172.16.1.1` | `172.16.1.127` |
| Engineering | `172.16.1.128/26` | `255.255.255.192` | `172.16.1.129` | `172.16.1.191` |
| Finance | `172.16.1.192/27` | `255.255.255.224` | `172.16.1.193` | `172.16.1.223` |
| IT | `172.16.1.224/27` | `255.255.255.224` | `172.16.1.225` | `172.16.1.255` |
| Servers | `172.16.2.0/28` | `255.255.255.240` | `172.16.2.1` | `172.16.2.15` |

The first usable IP address in each subnet was assigned to the router as the default gateway.

---

## What I Implemented

- VLSM-based IPv4 subnet allocation
- Six departmental VLANs
- Access-port configuration
- 802.1Q trunking between switches
- Restricted VLANs on trunk links
- Router-on-a-stick
- Inter-VLAN routing
- DHCP for user VLANs
- Static addressing for the Server VLAN
- Spanning Tree Protocol verification
- End-to-end connectivity testing
- Network troubleshooting

---

## VLAN Design

| VLAN | Department |
|---:|---|
| 10 | Operations |
| 20 | Sales |
| 30 | Engineering |
| 40 | Finance |
| 50 | IT |
| 60 | Servers |

### S2 Access Ports

```text
Fa0/1 - Fa0/15   -> VLAN 10 - Operations
Fa0/16 - Fa0/21  -> VLAN 60 - Servers
```

### S3 Access Ports

```text
Fa0/1 - Fa0/9    -> VLAN 20 - Sales
Fa0/10 - Fa0/15  -> VLAN 30 - Engineering
Fa0/16 - Fa0/19  -> VLAN 40 - Finance
Fa0/20 - Fa0/23  -> VLAN 50 - IT
```

---

## Trunk Configuration

The trunk links were configured to carry only the VLANs required on each path.

```text
S1 Fa0/23 -> S2 -> VLANs 10,60
S1 Fa0/24 -> S3 -> VLANs 20,30,40,50
S1 Gi0/1  -> Router -> VLANs 10,20,30,40,50,60
```

Example:

```cisco
interface fa0/23
 switchport mode trunk
 switchport trunk allowed vlan 10,60
```

Trunk operation was verified using:

```cisco
show interfaces trunk
```

---

## Router-on-a-Stick

Inter-VLAN routing was implemented using router subinterfaces and IEEE 802.1Q encapsulation.

```cisco
interface g0/0/0
 no shutdown

interface g0/0/0.10
 encapsulation dot1Q 10
 ip address 172.16.0.1 255.255.255.0

interface g0/0/0.20
 encapsulation dot1Q 20
 ip address 172.16.1.1 255.255.255.128

interface g0/0/0.30
 encapsulation dot1Q 30
 ip address 172.16.1.129 255.255.255.192

interface g0/0/0.40
 encapsulation dot1Q 40
 ip address 172.16.1.193 255.255.255.224

interface g0/0/0.50
 encapsulation dot1Q 50
 ip address 172.16.1.225 255.255.255.224

interface g0/0/0.60
 encapsulation dot1Q 60
 ip address 172.16.2.1 255.255.255.240
```

The subinterfaces were verified using:

```cisco
show ip interface brief
```

---

## DHCP Configuration

The router provides DHCP addressing for the five user VLANs.

Example:

```cisco
ip dhcp excluded-address 172.16.0.1 172.16.0.10

ip dhcp pool Operations
 network 172.16.0.0 255.255.255.0
 default-router 172.16.0.1
```

Separate DHCP pools were configured for:

- Operations
- Sales
- Engineering
- Finance
- IT

DHCP operation was verified using:

```cisco
show ip dhcp pool
show ip dhcp binding
```

The Server VLAN uses **static addressing** rather than DHCP.

---

## Troubleshooting

One of the most useful parts of this project was troubleshooting connectivity and DHCP failures.

Initially, clients were unable to obtain IP addresses.

### Issue 1: Missing Router Trunk

The link between the central switch and router had not been configured as a trunk.

After configuring the S1 router-facing interface:

```cisco
interface gi0/1
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30,40,50,60
 no shutdown
```

Engineering, Finance, and IT began receiving DHCP addresses.

However, Operations and Sales still failed.

### Issue 2: Operations and Sales Connectivity

To determine whether DHCP itself was responsible, I temporarily configured an Operations PC with a valid static address:

```text
IP Address:      172.16.0.20
Subnet Mask:     255.255.255.0
Default Gateway: 172.16.0.1
```

I then attempted to ping the gateway:

```text
ping 172.16.0.1
```

The ping failed.

This showed that the issue was not simply DHCP because the device could not reach its gateway even with a valid static configuration.

I checked:

- VLAN membership
- Access ports
- Trunk configuration
- STP forwarding state
- Router subinterfaces

The Operations router subinterface was then inspected:

```cisco
interface g0/0/0.10
do show interfaces g0/0/0.10
```

The configured address was:

```text
176.16.0.1/24
```

instead of:

```text
172.16.0.1/24
```

The Sales VLAN had the same type of gateway addressing error.

After correcting both router subinterfaces, Operations and Sales successfully communicated with their gateways and received DHCP addresses.

---

## Troubleshooting Lesson

The biggest lesson from this project was that **the service showing the symptom is not necessarily the service causing the problem**.

The visible symptom was a DHCP failure, but the underlying problem was basic Layer 3 connectivity caused by incorrect gateway addressing.

Using a temporary static IP and testing connectivity to the default gateway helped separate a DHCP problem from a network connectivity problem.

My troubleshooting process became:

```text
Physical Connectivity
        |
        v
Access Port
        |
        v
VLAN Membership
        |
        v
Trunk Configuration
        |
        v
STP Forwarding
        |
        v
Router Subinterface
        |
        v
IP Addressing
        |
        v
DHCP
```

---

## Design Considerations and Future Scalability

Although the VLSM design satisfies the current host requirements, some subnets have limited capacity for future expansion.

For example, Finance requires 30 hosts and was allocated:

```text
172.16.1.192/27
```

A `/27` provides exactly **30 usable host addresses**.

This satisfies the current requirement but leaves no additional host capacity.

In a production network for a growing organization, I would consider allocating a larger subnet such as a `/26` if significant Finance department growth were expected.

I retained the `/27` allocation in this lab because it demonstrates efficient VLSM allocation based on the stated requirements.

Other future improvements could include:

- Reserving address space for additional departments
- Reviewing subnet sizes as departments grow
- Implementing ACLs between departmental VLANs
- Adding redundant links and devices
- Implementing network monitoring and centralized management

---

## Verification Commands

Some of the commands used to verify and troubleshoot the network included:

```cisco
show vlan brief
show interfaces trunk
show spanning-tree vlan 10
show ip interface brief
show interfaces g0/0/0.10
show ip dhcp pool
show ip dhcp binding
show running-config
```

Configurations were saved using:

```cisco
copy running-config startup-config
```

---

## Skills Practised

- VLSM
- IPv4 subnetting
- IP address planning
- VLAN configuration
- Access-port configuration
- IEEE 802.1Q trunking
- Router-on-a-stick
- Inter-VLAN routing
- DHCP
- Static IP addressing
- Spanning Tree Protocol
- Cisco IOS CLI
- Network troubleshooting
- Network design and scalability planning

---

## Tools Used

- Cisco Packet Tracer
- Cisco IOS CLI

---

## Project Series

**10 Weeks of Networking: From NetAcad Student to Network Builder**

- Week 1: Small Business Network
- **Week 2: VLSM Growing Company Network**
- Week 3: Network Troubleshooting Challenge
