# Transitioning a Proxmox Cluster from a Flat Network to a VLAN
![Proxmox](https://img.shields.io/badge/Proxmox-ED1C24?style=for-the-badge&logo=proxmox&logoColor=white)
![OPNsense](https://img.shields.io/badge/OPNsense-EF4F1A?style=for-the-badge&logo=opnsense&logoColor=white)
![CARP](https://img.shields.io/badge/CARP-800000?style=for-the-badge&logo=freebsd&logoColor=white)

# Requirements
Switch: Must be a managed switch, I'm using a Netgear GS108Ev4, VLAN-capable.
VLANs configured: Yes (e.g., Default, Management, Cluster, Monitoring, Security, etc.).
Current Router Mode: Basic 802.1Q VLAN.

# Transition plan:
- Move 4-node Proxmox cluster to a dedicated VLAN (e.g., VLAN 11 - "Management", VLAN 21 - "Cluster").
- Add bare-metal Ubuntu server to VLAN.
- Use OPNsense HA (CARP) setup with:
- Two Nodes using traditional dual-NIC routing.
  
Eventually, most ports will be Trunk, except port 8 (Access to VLAN 1, connected to Wi-Fi extender).

# Phase 1:  Enabling VLan on the First Node:
Trunk Port with Native (Untagged) VLAN Allowed
Leave VLAN 1 untagged on the trunk port (on the switch).

vmbr0 receives untagged traffic → handled by Proxmox's main interface (e.g., eno1 with IP 192.168.0.X).
vmbr0.20 receives VLAN 20 tagged traffic → handled by Proxmox’s virtual VLAN interface with IP 10.0.20.X.
Result: Dual stack — same NIC handles both networks.

Proposed VLAN/Subnet Map:
```
VLAN ID    Purpose        Subnet              Gateway  
1          Default/Mgmt   192.168.0.0/24      192.168.0.1
10         Management     10.0.10.0/24        10.0.10.1
20         Cluster        10.0.20.0/24        10.0.20.1
30         Monitoring     10.0.30.0/24        10.0.30.1
40         Storage        10.0.40.0/24        10.0.40.1
50         General        10.0.50.0/24        10.0.50.1
60         ---            10.0.60.0/24        10.0.60.1
70         ---            10.0.70.0/24        10.0.70.1
80         Security       10.0.80.0/24        10.0.80.1
99         WAN / CARP     Public/RFC1918      N/A
```

# 1.1 Change Node interface configuration in /etc/network/interfaces
- First, make a backup of the interfaces file:
```bash
cp /etc/network/interfaces /etc/network/interfaces.bak
```
- Then alter the file to allow both flat and VLAN transmission
```bash
auto lo
iface lo inet loopback

iface enp8s0 inet manual

# Untagged Interface from setup (192.168.0.X)
auto vmbr0
iface vmbr0 inet static
    address 192.168.0.X/24
    gateway 192.168.0.1
    bridge-ports enp8s0
    bridge-stp off
    bridge-fd 0

# VLAN 20 Interface for Cluster (10.0.20.0/24)
auto vmbr0.20
iface vmbr0.20 inet static
    address 10.0.20.X     # Replace with your new VLAN IP Address
    netmask 255.255.255.0
#    gateway 10.0.20.1    # You can only have one default gateway, for the time being, this needs to be commented out

iface wlp7s0 inet manual

source /etc/network/interfaces.d/*
```

- Apply changes
```bash
ifrelaod -a
```

# 1.2 Update the VLAN Settings in the Proxmox GUI

- In the Proxmox GUI navigate to the Network tab for the node you're transitioning:<NODE> -> System -> Network
  - Highlight the Node's Linux Bridge and click 'Edit' (this will most likely be named vmbr0 in Proxmox), you should also see 'vmbr0.20' as Type Linux VLAN
  - On the right side of the the 'Edit: Linux Bridge' pop-up, click VLAN Aware, and 'OK'
file:///home/larryman/Pictures/Screenshots/Screenshot%20from%202025-08-04%2011-15-05.png<img width="450" height="auto" alt="screenshot" src="https://github.com/user-attachments/assets/354d23ee-1d47-4803-933d-008338b7bcbf" />

# 1.3 Switch the Mode on the appropriate port to 'Trunk(uplink)' and save, and verify in the terminal.
You should see both vmbr0 and vmbr0.20 in the ouput:
```bash
$ ip route
default via 192.168.X.X dev vmbr0 proto kernel onlink 
10.0.20.0/24 dev vmbr0.20 proto kernel scope link src 10.0.20.X 
<snipped>
192.X.X.X/24 dev vmbr0 proto kernel scope link src 192.X.X.X
``` 





