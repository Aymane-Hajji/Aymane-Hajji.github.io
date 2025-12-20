---
title: "Enterprise Site-to-Site IPSec VPN Implementation (VyOS)"
date: 2025-12-13
author: Aymane Aboukhaira
status: Deployed / Operational
---

# Enterprise Site-to-Site IPSec VPN Implementation 

Date: 13 December 2025  
Author: Aymane Aboukhaira  
Status: Deployed / Operational  


---

## 1. Executive Summary

This project demonstrates the successful implementation of a secure, routed Site-to-Site IPSec VPN connecting a simulated Branch Office network to a Headquarters (HQ) network over an untrusted public backbone.

The primary objective was to establish encrypted connectivity between private LAN subnets, allowing seamless and secure access to internal resources across geographically separated locations, while ensuring that internal traffic remains hidden from the intermediate transport network.

The solution leverages VyOS open-source routers, OSPF dynamic routing for the underlay network, and a policy-based IPSec VPN for the overlay tunnel.

---
## 2. Technical Scope & Architecture

### 2.1 Design Overview

The architecture is divided into two logical layers:

Underlay Network (Routing / Transport Layer)
- Simulates the public internet using a chain of four routers
- Uses OSPF Area 0 to ensure IP reachability between WAN interfaces
- Responsible only for packet transport, not security

Overlay Network (VPN / Security Layer)
- An IPSec tunnel built on top of the underlay
- Encrypts traffic between private LAN subnets
- Makes intermediate ISP routers invisible to end-to-end communication

---

### 2.2 Technologies Used

- Virtualization Platform: VMware Workstation / ESXi  
- Network OS: VyOS 1.4 (Rolling Release)  
- Routing Protocol: OSPFv2  
- VPN Protocol: IPSec (IKEv1 + ESP, Tunnel Mode)  
- Client OS:
  - Branch: Windows 7
  - HQ: Linux / Windows

---

## 3. Network Topology & Addressing Schema

### 3.1 Topology Diagram

![Network Topology Diagram](/assets/images/topology.webp)

---

### 3.2 IP Addressing Table

Device | Hostname | Interface | Description | IP Address
------ | -------- | --------- | ----------- | ----------
Router 1 | branch-rtr | eth1 | Branch LAN Gateway | 10.1.1.1/24
 |  | eth2 | WAN to ISP A | 10.1.2.1/24
Router 2 | isp-a-rtr | eth1 | Connects to Branch | 10.1.2.2/24
 |  | eth2 | ISP Backbone | 10.2.3.2/24
Router 3 | isp-b-rtr | eth1 | ISP Backbone | 10.2.3.3/24
 |  | eth2 | Connects to HQ | 10.3.4.3/24
Router 4 | hq-rtr | eth2 | WAN to ISP B | 10.3.4.4/24
 |  | eth3 | HQ LAN Gateway | 10.4.4.1/24
Client 1 | branch-pc | eth0 | Branch Workstation | 10.1.1.10/24
Client 2 | hq-server | eth0 | HQ Resource | 10.4.4.44/24

---

## 4. Implementation Details

### 4.1 Phase 1: Underlay Connectivity (OSPF)

Before deploying the VPN, end-to-end IP connectivity between the Branch WAN (10.1.2.1) and HQ WAN (10.3.4.4) was required.

OSPF Area 0 was configured on all routers to dynamically advertise all WAN subnets. LAN subnets were temporarily advertised during initial testing.

Example configuration (Router 1):

```bash
set interfaces ethernet eth1 address '10.1.1.1/24'
set interfaces ethernet eth2 address '10.1.2.1/24'

set protocols ospf area 0 network '10.1.1.0/24'
set protocols ospf area 0 network '10.1.2.0/24'
```

Verification: Successful OSPF adjacency was confirmed via show ip route ospf, showing routes marked with O*>, indicating dynamic discovery of remote WAN subnets.

4.2. Phase 2: Overlay Configuration (IPSec VPN)
A Policy-Based VPN was configured between Router 1 and Router 4. The policy dictates that traffic originating from 10.1.1.0/24 destined for 10.4.4.0/24 must be encrypted.

4.2.1. Security Parameters (IKE/ESP Proposals)
To ensure strong security, the following encryption standards were defined:

IKE Phase 1 (Main Mode):

Encryption: AES-256

Hash: SHA-256

Diffie-Hellman Group: 14 (2048-bit modulus)

Lifetime: 28800 seconds
--------
ESP Phase 2:

Encryption: AES-256

Hash: SHA-256

Mode: Tunnel

PFS (Perfect Forward Secrecy): Enabled

4.2.2. Peer Configuration
The VPN peer relationship was defined using Pre-Shared Key (PSK) authentication.

Configuration Snippet (Router 1 - Branch):

Bash

# Define Proposals
```bash

set vpn ipsec ike-group IKE-PROP-1 proposal 1 encryption 'aes256'
set vpn ipsec ike-group IKE-PROP-1 proposal 1 hash 'sha256'
set vpn ipsec ike-group IKE-PROP-1 proposal 1 dh-group '14'
set vpn ipsec esp-group ESP-PROP-1 proposal 1 encryption 'aes256'
set vpn ipsec esp-group ESP-PROP-1 proposal 1 hash 'sha256'
set vpn ipsec esp-group ESP-PROP-1 pfs 'enable'
```

# Define Peer Connection
```bash
set vpn ipsec site-to-site peer 10.3.4.4 authentication mode 'pre-shared-secret'
set vpn ipsec site-to-site peer 10.3.4.4 authentication pre-shared-secret 'secret123'
set vpn ipsec site-to-site peer 10.3.4.4 ike-group 'IKE-PROP-1'
set vpn ipsec site-to-site peer 10.3.4.4 local-address '10.1.2.1'
```
# Define Traffic Selectors (The Policy)
```bash

set vpn ipsec site-to-site peer 10.3.4.4 tunnel 1 local prefix '10.1.1.0/24'
set vpn ipsec site-to-site peer 10.3.4.4 tunnel 1 remote prefix '10.4.4.0/24'
(Router 4 configuration is the mirror image of the above, pointing back to peer 10.1.2.1 with subnets reversed).
```

5. Verification and Validation
The following tests confirmed the successful deployment and operation of the VPN infrastructure.

5.1. Control Plane Verification (Tunnel Status)
Command: show vpn ipsec sa on VyOS. Result: Status confirmed as State: up. The presence of "Bytes Out/In" counters increasing confirms encrypted traffic flow.

5.2. Data Plane Verification (End-to-End Connectivity)
Ping tests were conducted from the Branch Windows Client (10.1.1.10) to the HQ Server (10.4.4.44). Result: Successful ICMP replies with <2ms latency and 0% packet loss.

5.3. Path Analysis (Traceroute)
A traceroute was performed from the Branch LAN to the HQ LAN to verify traffic encapsulation.

Expected Behavior (Normal Routing): 3 hops (R1 -> R2 -> R3 -> R4).

Observed Behavior (VPN Routing):

```bash

C:\> tracert 10.4.4.44
Tracing route to 10.4.4.44 over a maximum of 30 hops:
  1    <1 ms    10.1.1.1 (Branch Gateway)
  2     * Request timed out. (HQ Gateway / VPN Terminator)
  3    <1 ms    10.4.4.44 (HQ Destination)
```
Conclusion: The intermediate routers (hops 2 and 3 in a normal trace) are invisible, confirming that traffic is successfully traversing the encrypted tunnel.


6. Challenges Encountered and Resolutions
During the implementation phase, several critical issues were encountered and resolved:

OSPF Persistence Failure:

Issue: OSPF adjacencies were lost after initial router reboots, breaking underlay connectivity.

Resolution: Determined that the running configuration had not been saved to the startup configuration. Executed save command on all routers after confirming OSPF stability.

IKE Protocol Mismatch:

Issue: VPN Phase 1 failed to establish. Logs indicated "no proposal chosen."

Resolution: Audited configuration and found Router 1 defaulted to IKEv2 while Router 4 was explicitly set to IKEv1. Standardized both peers to use key-exchange 'ikev1'.

Authentication Failure (PSK Typo):

Issue: VPN status remained "down" despite correct crypto proposals. Logs showed authentication errors.

Resolution: A case-sensitivity error was found in the pre-shared secret (Secret123 vs secret123). The secrets were updated on both peers to match exactly.

7. Future Scope
To enhance this infrastructure for a production environment, the following steps are recommended:

DHCP Services: Configure DHCP servers on Router 1 and Router 4 to automatically assign IPs to LAN clients.

Zone-Based Firewalling: Implement strict firewall policies on the edge routers to permit only encrypted VPN traffic (UDP 500/4500, ESP) on WAN interfaces, denying all other inbound internet traffic.

High Availability (HA): Implement redundant routers at Branch and HQ locations using VRRP for gateway redundancy.
