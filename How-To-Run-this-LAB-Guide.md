## 9. How to Run This Lab (From Scratch)

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
git clone [https://github.com/Nabil-SLF/EVPN-VXLAN-Datacenter-Nokia-SRL.git](https://github.com/Nabil-SLF/EVPN-VXLAN-Datacenter-Nokia-SRL.git)
cd EVPN-VXLAN-Datacenter-Nokia-SRL
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
