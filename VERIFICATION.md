# 🧪 Verification & Testing Guide

This document outlines the step-by-step verification process to ensure the **Nokia SR Linux EVPN-VXLAN Datacenter Fabric** is operating correctly. The testing methodology follows a top-down approach: verifying the Core (Spines), the Edge (Leafs), and finally the Application Layer (Servers).

---

## 🟢 Phase 1: Core Layer Verification (Spines)
We begin by verifying the underlay and overlay control planes on the Route Reflectors (Spines).

### 1. OSPF Adjacencies & Underlay Reachability
Verifying that both spines have established full OSPF adjacencies with the connected leaf nodes in Area 0.
```bash
# On spine1
show network-instance default protocols ospf neighbor

# On spine2
show network-instance default protocols ospf neighbor
```
![OSPF Spine1](images/ospf-spine1.png)
![OSPF Spine2](images/ospf-spine2.png)

Both spines show a `FULL` OSPF state with `leaf1` (10.10.10.1) and `leaf2` (10.10.10.2). This confirms the underlay routing topology is converged and loopbacks are mutually reachable.

### 2. iBGP Sessions (Route Reflectors)
Confirming the MP-BGP EVPN sessions between the Route Reflectors and the VTEPs.
```bash
# On spine1
show network-instance default protocols bgp summary

# On spine2
show network-instance default protocols bgp summary
```
![BGP Summary Spine1](images/bgp-summary-spine1.png)
![BGP Summary Spine2](images/bgp-summary-spine2.png)

The BGP sessions for the EVPN address family are successfully `established`. Both spines (AS 65001) are peering with the leaf nodes, ensuring control plane redundancy.

---

## 🔵 Phase 2: Edge Layer Verification (Leafs)
Next, we validate the encapsulation, tenancy, and multihoming configurations on the VTEPs (Leafs).

### 3. EVPN Route Exchange & MAC-VRF
Verifying that EVPN Type 2 (MAC/IP) routes are correctly advertised and installed in the local MAC tables.
```bash
# On leaf1
show network-instance default bgp-rib evpn routes
show network-instance vlan10 bridge-table mac-table

# On leaf2
show network-instance default bgp-rib evpn routes
show network-instance vlan10 bridge-table mac-table
```
![EVPN MAC Leaf1](images/evpn-mac-leaf1.png)
![EVPN MAC Leaf2](images/evpn-mac-leaf2.png)

The outputs confirm that both leafs dynamically learned the remote endpoints via EVPN `vxlan1` interfaces and local endpoints via `lag1`, effectively synchronizing the L2 broadcast domains across the IP core.

### 4. Distributed Anycast Gateway (ARP Synchronization)
Validating the Asymmetric IRB implementation and Anycast Gateway IP-to-MAC resolution.
```bash
# On leaf1
show arpnd arp-entries

# On leaf2
show arpnd arp-entries
```
![ARP Entries Leaf1](images/arp-entries-leaf1.png)
![ARP Entries Leaf2](images/arp-entries-leaf2.png)

Both leafs successfully resolve the ARP entries for hosts in `vlan10` (192.168.10.x) and `vlan20` (192.168.20.x) via the `irb1` subinterfaces, proving that the Layer 3 gateways are active and synchronized.

### 5. All-Active Multihoming (ESI & LACP)
Checking the status of the Ethernet Segment Identifiers (ESI) and the aggregated links toward the dual-homed servers.
```bash
# On leaf1
show system network-instance protocols evpn ethernet-segments
show interface lag1 detail

# On leaf2
show system network-instance protocols evpn ethernet-segments
show interface lag1 detail
```
![ESI LACP Leaf1](images/esi-lacp-leaf1.png)
![ESI LACP Leaf2](images/esi-lacp-leaf2.png)

The ESI `00:00:00:00:00:11` is operational in `all-active` mode on both leafs. The LACP port-channels (`lag1`) are in a forwarding state (`distributing`), ensuring loop-free, active-active load balancing without relying on STP.

---

## 🔴 Phase 3: Application Layer & Failover (Servers)
The final validation focuses on end-to-end dataplane reachability and High Availability.

### 6. Intra-VLAN & Inter-VLAN Reachability
Testing L2 bridging (same subnet) and L3 routing (different subnets via Asymmetric IRB).
```bash
# Intra-VLAN: srv1 (VLAN 10) to srv3 (VLAN 10)
docker exec -it clab-pro-evpn-srv1 ping -c 4 192.168.10.3

# Inter-VLAN: srv1 (VLAN 10) to srv4 (VLAN 20)
docker exec -it clab-pro-evpn-srv1 ping -c 4 192.168.20.4
```
![Intra VLAN Ping](images/intra-vlan-ping.png)
![Inter VLAN Ping](images/inter-vlan-ping.png)

End hosts communicate seamlessly across the fabric. The Intra-VLAN traffic is bridged across the VNI 10 VXLAN tunnel, while Inter-VLAN traffic is successfully routed at the ingress leaf using the `ip-vrf-1` L3 VNI.

### 7. Sub-second Failover Test (0% Packet Loss)
Simulating a physical link failure on a dual-homed server to validate the ESI failover mechanism.
```bash
# Terminal 1: Initiate continuous ping from srv1 to srv4
docker exec -it clab-pro-evpn-srv1 ping 192.168.20.4

# Terminal 2: Administratively down one uplink on srv1
docker exec -it clab-pro-evpn-srv1 ip link set eth1 down
```
![Failover Test Srv1](images/failover-test-srv1.png)
![Failover Recovery](images/failover-recovery.png)

Despite the deliberate link failure (`eth1`), the continuous ping shows **0% packet loss**. The EVPN multihoming control plane instantly redirected the traffic flow to the remaining active link (`eth2`), demonstrating true carrier-grade High Availability.
