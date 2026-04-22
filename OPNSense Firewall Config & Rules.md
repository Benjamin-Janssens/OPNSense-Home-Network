<div align="center">
  <b><h1>OPNSense Firewall Config</h1></b>
  <img width="400" height="400" alt="eMEUKrZ3_400x400" src="https://github.com/user-attachments/assets/f44ccb6c-c2f5-45bc-a925-835fa2319b99" />
</div>
	
<h2> Interface Assignments </h2>

<details>
<summary> Interface Assignments </summary>
<img width="2104" height="1042" alt="Interface assginments" src="https://github.com/user-attachments/assets/35b722dd-dc6c-4cc2-9a67-a0b55af16ed1" />

#### Proctectli hardware is running PROXMOX as the operating system, it is not virtualized
#### OPNsense is a VM inside PROXMOX that IS virtualized
#### Facilitating connection between these devices requires "bridging"(virtual switch) in order to work
#### VLANS sit ontop of these virtual switch ports as seen in interfaces overview

#### The literal name of the ports when shipped:   (Label on protectli) (the port on the box)
Port 4 = Enx646****55494  
Port 3 = Enx646****55493  
Port 2 = Enx646****55492  
Port 1 = Enx646****55491  

#### Proxmox re-labled the port names for simplicity:    (Proxmox GUI -> node -> network) (The linux rename)
Port 4 - > DMZ  
Port 3- > LAN  
Port 2 - > WAN  
Port 1  - > MANAGEMENT  

#### Proxmox bridges connected to physical ports:    (Proxmox GUI -> node -> network) ( The virtual switch ports)
DMZ - > vmbr3  
LAN - > vmbr2  
WAN - > vmbr1  
MANAGEMENT - > vmbr0  

#### Proxmox bridges are assigned to OPNSense ports (virtually plugging the back of the OPNsense box into the Proxmox switch):    (Proxmox GUI -> node -> OPNsense VM -> hardware)  (The VMS plug into the switch)
Vmbr3 - > net3   
Vmbr2 - > net2  
Vmbr1 - > net1  
Vmbr0 - > net0  

#### OPNsense interfaces are typically matched 1:1 with the linux bridges (OPNsense -> interfaces -> assignments -> Compare MAC address with proxmox -> OPNSense -> hardware ) (virtual ports doing firewall stuff)
Net3 -> BC:24:11:57:ED:A6 -> vtnet3  
Net2 -> BC:24:11:71:06:D8 -> vtnet2  
Net1 -> BC:24:11:22:57:41 -> vtnet1  
Net0 -> BC:24:11:3A:72:6B -> vtnet0  

#### OPNSENSE has configured interfaces/gateways ontop of these virtual ports (OPNsense GUI -> interfaces -> overview)
Port 4 -> Vtnet3 -> DMZ Interface  
Port 3 -> Vtnet2 -> Unassigned (Assigned via VLAN for a "tagged" trunk port)  
Port 2 -> Vtnet1 -> WAN Interface  
Port 1 -> Vtnet0 -> MGMT_DIRECT Interface  

#### OPNsense has configured VLAN interfaces ontop of unassigned port 2 as the managed switch will handle layer 2, the VLAN interfaces have an IP/gateway so VLAN traffic can leave and Enter
Vtnet2 -> VLAN1 TRUSTED_IOT Interface  
Vtnet2 -> VLAN2 TRUSTED_LAN Interface  
Vtnet2 -> VLAN3 DODGY_IOT Interface  

</details>

<h2> Interface Rules </h2>


<details>
<summary> Trusted LAN VLAN Interface </summary>
<img alt="TRUSTED LAN" src="https://github.com/user-attachments/assets/a59a1376-1a85-4a39-bd12-60d18602e6ff" />
The TRUSTED_LAN firewall configuration implements a structured, top-down security policy designed to enforce local DNS control, facilitate secure administrative access, and manage granular inter-VLAN routing before permitting external internet egress. By utilizing aliases for ports and network groups, the ruleset maintains a high level of readability and scalability.

#### Detailed Rule Breakdown
	• Force OPNsense DNS
		○ Reason: Redirects all DNS traffic from the subnet to the OPNsense local address. This prevents devices from using hardcoded third-party DNS servers, ensuring all queries are filtered or logged by the local resolver.
	• Allow Management ports to firewall
		○ Reason: Permits access to the OPNsense web GUI and SSH from the Trusted LAN using a specific Management_Ports alias. This ensures administrators can configure the system without opening unnecessary services.
	• Allow Pings to firewall
		○ Reason: Enables ICMP traffic to the OPNsense gateway. This is essential for basic connectivity testing and "heartbeat" monitoring from the client side.
	• Talk to DMZ via SSH & Web Ports
		○ Reason: Allows administrative and management traffic to flow from the Trusted LAN into the DMZ. It restricts communication specifically to the DMZ_Ports alias to prevent full exposure of the DMZ subnet.
	• Allow access to management network
		○ Reason: Grants the Trusted LAN full access to the MGMT_DIRECT and MGMT_VLAN subnets. This is critical for managing the Proxmox hypervisor and other core infrastructure components.
	• Allow Access to IoT services
		○ Reason: Permits the Trusted LAN to initiate connections to the TRUSTED_IOT network, allowing the user to control smart home devices while maintaining a one-way security boundary.
	• Allow Ping to DMZ
		○ Reason: Specifically allows ICMP traffic to the DMZ network to verify server availability and troubleshoot routing issues across the VLAN boundary.
	• ProtonVPN DNS
		○ Reason: Permits DNS traffic (10.2.0.0/24) specifically for the ProtonVPN gateway, to support split-tunneling or specific privacy-oriented routing requirements.
	• Allow Ping to Internet
		○ Reason: Allows users to ping external IP addresses (any address that is ! RFC1918). This helps verify if an internet connection is active while still blocking pings to other private internal subnets.
	• Allow TCP/UDP to Internet
		○ Reason: The primary "catch-all" for web browsing and general usage. It allows standard internet traffic to any destination outside of the private IP range (! RFC1918_Networks), effectively creating a "kill switch" that prevents unauthorized lateral movement to other internal VLANs.

</details>

<details>
<summary> Trusted IoT VLAN Interface </summary>
<img width="2088" height="512" alt="image" src="https://github.com/user-attachments/assets/f853dcc9-fd5b-4ba4-b152-d5f5fa90e708" />
The TRUSTED_IOT interface is configured with a focus on strict isolation and functional whitelisting. The policy architecture ensures that smart devices and appliances have the necessary connectivity for cloud services and local controllers while being explicitly barred from interacting with the network's administrative core and management hardware.

#### Detailed Rule Breakdown
	• Force OPNsense DNS
		○ Reason: Hijacks DNS requests from all devices on this interface and directs them to the OPNsense local resolver. This ensures that IoT devices cannot bypass local filtering or use unauthorized external DNS providers.
	• Allow TP-Link eco-system to speak to omada
		○ Reason: Permits traffic specifically to the Omada Controller ($192.168.100.4$) via the TP_Link_Port alias. This is required for the centralized management, provisioning, and monitoring of networking hardware.
	• Allow Phones to Media Server
		○ Reason: Grants access to the media server ($192.168.6.100$) using the Arrstack port alias. This allows IoT-based streaming clients or mobile devices to access media services without exposing the entire destination subnet.
	• Allow IoT to Internet
		○ Reason: Enables outbound internet connectivity by permitting traffic to any destination that is not defined within the RFC1918_Networks alias. This provides a path for firmware updates and cloud synchronization while preventing internal lateral movement.
	• Block IoT sniffing the firewall
		○ Reason: Actively prevents devices on this segment from probing or connecting to the firewall’s own management interfaces. This reduces the attack surface by making the gateway "invisible" to the IoT subnet.
	• Block all non expected IOT traffic to mgmt network
		○ Reason: Explicitly drops any traffic destined for the MGMT_DIRECT or MGMT_VLAN subnets. This serves as a primary security boundary to ensure IoT devices cannot communicate with infrastructure management layers.
	• Block IOT from hitting the switch
		○ Reason: Denies all traffic directed at the physical switch management IP ($192.168.1.2$). This prevents devices from attempting to exploit or log into the hardware that handles the network’s physical switching.
	• Block IOT from hitting the switch (Extended)
		○ Reason: A secondary security enforcement rule that ensures various source segments (including Trusted IoT, Dodgy IoT, and the DMZ) are strictly blocked from accessing the core switch administration IP.
</details>

<details>
<summary> Trusted Managemennt VLAN Interface </summary>
<img width="2228" height="1042" alt="Management VLAN" src="https://github.com/user-attachments/assets/85cecaf8-17b9-4bb9-be93-8eaa214c465f" />

The MGMT_VLAN interface acts as the administrative backbone of the network. The configuration is designed to be highly restrictive, ensuring that core infrastructure components—such as the hypervisor and controllers—can communicate with their respective endpoints while maintaining a hard boundary against other internal subnets. It prioritizes internal reachability for management tools before enforcing a final lockdown.


#### Detailed Rule Breakdown
	• Force OPNsense DNS
		○ Reason: Captures all DNS traffic from the management network and directs it to the local OPNsense resolver. This ensures that even administrative infrastructure follows local name resolution policies and logging.
	• Allow pings to the firewall
		○ Reason: Permits ICMP traffic to the gateway. This is necessary for verifying the health of the management interface and for diagnosing network latency or connectivity issues at the core.
	• Allow MGMT devices to speak
		○ Reason: A broad internal rule allowing devices within the MGMT_VLAN subnet to communicate with one another. This is essential for cluster communication, node-to-node management, and inter-service synchronization.
	• Allow OMADA to speak to IOT net
		○ Reason: Specifically permits the Omada Controller ($192.168.100.4$) to initiate connections into the TRUSTED_IOT network. This allows the controller to manage access points, switches, and other hardware residing in the IoT segment.
	• Block Trusted LAN
		○ Reason: An explicit "Block" rule targeting traffic from the TRUSTED_LAN. This creates a one-way street: while the Trusted LAN may be allowed to access Management (via its own interface rules), this rule prevents the Management network from initiating unauthorized sessions back into the Trusted LAN.
	• Allow MGMT out on 443, 80
		○ Reason: Grants specific hosts in the MGMT_HOSTS alias access to the internet over standard web ports (Web_Ports). This is limited to non-private addresses (! RFC1918_Networks) to facilitate software updates and repository access for management servers.
	• Allow proxmox time sync
		○ Reason: Permits the management network to communicate with the NetInfra alias. This is typically used to allow the Proxmox hypervisor or other infrastructure to reach NTP (Network Time Protocol) servers for accurate system clock synchronization.
	• Lock MGMT down
		○ Reason: A final "Block All" rule that drops any traffic from the MGMT_VLAN that hasn't been explicitly permitted by the rules above. This serves as the ultimate security failsafe, ensuring no "leaking" traffic can reach other parts of the network or the internet.
</details>

<details>
<summary> DMZ VLAN Interface </summary>
The DMZ interface is configured as a highly restricted "De-Militarized Zone," designed to host services that require external connectivity while being strictly barred from accessing any internal private resources. The ruleset is lean and functional, prioritizing local infrastructure services (DNS and NTP) before permitting outbound internet access through a definitive security kill switch.
<img width="2086" height="1042" alt="DMZ" src="https://github.com/user-attachments/assets/5fed33c1-63d1-473a-87f7-c16fa911b2f6" />


#### Detailed Rule Breakdown
	• Force OPNsense DNS
		○ Reason: Directs all DNS traffic from the DMZ subnet to the local OPNsense interface address. This ensures that hosted servers use the controlled internal resolver for name resolution rather than bypassing it via external providers.
	• TimeSync
		○ Reason: Permits NTP traffic ($Port 123$) from the DMZ network to the OPNsense gateway. Maintaining accurate system clocks is critical for log correlation, certificate validation, and the general stability of hosted services.
	• Allow Ping to Net
		○ Reason: Enables ICMP traffic from the DMZ to external destinations (any address that is ! RFC1918_Networks). This allows for connectivity testing to the internet while preventing the DMZ from probing other internal subnets.
	• Kill Switch - Block All to internal network
		○ Reason: The primary security anchor for this interface. It permits all other traffic (TCP/UDP) to reach the internet, but only if the destination is not part of the RFC1918_Networks alias. This effectively "kills" any attempt by a compromised DMZ host to initiate a connection to the LAN, IoT, or Management segments.
</details>
