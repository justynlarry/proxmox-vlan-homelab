# Transitioning a Proxmox Cluster from a Flat Network to a VLAN
![Proxmox](https://img.shields.io/badge/Proxmox-ED1C24?style=for-the-badge&logo=proxmox&logoColor=white)
![OPNsense](https://img.shields.io/badge/OPNsense-EF4F1A?style=for-the-badge&logo=opnsense&logoColor=white)
![CARP](https://img.shields.io/badge/CARP-800000?style=for-the-badge&logo=freebsd&logoColor=white)

# Requirements
Switch: Must be a managed switch, I'm using a Netgear GS108Ev4, VLAN-capable -> but not capable of handling the demands of CARP/High Availability.
  Until a new more robust Router has been acquired, this project is on hold.
VLANs configured: Yes (e.g., Default, Management, Cluster, Monitoring, Security, etc.).
Current Router Mode: Advanced 802.1Q VLAN.

# Project Overview

#Goal: Transition a 4-node Proxmox cluster and additional bare-metal server to VLANs and deploy two OPNsense VMs in a CARP/HA configuration.
#High-Level Plan
  - Move Proxmox nodes to dedicated VLANs (Management, Cluster).
  - Add bare-metal Ubuntu server to VLANs.
  - Deploy OPNsense HA (CARP) with two nodes using dual-NIC routing.
  - Configure trunk ports for VLAN traffic, leaving certain access ports (e.g., Wi-Fi extender) on VLAN 1.

# VLAN/Subnet Map
# VLAN ID	Purpose	   Subnet	        Gateway
1	  Default/Mgmt	  192.168.0.0/24  192.168.0.1
10	Management	    10.0.10.0/24	  10.0.10.X
20	Cluster	        10.0.20.0/24	  10.0.20.X
30	Monitoring	    10.0.30.0/24	  10.0.30.X
40	Storage	        10.0.40.0/24	  10.0.40.X
50	DMZ	            10.0.50.0/24	  10.0.50.X
70	LockBox	        10.0.70.0/24	  10.0.70.X
90	HASync	        10.0.90.0/24	  10.0.90.X

# CARP VIP Gateways
# VLAN    	CARP VIP
LAN	        10.0.10.254
Cluster	    10.0.20.254
Monitor	    10.0.30.254
Storage	    10.0.40.254
DMZ	        10.0.50.254
LockBox	    10.0.70.254
HASync	    10.0.90.254


# Phase 1 – Configure Proxmox Node Interfaces

Backup network interfaces first:

```bash
cp /etc/network/interfaces /etc/network/interfaces.bak
```

Example: OPNsenseRed-Primary Node (pveRed)

```bash
auto lo
iface lo inet loopback

# NIC connected to WAN Access Port
iface enp3s0 inet manual

# Linux Bridge for WAN -> vtnet0 in OPNsense
auto vmbr0
iface vmbr0 inet manual
    bridge-ports enp3s0
    bridge-stp off
    bridge-fd 0

# LAN Trunk -> vtnet1 in OPNsense
iface enp1s0f1 inet manual

auto vmbr1
iface vmbr1 inet manual
    bridge-ports enp1s0f1
    bridge-stp off
    bridge-fd 0
    bridge-vlan-aware yes
    bridge-vids 2-4094

# VLAN 10 - Management
auto vmbr1.10
iface vmbr1.10 inet static
    address 10.0.10.11/24
    gateway 10.0.10.1

# VLAN 20 - Cluster
auto vmbr1.20
iface vmbr1.20 inet static
    address 10.0.20.11/24

source /etc/network/interfaces.d/*
```

Example: OPNsenseGreen-Secondary Node (pveGreen)
bash
```
auto lo
iface lo inet loopback

# WAN
iface enp1s0 inet manual

auto vmbr0
iface vmbr0 inet manual
    bridge-ports enp1s0
    bridge-stp off
    bridge-fd 0

# LAN Trunk
iface enp3s0 inet manual

auto vmbr1
iface vmbr1 inet manual
    bridge-ports enp3s0
    bridge-stp off
    bridge-fd 0
    bridge-vlan-aware yes
    bridge-vids 2-4094

# VLAN 10 - Management
auto vmbr1.10
iface vmbr1.10 inet static
    address 10.0.10.12/24
    gateway 10.0.10.2

# VLAN 20 - Cluster
auto vmbr1.20
iface vmbr1.20 inet static
    address 10.0.20.12/24

source /etc/network/interfaces.d/*
```

Apply changes:
```bash
ifreload -a
```

# Phase 2 – Proxmox GUI Configuration

Navigate to Node -> System -> Network.

Edit vmbr1 (LAN trunk), check VLAN Aware, save.

Verify interfaces:
```bash
ip route
```

Should show both the default VLAN 10 and the cluster VLAN 20 routes.

Phase 3 – Switch Configuration (Sample)
Port	Device	VLAN/Role
1	pveGreen	WAN
2	pveRed	WAN
3	pveGold	Flat Network
4	pveBlack	Flat Network
5	pveGreen	LAN
6	pveRed	LAN
7	bigBlue	LAN
8	ISP Access	Access VLAN 1

Remaining VLANs are configured similarly, each with their own VLAN ID and subnet.

Phase 4 – OPNsense VM Installation

Create two VMs in Proxmox: OPNsenseRed-Primary and OPNsenseGreen-Secondary.

Assign interfaces as per VLAN bridge mapping:

OPNsense VM	Management IP	Cluster IP	WAN Interface	LAN Interface	NIC Mapping
OPNsenseRed-Primary	10.0.10.11	10.0.20.11	vmbr0 -> vtnet0	vmbr1 -> vtnet1	enp1s0f0 / enp3s0
OPNsenseGreen-Secondary	10.0.10.12	10.0.20.12	vmbr0 -> vtnet0	vmbr1 -> vtnet1	enp1s0 / enp3s0

Configure VLAN interfaces in OPNsense as per the VLAN/Subnet Map.

Configure DHCP on OPNsenseRed-Primary for required VLANs (e.g., DMZ50), mirrored to Green via CARP sync.

Phase 5 – CARP / HA Configuration

Define CARP VIPs for each VLAN (LAN, Cluster, Monitoring, Storage, DMZ, LockBox, HASync).

Sync OPNsenseGreen-Secondary from the primary for configuration and firewall rules.

Verify failover functionality (note: limited by switch capability; full HA requires a CARP-capable switch).

Notes

Switch Limitation: Netgear GS108Ev4 cannot handle CARP failover properly. Full HA testing requires a switch supporting multicast/failover.

Gateway Rules: Only one default gateway per node; VLAN interfaces should not configure additional gateways unless using policy routing.

VLAN Awareness: Ensure bridge-vlan-aware yes and correct bridge-vids ranges for trunk ports.
