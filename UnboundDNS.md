<h2><p align="center"><b> UnboundDNS Filtering </b></p></h2>
<p align="center">
  <a href="https://github.com/user-attachments/assets/32cf0cde-7502-4bab-8aa6-1a10daa4c0c6">
    <img src="https://github.com/user-attachments/assets/32cf0cde-7502-4bab-8aa6-1a10daa4c0c6" width="1000">
  </a>
</p>

#### Services: Unbound DNS Configuration

General Settings
- Status: Enabled
- Listen Port: 53
- Network Interfaces: * DMZ, DODGY_IOT, MGMT_DIRECT, MGMT_VLAN, TRUSTED_IOT, TRUSTED_LAN, WAN, WorkVlan
- DNSSEC Support: Enabled

**Registration:** * Register ISC DHCP4 Leases: Enabled (allows resolution of local hostnames assigned via DHCP)
- Local Zone Type: Transparent
#### DNS over TLS (DoT) - To ensure privacy for outbound DNS queries, all traffic is encrypted before leaving the network
- Use System Nameservers: Disabled (forces use of custom forwarding)
#### Custom Forwarding:
- Server IP: 1.1.1.1
- Server Port: 853
- Provider: Cloudflare

#### DNS Blacklisting (DNSBL) - Integrated threat intelligence to block malicious domains at the resolution stage.
- Status: Enabled
#### Type of DNSBL:
- Abuse.ch: ThreatFox IOC database
- AdGuard List: Comprehensive ad and tracking protection
- Steven Black List: Consolidated host-based blocking
- Command: Block ads and malware
