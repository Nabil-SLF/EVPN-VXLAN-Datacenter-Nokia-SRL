# EVPN-VXLAN-Datacenter-Nokia-SRL
# 🌐 Enterprise Datacenter Fabric: EVPN-VXLAN with All-Active Multihoming (Nokia SR Linux)

## 📌 1. Project Overview
This project demonstrates the design, deployment, and verification of a modern, highly available Datacenter Fabric using **Nokia SR Linux (SRL v25)**. 

The architecture is built on a robust **OSPFv2 underlay** and an **iBGP EVPN/VXLAN overlay**, providing seamless Layer 2 and Layer 3 virtualization via **Asymmetric IRB**. As part of a complete, production-ready fabric design, we also integrated **EVPN All-Active Multihoming (ESI)** at the access layer to optimize link utilization and ensure fast failover, effectively eliminating the need for legacy STP.

## ☁️ 2. Lab Environment & Prerequisites
This deployment was tested on a cloud-based environment to handle multiple Nokia SRL instances smoothly.

* **Cloud Provider:** Microsoft Azure
* **Virtual Machine:** Ubuntu Linux (Size: Standard D2s v3 - 2 vCPUs, 8 GiB Memory)
* **Container Runtime:** Docker (Version 27.5.1)
* **Orchestration Tool:** Containerlab (Version 0.64.0)
* **Network OS (NOS):** Nokia SR Linux (`ghcr.io/nokia/srlinux`)

## 🏗️ 3. Network Architecture & Topology
The setup uses a standard 2-Spine / 2-Leaf architecture.

![Topology Design](Topology/topology-design.png)

### 📊 IP & BGP Allocation Matrix
All nodes operate within the same Autonomous System (**AS 65001**) to form an iBGP EVPN Overlay.

| Node        | Role                         | System IP (Underlay)      | BGP ASN |
| :---        | :--------------------------- | :-----------------------  | :-------|
| **spine1**  | Spine / iBGP Route Reflector | `1.1.1.1/32`              | `65001` |
| **spine2**  | Spine / iBGP Route Reflector | `2.2.2.2/32`              | `65001` |
| **leaf1**   | Leaf / VTEP                  | `10.10.10.1/32`           | `65001` |
| **leaf2**   | Leaf / VTEP                  | `10.10.10.2/32`           | `65001` |

## 🛠️ 4. Underlay Network (OSPF)
**OSPFv2 (Area 0)** is configured as the Interior Gateway Protocol (IGP). It runs across all physical links and system interfaces, ensuring loop-free reachability between all `System IPs` (`1.1.1.1`, `2.2.2.2`, `10.10.10.1`, `10.10.10.2`). This is essential for establishing BGP peerings and VXLAN tunnels.

## 🌐 5. Overlay Network (iBGP EVPN)
The control plane is driven by **MP-iBGP** with the EVPN address family:
* `spine1` and `spine2` act as **Route Reflectors**, eliminating the need for a full BGP mesh between the leafs.
* The EVPN control plane handles endpoint discovery natively using **EVPN Type 2 (MAC/IP)** routes.

## 🏢 6. Dataplane & Tenancy (VXLAN & Asymmetric IRB)
Data forwarding over the IP core is encapsulated in **VXLAN** tunnels.
* **L2 Virtualization (MAC-VRF):** Segmented into `vlan10` and `vlan20`.
* **L3 Routing (IP-VRF & Asymmetric IRB):** We implemented Asymmetric IRB using `ip-vrf-1`. Routing occurs at the ingress leaf, and traffic is bridged across the VXLAN tunnel to the egress leaf.
* **Anycast Gateway:** Hosts share the same Default Gateway IP configured on the `irb1` subinterfaces (`192.168.10.254` for VLAN 10 and `192.168.20.254` for VLAN 20).

## 🔗 7. All-Active Multihoming (EVPN ESI)
Servers are connected to both leafs using **LACP (802.3ad) port-channels**. 
By configuring an Ethernet Segment Identifier (ESI, `00:00:00:00:00:11`) and `bgp-instance 1`, both `leaf1` and `leaf2` operate in an **Active-Active** state for the same server. This allows load balancing and eliminates single points of failure.

## ✅ 8. Complete Verification & Testing Guide
To maintain readability, the complete end-to-end testing procedure—including control plane verification (OSPF, iBGP, EVPN), dataplane validation (MAC/ARP tables, VXLAN tunnels), and the **0% packet loss failover test**—has been detailed in a separate document.

👉 **[Click here to view the comprehensive Verification & Testing Guide](VERIFICATION.md)**

## 🚀 9. How to Run This Lab (From Scratch)

**Step 1: Prepare the Ubuntu VM (Install Docker & Containerlab)**
If you are starting from a fresh Ubuntu VM, run these commands to install the required dependencies:
```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install Docker
curl -fsSL [https://get.docker.com](https://get.docker.com) -o get-docker.sh
sudo sh get-docker.sh

# Install Containerlab
bash -c "$(curl -sL [https://get.containerlab.dev](https://get.containerlab.dev))"
```

**Step 2: Pull the Nokia SR Linux Image**
The router image is not hosted in this repository. You must pull it directly from the Nokia GitHub Container Registry:
```bash
sudo docker pull ghcr.io/nokia/srlinux:latest
```

**Step 3: Clone This Repository**
Download the topology and configuration files from this project:
```bash
git clone [https://github.com/YOUR_USERNAME/Nokia-SRL-EVPN-Datacenter.git](https://github.com/YOUR_USERNAME/Nokia-SRL-EVPN-Datacenter.git)
cd Nokia-SRL-EVPN-Datacenter
```

**Step 4: Deploy the Fabric**
Deploy the lab using Containerlab. This will spin up the containers, wire the topology, and apply the configurations automatically:
```bash
sudo clab deploy -t topo.yml
```

**Step 5: Verify & Access Nodes**
Containerlab automatically configures SSH access to the nodes. You can securely log into any router using the `admin` user:
```bash
# Access spine1 via SSH
ssh admin@clab-pro-evpn-spine1

# Access leaf1 via SSH
ssh admin@clab-pro-evpn-leaf1
```
*(Note: Default password for Nokia SR Linux is `NokiaSrl1!` or it may authenticate automatically via SSH keys injected by Containerlab).*

Alternatively, you can access the shell directly via Docker:
```bash
docker exec -it clab-pro-evpn-leaf1 sr_cli
```

**Step 6: Teardown**
When you are done testing, you can destroy the lab and clean up the resources:
```bash
sudo clab destroy -t topo.yml --cleanup
```
