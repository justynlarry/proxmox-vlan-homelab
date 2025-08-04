# Transitioning a Proxmox Cluster from a Flat Network to a VLAN
![Proxmox](https://img.shields.io/badge/Proxmox-ED1C24?style=for-the-badge&logo=proxmox&logoColor=white)
![OPNsense](https://img.shields.io/badge/OPNsense-EF4F1A?style=for-the-badge&logo=opnsense&logoColor=white)
![CARP](https://img.shields.io/badge/CARP-800000?style=for-the-badge&logo=freebsd&logoColor=white)

# Requirements
Switch: Must be a managed switch, I'm using a Netgear GS108Ev4, VLAN-capable.
VLANs configured: Yes (e.g., Default, Management, Cluster, Monitoring, Security, etc.).
Current Router Mode: Basic 802.1Q VLAN.

# Transition plan:
- Move 4-node Proxmox cluster to a dedicated VLAN (e.g., VLAN 20 - "Cluster").
- Add bare-metal Ubuntu server to VLAN.
- Use OPNsense HA (CARP) setup with:
- One Node using traditional dual-NIC routing.
- One Node using Router on a Stick (single NIC with VLAN tags).
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

# Phase 2:  Install OPNsense as a 'Router on a Stick'

#2.1 Download OPNsense
- Download the .iso file from OPNsense:  https://opnsense.org/download/
- Upload this to your Proxmox ISO storage.

# 2.2 Create the VM:
- In Proxmox, click Create VM, and give it a unique ID and name.
- OS: Select Use CD/DVD disc image and choose the OPNsense ISO you uploaded.
- System: Leave the defaults unless you have specific needs.
- Disks: A small disk (16-32 GB) is sufficient for OPNsense.
- CPU: 1-2 cores is adequate for this setup.
- Memory: 1-2 GB of RAM is more than enough for a basic setup.
- Network: This is the most important part:
    - Bridge: Select the vmbr0 bridge you created earlier.
    - VLAN Tag: Leave this field blank. By leaving it blank, you're telling Proxmox to pass all VLAN-tagged traffic from the vmbr0 bridge to the VM's virtual NIC.
    - Model: Use VirtIO (virtio) for better performance.

# 2.3 Boot and Install:
- Start the VM and open the console.
- Follow the OPNsense installer prompts. The default settings are usually fine, hit <enter> for LAGG, WAN, VLAN setup options.

The installer will prompt you to configure network interfaces. You'll only see one interface which should be vtnet0, if it's not present, assign vtnet0 as your LAN interface.
    - Do not configure a WAN interface at this stage. You're using a single interface for everything.
    
OPNsense default credentials to enter installation:
- Username: installer
- Password: opnsense

Use these credentials to log in. You will then be able to proceed with the installation process.
    - If you somehow get past the installer and it asks for a login to the console:
        - Username: root
        - Password: opnsense

- Choose your keyboard layout
- Choose UFS Installation
- Select your Hard Disk
  
* After the installation is complete, you should IMMEDIATELY change these default passwords. The web GUI will walk you through setting a new root password as part of the initial setup wizard.

Once installation is complete, remove the ISO and reboot the VM.

# 2.4 ***UNDER CONSTRUCTION***
Comprehensive Blueprint: OPNsense "Router on a Stick" on Proxmox
This guide consolidates all the steps, troubleshooting, and best practices we've discussed into a single, easy-to-follow plan.

Phase 1: Proxmox and OPNsense VM Setup
Goal: Create an OPNsense VM and configure the underlying Proxmox network to handle VLAN-tagged traffic.

Proxmox Network Bridge Configuration:

Log into your Proxmox web GUI.

Go to Datacenter -> <Your Proxmox Node> -> System -> Network.

Identify your physical NIC (enp8s0 in your case) that will connect to your managed switch.

Click Create -> Linux Bridge.

Name: vmbr0

Bridge Ports: enp8s0

VLAN aware: Check this box.

Click Create and then Apply Configuration.

Create the OPNsense VM:

Click Create VM.

General: Give it a name (e.g., opnsense-router-on-a-stick).

OS: Select the OPNsense ISO you've uploaded.

System: Defaults are fine.

Disks: Use UFS as the file system during installation. A 16-32 GB disk is sufficient.

CPU: 1-2 cores.

Memory: 1-2 GB.

Network:

Bridge: Select vmbr0.

VLAN Tag: Leave this field blank.

Model: Select VirtIO (virtio). This is crucial for the installer to detect the NIC.

OPNsense Installation:

Start the OPNsense VM and open the console.

Log in with the default credentials: installer / opnsense.

Follow the installation prompts.

When asked about the file system, choose UFS.

When asked to assign interfaces, manually type vtnet0 and assign it to the LAN interface.

Do not assign a WAN interface.

Complete the installation and reboot.

Phase 2: OPNsense Initial Configuration
Goal: Establish network access to the OPNsense GUI and configure your VLANs.

Initial Access to OPNsense Web GUI:

On a separate machine or a temporary VM, connect to the vmbr0 bridge (with no VLAN tag).

Manually set the client's IP to be in the same subnet as OPNsense's default LAN IP (192.168.1.1). For example, use 192.168.1.100.

In a web browser on that client, navigate to https://192.168.1.1.

Log in with the default credentials: root / opnsense.

Change the root password immediately.

Assign VLAN Interfaces:

Go to Interfaces -> Assignments -> VLANs.

Click + to add a VLAN for each of your networks (e.g., Management, Cluster).

Parent interface: vtnet0

VLAN tag: The VLAN ID (e.g., 10 for Management, 20 for Cluster, etc.).

Description: A meaningful name for clarity.

Go to Interfaces -> Assignments and assign each of these newly created VLANs to a new interface (e.g., OPT1, OPT2).

Configure Each VLAN Interface:

Go to Interfaces -> [Your VLAN Interface] (e.g., Management, Cluster).

Check the "Enable" box.

Description: A clear name.

IPv4 Configuration Type: Static IPv4.

IPv4 Address: A unique IP address for that VLAN, typically the first usable IP (e.g., 10.0.10.1/24 for Management, 10.0.20.1/24 for Cluster).

Leave all other settings at their defaults for now.

Repeat for each VLAN.

Click Save and then Apply changes.

Note: The WAN/CARP VLAN (VLAN 99) should be created but not enabled or configured at this time. This is for your future redundant setup.

Phase 3: Service and Firewall Configuration
Goal: Make your VLANs functional by enabling DHCP and allowing traffic to pass.

Configure DHCP for each VLAN:

Go to Services -> DHCPv4 -> [Your VLAN Interface].

Check the "Enable" box.

Subnet: This will be pre-filled (e.g., 10.0.20.0).

Subnet mask: This will be pre-filled (255.255.255.0).

Range: Set a range for dynamic IP addresses (e.g., 10.0.20.100 to 10.0.20.200).

Click Save.

Look for a button or notification to "Start Service" and click it to apply the changes.

Repeat for each active VLAN.

Configure Firewall Rules for Initial Testing:

Go to Firewall -> Rules -> [Your VLAN Interface].

Click + to add a new rule.

Action: Pass.

Interface: Your VLAN interface (e.g., Cluster).

Direction: in.

Protocol: any.

Source: Select [Your VLAN] net (e.g., Cluster net).

Destination: any.

Log: Check this box for easy troubleshooting.

Description: A clear name (e.g., ALLOW all traffic from Cluster VLAN).

Click Save and then Apply changes.

Repeat for each active VLAN.

Phase 4: Final Testing
Goal: Validate that your setup is working correctly.

Managed Switch Configuration:

Log into your managed switch.

Configure the port connected to your Proxmox host as a trunk port that is a tagged member of all your OPNsense VLANs.

Configure an access port for each VLAN you want to test. Set the PVID for each port to the corresponding VLAN ID (e.g., PVID 10 for a port you'll plug a management computer into).

Test Connectivity:

Plug a device into one of the configured access ports.

Verify that the device automatically receives an IP address from the correct OPNsense DHCP server.

Test internet connectivity.

Check the Firewall -> Log Files -> Live View in OPNsense to see your traffic being passed.




