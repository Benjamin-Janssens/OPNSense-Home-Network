<div align="center">
  <b><h1>OPNSense Firewall Config</h1></b>
  <img width="400" height="400" alt="eMEUKrZ3_400x400" src="https://github.com/user-attachments/assets/f44ccb6c-c2f5-45bc-a925-835fa2319b99" />
</div>

<h2> Interface Rules </h2>


<details>
<summary> Trusted LAN Interface </summary>
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
<summary> Trusted IoT Interface </summary>
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
<summary> Trusted LAN Interface </summary>
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
<summary> Management VLAN </summary>
<img width="2101" height="1042" alt="Management VLAN" src="https://github.com/user-attachments/assets/3c1eb8eb-75b4-4110-b62f-9fd540d7f167" />

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
