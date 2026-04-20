<h2><p align="center"><b> NETWORK DIAGRAM </b></p></h2>
<p align="center">
  <a href="https://github.com/user-attachments/assets/75bffa51-5037-4389-909a-3a1eb2ed1009">
    <img src="https://github.com/user-attachments/assets/75bffa51-5037-4389-909a-3a1eb2ed1009" width="1000">
  </a>
</p>

<b> Network Configuration Summary (Authoritative) </b>
<details>
<summary> Core Platform </summary>

  
    • Host: Proxmox VE   
    • Firewall VM: OPNsense  
    • Hardware: Protectli Intel N150 16GB LPDDR5 RAM 1TB NVMe M.2 PCIE SSD 4x intel I226-v 2.5 gbps NICs  
    • OS: Ubuntu Server  
    • Containerization: Docker Compose  
    • Switch: "Smart" Managed switch (802.1q VLAN-aware)  
    • Wireless: TP-Link Omada EAP650-D VLAN-aware AP with PPSK support  
</details>


<details>  
<summary> Physical & Logical Topology </summary>

  
#### Proxmox Host
	• Proxmox management IP: 192.168.100.2/24
	• Default gateway: 192.168.100.1
	• Proxmox management traffic lives on VLAN 100 (MGMT)
</details>


<details>
  <summary> Proxmox Interfaces & Bridges </summary> 

  
#### vmbr0 — (Direct Patch port)
	• Type: Linux bridge
	• Interface NIC Driver: Virtio
	• Mode: VLAN-aware = No
	• Addressing: Broadcast 192.168.254.1/30
	• Role: Patch Port
	• Attached NIC: lan (intel I226-v)
	• Connected to: OPNsense
#### vmbr1 — WAN Bridge
	• Type: Direct Interface PCIE passthrough
	• Interface NIC Driver: PCIE device
	• Mode: VLAN-aware = No
	• Addressing: IPoE
	• Role: WAN interace
	• Attached NIC: icg0 (intel I226-v)
	• Connected to: OPNsense
	
#### vmbr2 — Core LAN Trunk Bridge
	• Type: Linux bridge
	• Interface NIC Driver: Virtio
	• Mode: VLAN-aware = YES
	• Addressing: No IP address - Strictly Trunk Port (192.168.1.1/24, 192.168.2.1/24, 192.168.4.1/24, 192.168.5.1/24)
	• Role: Backbone trunk carrying all internal VLANs
	• Attached NIC: lan (intel I226-v)
	• VLANs carried: 10, 20, 40, 100 (and others as required)
	• Connected to: Managed switch trunk port, OPNsense
#### vmbr2.100 — Management VLAN Interface
	• Type: Linux VLAN sub-interface
	• Parent: vmbr2
	• VLAN ID: 100
	• IP: 192.168.100.2/24
	• Gateway: 192.168.100.1
	• Purpose: Proxmox host management access
	Proxmox does not have IPs on other VLANs.

#### vmbr3 — DMZ 
	• Type: Direct Physical interface
	• Interface NIC Driver: Virtio
	• Mode: VLAN-aware = No
	• Addressing: 192.168.6.100
	• Role: DMZ server, isolated via routing rules
	• Attached NIC: DMZ (intel I226-v)
	• VLANs carried: 10, 20, 40, 100 (and others as required)
	• Connected to: Managed switch trunk port
  
#### OPNsense VM
#### Interfaces
	• WAN: vmbr1 (direct WAN handoff) (PCIE passthrough)
	• LAN / Trunk: vmbr2 (VLAN trunk)
	• VLAN Interfaces on OPNsense:
		○ VLAN 100 — MGMT
		○ VLAN 10 — Trusted LAN
		○ VLAN 20 — Trusted IoT
		○ VLAN 40 — DMZ (example)
#### Routing
  	• OPNsense is default gateway for all internal VLANs
  	• Inter-VLAN routing controlled by firewall rules
</details>

<details>
<summary> VLANs & Subnets </summary>

  
#### Subnet	| Name |	Route | Config |	Gateway | VLAN
<p>• 192.168.1.x/24	| Trusted IoT LAN |	(defined in OPNsense) |	192.168.1.1 | 10</p>
<p>• 192.168.2.x/24 |	Trusted LAN |	(defined in OPNsense)	192.168.2.1 | 20</p>
<p>• 192.168.4.x/24 |	Dodgy IoT LAN |	(defined in OPNsense)	192.168.4.1 | 40</p>
<p>• 192.168.5.x/24 |	Work | (defined in OPNsense)	192.168.5.1 | 50</p>
<p>• 192.168.6.x/24 |	DMZ |	(defined in OPNsense) |	192.168.6.1 | N/A</p>
</details>

<details> 
<summary> Switch Configuration </summary>

  
#### Switch Management
	• Switch management IP
		○ Lives on a native / untagged VLAN
		○ Flat “.1” management network
	• Physical management access port: Port 3
		○ Untagged / native VLAN only
		○ Used exclusively for switch administration
	• No routing done by switch (pure L2, management plane only)
 
#### Trunk Ports
#### <p>Port 10 — Core Trunk to Protectli / Proxmox </p>
	• Connected device: Protectli (Proxmox host)
	• Mode: 802.1Q trunk
	• Tagged VLANs:
		○ VLAN 10 — Trusted LAN / Cameras
		○ VLAN 20 — Trusted IoT
		○ VLAN 40 — Dodgy IoT
		○ VLAN 100 — Management
	• Untagged VLAN: none (or unused/native only)
	• Purpose: Backbone uplink carrying all internal VLANs to OPNsense via vmbr2
 
#### Access Ports
#### <p>Ports 6–8 — Cameras</p>
	• Mode: Access / untagged
	• PVID: VLAN 10
	• Devices: IP cameras
	• Traffic behavior:
		○ Untagged on the wire
		○ Tagged internally as VLAN 10
	• Gateway: OPNsense VLAN 10 interface
 
#### Desktop / Admin Ports
	• Mode: Access
	• PVID: VLAN 100 (MGMT)
	• Devices: Admin desktop / laptop
	• Purpose: Direct access to infrastructure services
  
#### Management Plane Summary
#### <p>The following all live in VLAN 100 (MGMT):</p>
	• Proxmox host management (192.168.100.2)
	• OPNsense management interface (192.168.100.1)
	• Omada Controller (192.168.100.4)
Only MGMT VLAN has direct access to infrastructure services.

#### Wireless Configuration
#### <p>Access Point</p>
	• VLAN-aware
	• Single SSID using PPSK
	• PPSK maps clients into VLANs:
		○ Trusted LAN → VLAN 10
		○ Trusted IoT LAN → VLAN 20
		○ Dodgy IoT LAN → VLAN 40
    ○ Work LAN → VLAN 50
    
</details>
