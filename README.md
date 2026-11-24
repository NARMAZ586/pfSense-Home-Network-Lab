# pfSense-Home-Network-Lab
This lab demonstrates pfSense running as a router for a home network, through virtualization on a proxmox host. Features such as VLAN segmentation, zone-style firewall policy, WireGuard, and Wazuh SIEM monitoring were included.

Include the main image here

Performed in Lab
- Routing and firewalling with pfSense (WAN/LAN, NAT, stateful rules)
- 802.1Q VLANs and router on a stick on a VLAN-aware Linux Bridge
- Zone-based firewall design implemented on an interface-based firewall
- DHCP scopes, static mappings and local DNS
- WireGuard remote-access VPN with least-privilege rules added
- SIEM integration: pfSense syslog and Wazuh agents

## Table of contents
1. Hardware
2. Architecture
3. Build Walkthrough
4. Verification

# Hardware
Include the image of the machine running proxmox here

| Component | Details | Role |
| --- | --- | ---|
| Host | HP EliteDesk 800 G3 MT, Intel i7-7700, 16GB RAM, 256 GB NVMe | Proxmox VE Hypervisor |
| Extended NIC | HP NC360T dual-port gigabit | pfSense WAN(port 1) and LAN(port 2) | 
| Built-in NIC | Intel gigabit | Proxmox managment | 
| Access Point | Home Router(TP-Link) changed to Access Point mode | Wi-Fi for the home | 
| Modem | ISP-provided | Public IP handed to pfSense |

## Cabling
| Physical port | Proxmox bridge | Connects to | 
| --- | --- | --- |
| NC360t port 1 | vmbr1 | Modem (pfSense WAN) | 
| NC360T port 2 | vmbr2 | Access Point (pfSense LAN) |
| Onboard NIC | vmbr0 | TP-Link (Proxmox management, 192.168.0.50) |

### Required to rebuild
| Requirement | Minimum | Why |
| --- | --- | --- |
| Physical NICs | 2 minimum, 3 is recommended | pfSense is the front of the network, it needs one WAN an one LAN por. A third NIC keeps Proxmox management reachable even if pfSense is down | 
NIC Brand | Intel | Intel based NICs are widely regarded as having the best driver support for pfSense |
| RAM | 16 GB with all features | Budget below | 
| Storage | Around 100 GBs needed | pfSense 32 Gb, each debian VM 8 GB(there are two), Wazuh. 50 GB |
| Access Point | Any router that supports Access Point mode, else a regular true access point can instead be used | pfSense performs the routing and DHCP leasing, the AP only provides Wi-Fi connectivity for the home |

### RAM budget
| VM | RAM | Notes |
| --- | --- | --- |
| pfSense | 3 Gb | More than enough to run firewall rules and Wireguard |
| debian-vlan10 / debian-vlan20 | 1 GB each | debian is a lightweight distro, so this is enough to run |
| Wazuh | 8 GB | Indexer, server and dashboard on one VM |
| Proxmox host | ~1 - 2 GB | |
| Total ~14 GB of 16 GB | |

Alternatives: instead of a desktop like this one a mini PC with two or more built-in NICs could be used to achieve the same lab

# Architecture
Topology
### Addressing
| Network | pfSense interface | Bridge | Subnet | Gateway | DHCP Range | 
| --- | --- | --- | --- | --- | --- |
| WAN | WAN(vtnet0) | vmbr1 | ISP-assigned public IP | ISP | client |
| LAN | LAN(vtnet1) | vmbr2 | 192.168.x.0/24 | 192.168.x.x/24 | .75 - .253 |
| VLAN 10 | VLAN10(vtnet2.10) | vmbr4 | 10.20.10.x/24 | 10.20.10.x | .100 - .199 |
| VLAN 20 | VLAN20(vtnet2.20) | vmbr4 | 10.20.20.x/24 | 10.20.20.x | .100 - .199 |
| WireGuard | WG (tun_wg0) | (tunnel) | 10.20.99.x/24 | 10.20.99.x | static per peer |

### Static Addresses (LAN .2 - .74 reserved)
| Host | IP | How | 
| --- | --- | --- |
| TP-Link(AP) | | 192.168.0.x | DHCP static mapping archer.home.arpa |
| Mesh Unit | 192.168.0.x | DHCP static mappings |
| Proxmox Host | 192.168.0.50 | Static on host, DNS host override proxmox.home.arpa |
| Wazuh | 192.168.0.x | Static on VMs, DNS host override wazuh.home.arpa |

### Zones and Policy
pfSense is an interface-based firewall: rules apply to traffic entering an interface, the first match wins, and new interfaces block everything by default. Zone-based firewalls instead write policy between groups of interfaces. This lab implements the zone model in pfSense with an interface group (LAB), whoe rules are evaluated before per-interface rules.

| Zone | Members |
| --- | --- |
| OUTSIDE | WAN |
| TRUSTED | LAN |
| LAB | VLAN10 + VLAN20(interface group LAB) |
| REMOTE | WG (WireGuard clients) |

Policy 

# Build Walkthrough
Each phase below is should be enough to reproduce the work shown in this lab
include picture here
1. Proxmox bridges: vmbr1 on NC360T port 1 (WAN), vmbr2 on port 2 (LAN), both with no host IP. vmbr0 keeps the host's management IP
2. pfSense VM NICs: net0 on vmbr1, net1 on vmbr2, model VirtIO.
3. Before connecting anything, set the LAN from the pfSense console(option 2)example shown next, feel free to change: 192.168.0.1/24, DHCP .75 - .253, keep the web GUI on HTTPS
4. Switch router to access point mode, or connect access point to `vmbr0` if you choose to use a physical one
5. Move the modem cable to NC360T port 1, then perform a power cycle for 60 seconds. This is so the modem releases the old router's MAC address. Reboot pfSense so the WAN requests a new lease
6. Connect NC360T port 2 to the TP-Link
7. Verify from the Proxmox host shell, which works even without Wi-Fi or DHCP

```bash
ping -c 4 192.168.x.x       # LAN path to pfSense
ping -c 4 8.8.8.8       # WAN routing
ping -c 4 google.com    # DNS
```
pfSense dashboard after cutover
more stuff to do afterwards
| Setting | Value | 
| --- | --- |
|Hostname/ domain | pfSense / home.arpa |
| WAN | DHCP from modem; block private networks and block bogons checked |
| Web GUI | HTTPS, LAN only; anti-lockout rule kept |
| Admin | Default passwords changed 
| Proxmox | pfSense VM start at boot = yes |

- Optional but helpful taking backups of the pfSense config and taking snapshots of the pfSense VM itself on Proxmox

more stuff to come back to after

- Static mappings (services > DHCP server > LAN): the AP and mesh units get reserved IPs
- DNS Resolver (Services > DNS Resolver):
- Register DHCP static mappings, so archer.home.arpa resolves automatically
- Host overrides for static hosts: proxmox.home.arpa, wazuh.home.arpa
- Network interfaces: All, so VLAN and VPN clients can query their gateway

DHCP static mappings DNS host overrides 
