

# 2-Tier-Collapsed-Core-Enterprise






## Table of Contents
- [Overview](#overview)
- [Technologies-implemented end-to-end](#technologies-implemented-end-to-end)
- [Topology](#topology)
- [Device Inventory](#device-inventory)
- [Vlan Design](#vlan-design)
- [Key Configuration](key-configuration)
- [PRTG Monitoring — Architecture Note](prtg-monitoring-architecture-note)
- [Verification Checklist](verification-checklist)
- [Troubleshooting Log](troubleshooting-log)
- [Screenshots](screenshots)
- [Skills Demonstrated](skills-demonstrated)
- [Tools and Platform](tools-and-platform)




## Overview
This is a realistic enterprise network simulation built in Cisco Modeling Labs (CML), modeled after real SMB deployments. This project demonstrates a 2-tier collapsed core architecture where the core and distribution layers are merged into a single Layer 3 switch: the dominant design pattern for small-to-medium enterprise environments under ~200 endpoints.
Every design decision in this lab has a business and technical justification — from why VTP runs in server/client mode instead of transparent, to why VLAN 20 is pruned from branch trunks, to why the MGMT VLAN is excluded from the NAT ACL. 








## Technologies implemented end-to-end:

	•	VLAN segmentation with business-function-based design (Corp, Voice, Guest, MGMT)
	•	Inter-VLAN routing via Layer 3 SVIs on a collapsed core switch
	•	Centralized DHCP with per-VLAN scopes, exclusions, and VoIP Option 150
	•	NAT/PAT overload for internet egress with RFC 1918 address translation
	•	802.1Q trunking with VLAN pruning across 3 access switches
	•	Rapid PVST+ with explicit root bridge assignment
	•	VTP server/client VLAN propagation
	•	SSH v2 hardened management access on all 5 devices
	•	SNMP v2c monitoring with PRTG Network Monitor
	•	Wireshark packet capture validation at a  Key Capture Point
	•	13 endpoint nodes generating real DHCP, ARP, ICMP, and application traffic






## Topology:

                    	 [ INTERNET ]
                             |
                        [ R1-EDGE ]
                      WAN · NAT/PAT
                             |
                    ┌────────────────┐
                    │   SW1-CORE     │  ← TIER 1: Collapsed Core
                    │  Layer 3 · L3  │    SVIs · DHCP · VTP Server
                    │  ip routing    │    STP Root · Trunk Agg.
                    └────────────────┘
                   /        |         \
           ┌──────┐   ┌──────────┐   ┌──────────┐
           │SW2-HQ│   │SW3-BR-A  │   │SW4-BR-B  │  ← TIER 2: Access
           │ L2   │   │  L2      │   │  L2      │    PortFast · BPDU Guard
           └──────┘   └──────────┘   └──────────┘    VTP Clients
           /    \        /    \          /    \
        Corp  Voice   Corp  Guest     Corp  Guest
        PCs   PCs     PCs    PCs      PCs    PCs



Why 2-Tier?

In a 3-tier design, there is are separate Core → Distribution → Access layers. A 2-tier collapsed core merges Core and Distribution into one layer — reducing hardware cost and complexity while maintaining full routing, VLAN, and STP functionality. 



## Device Inventory

| Hostname     | Tier | Role              | Platform       | Key Functions |
|-------------|------|------------------|---------------|--------------|
| R1-EDGE     | 1    | WAN Edge         | IOS 15.9      | NAT/PAT, default route, WAN handoff |
| SW1-CORE    | 1    | Collapsed Core   | IOSvL2        | SVIs, DHCP server, VTP server, STP root |
| SW2-HQ      | 2    | Access Switch    | IOSvL2        | Corp/Voice ports, PortFast, BPDU Guard |
| SW3-BRANCH-A| 2    | Access Switch    | IOSvL2        | Corp/Guest ports, PortFast |
| SW4-BRANCH-B| 2    | Access Switch    | IOSvL2        | Corp/Guest ports, Port Security |
| PRIG-SRV    | 2    | Monitoring Server| Ubuntu 22.04  | PRTG, SNMP v2c |
| Endpoints   | 2    | Clients          | CML Desktop   | DHCP clients, connectivity testing |



## VLAN Design

| VLAN Name | Subnet        | Gateway     | DHCP Range              | Notes |
|----------|--------------|------------|--------------------------|------|
| CORPORATE| 192.168.10.0/24 | 192.168.10.1 | 192.168.10.100 - 200 | All access switches |
| HQ ONLY  | 192.168.20.0/24 | 192.168.20.1 | 192.168.20.100 - 150 | HQ only |
| GUEST    | 192.168.30.0/24 | 192.168.30.1 | 192.168.30.100 - 199 | Branches only |
| MGMT     | 10.99.0.0/24    | 10.99.0.1    | Static only           | OOB management |



VLAN Pruning: VLAN 20 (Voice) is excluded from SW3 and SW4 trunk allowed lists — no phones at branch locations. Carrying only the required VLANs per trunk reduces broadcast domain scope.





## Key Configurations:

### NAT/PAT — R1-EDGE:
ip access-list standard NAT-INSIDE

 permit 192.168.10.0 0.0.0.255
 
 permit 192.168.20.0 0.0.0.255
 
 permit 192.168.30.0 0.0.0.255
 
ip nat inside source list NAT-INSIDE interface GigabitEthernet0/0 overload


### Inter-VLAN Routing — SW1-CORE:
ip routing

interface Vlan10

 ip address 192.168.10.1 255.255.255.0
 
 no shutdown
 
interface Vlan20

 ip address 192.168.20.1 255.255.255.0
 
 no shutdown
 
interface Vlan30

 ip address 192.168.30.1 255.255.255.0
 
 no shutdown

 
### DHCP — SW1-CORE (Centralized):

ip dhcp excluded-address 192.168.10.1 192.168.10.99

ip dhcp pool VLAN10-CORPORATE

 network 192.168.10.0 255.255.255.0
 
 default-router 192.168.10.1
 
 dns-server 8.8.8.8 8.8.4.4
 
 domain-name corp.local
 
 lease 1

 
### VTP & STP — SW1-CORE:

vtp domain CORP-LAB

vtp mode server

vtp version 2

spanning-tree mode rapid-pvst

spanning-tree vlan 10,20,30,99 root primary


### SNMP / PRTG Monitoring

SNMP v2c is fully configured on all 5 network devices. PRTG Network Monitor is installed on the Windows host and actively monitors R1-EDGE with live sensors. Internal switches (SW1–SW4) have SNMP configured and are verified working within the switches but are not directly polled by the external PRTG instance due to CML’s libvirt network isolation — see PRTG Monitoring — Architecture Note for full details.


SNMP configured on all 5 devices:

snmp-server community LAB-READ RO

snmp-server location CML-2Tier-Lab

snmp-server contact NOC-Admin

snmp-server host 192.168.17.1 version 2c LAB-READ

snmp-server enable traps snmp linkdown linkup coldstart

PRTG actively monitoring R1-EDGE

	•	Ping v2 — live latency (2ms avg) 
	•	DNS v2 — up 
	•	HTTP v2 — up 
	•	All 5 devices added to PRTG 2-Tier Enterprise group with correct SNMP v2c credentials
  
SNMP verified working internally on all switches:

	•	show snmp community confirms LAB-READ RO active on SW1–SW4
	•	show snmp confirms SNMP agent running on all devices
	•	Trap destination 192.168.17.1 (Windows host) configured on all devices


### Verification Checklist:

	•	SW1# show vlan brief — VLANs 10/20/30/99 active, VTP propagated to access switches
	•	SW1# show interfaces trunk — correct VLANs allowed, native VLAN 1
	•	SW1# show ip interface brief — all 4 SVIs up/up
	•	SW1# show spanning-tree vlan 10 — “This bridge is the root”
	•	SW1# show ip dhcp binding — Corp PC received 192.168.10.100+
	•	Ping 192.168.20.1 from Corp PC — inter-VLAN routing working
	•	Ping 8.8.8.8 from Corp PC — NAT/PAT working
	
### PRTG Monitoring — Architecture Note:

SNMP v2c is fully configured on all 5 network devices with community string LAB-READ. PRTG Network Monitor is installed on the Windows host and successfully monitors R1-EDGE with live sensors.

Why internal switches (SW1–SW4) are not directly polled by PRTG:

CML runs on top of VMware Workstation using libvirt for internal network virtualization. The lab nodes communicate over an internal libvirt bridge (virbr0, 192.168.255.0/24) that is isolated from the host OS network stack. Traffic between lab nodes on virbr0 is switched at the bridge level and never passes through the CML VM’s kernel — meaning it cannot be intercepted, routed, or forwarded to external hosts like the Windows PRTG server.




Extensive troubleshooting was performed including:

	•	Enabling IP forwarding on the CML VM (sysctl net.ipv4.ip_forward=1)
	•	Adding static routes on the CML VM for all lab subnets via R1-EDGE’s WAN IP
	•	Adding Windows host routes pointing to the CML VM as next hop
	•	Flushing nftables rulesets to eliminate firewall blocking
	•	Modifying LIBVIRT_FWI iptables/nftables chains
	•	Confirmed PRTG SNMP queries reaching bridge0 via tcpdump — but packets do not cross the libvirt boundary into virbr0

Root cause: libvirt’s internal bridge isolates virtual node traffic from the host network by design. SNMP queries from the Windows host reach the CML VM’s bridge0 interface but cannot be forwarded into the virbr0 segment where lab nodes reside without a dedicated in-band monitoring node inside the lab network.

Production equivalent: In a real enterprise environment, PRTG would run on a dedicated monitoring server connected directly to the management VLAN — not on an external host trying to reach through a hypervisor NAT layer. This project accurately reflects that architectural requirement.

SNMP configuration is complete and verified on all devices:


	•	All 5 devices respond to SNMP queries from within the lab network
	•	SNMP traps configured to send to 192.168.17.1 (Windows host)
	•	R1-EDGE fully monitored in PRTG with live ping, DNS, and HTTP sensors
	•	All devices added to PRTG 2-Tier Enterprise group with correct SNMP v2c credentials


### Troubleshooting Log:

This section documents real issues encountered during the project build, the methodology used to diagnose them, and the root cause analysis. 

#### Issue 1: MGMT SVI Unreachable — Native VLAN Mismatch
Symptom: SW1-CORE could not ping SW2-HQ, SW3-BRANCH-A, or SW4-BRANCH-B via their MGMT IPs (10.99.0.11/12/13). Trying to SSH into access switches failed. PRTG could not poll any switch. All trunk links showed up/up and VLAN 99 was in STP forwarding state on all trunks.
Troubleshooting methodology:

	1.	Confirmed physical links were up — show interfaces trunk showed all trunks up/up
	2.	Confirmed VLAN 99 was active and in forwarding state on all trunks
	3.	Confirmed Vlan99 SVIs were up on access switches — show interfaces Vlan99 showed up/up with correct IPs
	4.	Confirmed ARP entries existed on SW1-CORE for all MGMT IPs
	5.	Captured interface counters — confirmed frames were leaving SW1-CORE Gi0/1 but Vlan99 input counter on SW2-HQ was NOT incrementing, despite frames arriving on Gi0/0
	6.	Identified that frames were arriving at the physical port but not being processed by the SVI
Root cause: Native VLAN was set to 99 on all trunks. The native VLAN is transmitted untagged on a trunk. A Switch Virtual Interface (SVI) processes only tagged frames for its VLAN — it does not process untagged frames. With VLAN 99 as native, all MGMT traffic was being sent untagged and the Vlan99 SVI was silently discarding it. The frames arrived at the physical port but never reached the SVI.

Fix: Changed native VLAN from 99 to 1 on all trunk interfaces on all switches:

interface GigabitEthernet0/x (whichever interface was applied to on each switch)
 switchport trunk native vlan 1

With native VLAN set to 1, VLAN 99 traffic is now sent tagged — and the Vlan99 SVI correctly processes it.
Lesson learned: SVIs require their VLAN traffic to arrive tagged. Never use your management VLAN as the native VLAN on trunk ports. Native VLAN 1 is the industry standard default and avoids this class of issue entirely.

#### Issue 2: SSH Locked Out — transport input none
Symptom: After initial configuration, SSH connections to several switches were refused with “Connection refused” even though show ip ssh confirmed SSH was enabled and RSA keys were generated.
Troubleshooting methodology:

	1.	Confirmed SSH was enabled — show ip ssh returned SSH Enabled version 2.0
	2.	Confirmed RSA keys existed — show crypto key mypubkey rsa returned valid keys
	3.	Console access still worked — isolated the issue to VTY lines
	4.	Ran show line vty 0 4 — discovered transport input none was configured
Root cause: During initial VTY configuration, transport input none was accidentally applied to the VTY lines. This command disables ALL remote access methods — SSH, Telnet, everything. The switch had a working SSH service but the VTY lines refused all incoming connections.

Fix:

line vty 0 4
 transport input ssh

Lesson learned: Always verify VTY line configuration after initial setup with show run | include transport. The difference between transport input none and transport input ssh is a single word — but one completely locks you out of remote management. 
#### Issue 3: IOSvL2 Command Limitations

Symptom: Several SNMP commands that worked on R1-EDGE (IOSv) failed with invalid input detected on all four switches (IOSvL2).
Commands that failed on IOSvL2:

	•	snmp-server contact noc@corp.local — the @ symbol caused a parse error
	•	snmp-server enable traps config — command not supported on IOSvL2
Troubleshooting methodology:

	1.	Confirmed commands were  correct by testing on R1-EDGE (IOSv) — they worked
	2.	Identified that all failing devices were running IOSvL2, not IOSv
	3.	Researched IOSvL2 command set differences — confirmed these are known platform limitations
Root cause: IOSvL2 is a stripped-down switch image used in CML for Layer 2/3 simulation. It does not support the full IOSv command set. Specifically: the SNMP contact field parser in IOSvL2 rejects special characters including @, and the config trap type is not implemented in the IOSvL2 SNMP agent.

Fix:

Use _  instead of @ in contact field on IOSvL2
snmp-server contact NOC-Admin

Removed  unsupported trap type on IOSvL2
 snmp-server enable traps config  ← removed from switch configs
added snmp-server enable traps snmp linkdown linkup coldstart

Lesson learned:

 The same Cisco configuration does not always work identically across different IOS platform variants. 

 
#### Issue 4: Missing ip default-gateway on Access Switches
Symptom: After confirming VLAN 99 SVIs were up on all access switches, SSH and SNMP still failed from external hosts. The switches could ping their own SVI IP but could not reach any device outside the 10.99.0.0/24 subnet.
Troubleshooting methodology:

	1.	Confirmed Vlan99 SVI was up/up on SW2-HQ — show interfaces Vlan99 
	2.	SW2-HQ could ping 10.99.0.1 (SW1-CORE MGMT SVI) 
	3.	SW2-HQ could NOT ping 192.168.17.1 (Windows host) 
	4.	Ran show run | include default-gateway — returned no output
	5.	Identified missing default gateway
Root cause: Access switches run with ip routing disabled — they are Layer 2 only and cannot route packets themselves. Without ip default-gateway, a switch has no way to forward management traffic destined for subnets outside its directly connected VLAN. It could reach 10.99.0.1 (same subnet) but had no path for anything else.

Fix:

Add the ip default-gateway 10.99.0.1 command to all L2 switches 

### Issue 5: STP Topology Changes Causing MGMT Instability
Symptom: After fixing native VLAN and default-gateway issues, MGMT connectivity to access switches became intermittent. Pings to 10.99.0.11 would succeed 4/5 then fail, then succeed again in an unpredictable pattern.


Troubleshooting methodology:

	1.	Confirmed routes and SVIs were correct — all static
	2.	Ran show spanning-tree detail | include topology on SW1-CORE — topology change counter was incrementing continuously (4→2→5→4)
	3.	Ran show spanning-tree vlan 99 detail — identified topology changes originating from Gi0/1 (trunk to SW2-HQ)
	4.	Identified that Corp and Voice PC nodes were powered off — their ports were continuously flapping
	5.	Confirmed PortFast was not globally enabled on access switches

Root cause: When a port transitions from down to up (or up to down), STP treats this as a topology change event and triggers network-wide reconvergence. During reconvergence, MAC address tables are flushed and ports briefly re-enter learning state — causing packet loss. With multiple PC nodes powered off, their ports were continuously flapping, triggering repeated topology changes that caused the intermittent MGMT packet loss.

Fix:
Apply these commands on all access switches
spanning-tree portfast default
spanning-tree portfast bpduguard default

PortFast causes access ports to skip Listening and Learning states and go directly to Forwarding. PortFast ports do not generate topology change notifications when they transition — preventing end device port flaps from triggering network-wide STP reconvergence.

Lesson learned: STP topology changes can be one of the most common and subtle causes of intermittent network issues in SMB environments. I learned that when I see intermittent packet loss on a stable network - check show spanning-tree detail | include topology immediately. PortFast should always be enabled globally on access switches — it is safe on ports connected to end devices and prevents unnecessary reconvergence events.


### Issue 6: PRTG Cannot Poll Internal Switches — libvirt Isolation
See PRTG Monitoring — Architecture Note for full details.

Summary: After extensive troubleshooting including IP forwarding, static routing, nftables/iptables rule modification, and tcpdump packet capture analysis, the root cause was identified as a fundamental CML platform constraint: libvirt’s internal bridge (virbr0) switches traffic between CML lab nodes at the kernel bypass level, making it impossible to route SNMP packets from an external Windows host into the network without a dedicated in-band monitoring node on the management VLAN.

Lesson learned: Monitoring servers belong inside the network on a dedicated management VLAN — not outside trying to reach through a hypervisor NAT boundary. 

 All screenshots were captured while the project was worked on live. Each image is annotated below with what it proves and which verification step it corresponds to.

###Screenshots

### 1. CML Topology Canvas
File: <img width="1919" height="1026" alt="2 Tier Collapsed Core Topology" src="https://github.com/user-attachments/assets/c408fb3d-b167-4f84-9680-e108711dd01c" />


What it shows: Full 2-tier collapsed core topology in CML Workbench — R1-EDGE at the top connected to the Internet (ext-conn-0 NAT connector), SW1-CORE as the collapsed core aggregating all three access switches, 13 endpoint Desktop nodes distributed across Corp, Voice, and Guest VLANs.
What it proves: Complete topology was built and all nodes are in STARTED/BOOTED state.

### 2. VLAN Database — VTP Propagation
File: <img width="1919" height="801" alt="image" src="https://github.com/user-attachments/assets/a87653d7-df8a-4aa2-9b50-7b6d5ffd2cc3" />


Command: show vlan brief on SW2-HQ
What it shows: VLANs 10 (CORPORATE), 20 (VOICE), 30 (GUEST), and 99 (MGMT) all active on SW2-HQ with correct port assignments.
What it proves: VTP propagation is working — VLANs created on SW1-CORE (VTP server) were automatically distributed to SW2-HQ (VTP client) without manual configuration on the access switch.

### 3. Trunk Links — VLAN Pruning
File: <img width="1914" height="858" alt="2 Tier Trunking verification" src="https://github.com/user-attachments/assets/e8921de9-6e06-48ff-920f-d4e27f49f214" />


Command: show interfaces trunk on SW1-CORE
What it shows: All three trunk links (Gi0/1→SW2, Gi0/2→SW3, Gi0/3→SW4) with allowed VLANs, native VLAN 1, and VLANs in STP forwarding state.
What it proves: 802.1Q trunking is operational and VLAN pruning is correctly applied — VLAN 20 (Voice) is absent from SW3 and SW4 trunk allowed lists since there are no phones at branch locations.

### 4. STP Root Bridge Verification
File: <img width="1915" height="860" alt="2 Tier SW1 Core is root bridge verification check" src="https://github.com/user-attachments/assets/213b3fb8-28f9-4954-9454-99dd5c399c12" />


Command: show spanning-tree vlan 10 on SW1-CORE
What it shows: Output displaying This bridge is the root with SW1-CORE holding the lowest bridge priority (4096) for VLAN 10.
What it proves: SW1-CORE is correctly elected as the Spanning Tree root bridge for all VLANs, ensuring optimal traffic flow and preventing suboptimal forwarding paths through access switches.

### 5. DHCP Bindings — Multi-VLAN Leases
File: <img width="1917" height="910" alt="2 Tier Sh IP DHCP BINDING verification check" src="https://github.com/user-attachments/assets/42c0001f-b2e7-4984-bfc4-8450d5d0c369" />


Command: show ip dhcp binding on SW1-CORE
What it shows: Active DHCP leases across VLAN 10 (9 Corp workstations at 192.168.10.x), VLAN 20 (0 Voice — phones offline), and VLAN 30 (4 Guest devices at 192.168.30.x).
What it proves: Centralized DHCP on SW1-CORE is correctly serving addresses across all three user VLANs simultaneously with correct scope assignments per VLAN.

### 6. Inter-VLAN Routing — End-to-End Ping
File: <img width="1908" height="785" alt="Screenshot 2026-04-18 182237" src="https://github.com/user-attachments/assets/335c1f2c-6cd2-4e21-8f17-65bf5b0e681e" />


Command: ping 192.168.20.1 from CORP-PC-1 (VLAN 10)
What it shows: Successful ICMP echo replies from SW1-CORE’s VLAN 20 SVI (192.168.20.1) originating from a Corp workstation on VLAN 10.
What it proves: Inter-VLAN routing via SVIs is fully operational — traffic crosses VLAN boundaries through SW1-CORE’s Layer 3 routing engine without any external router.

### 7. NAT/PAT — Internet Connectivity
File: <img width="1916" height="838" alt="2 Tier Corp-PC proof that PCs have access to Internet" src="https://github.com/user-attachments/assets/370cfdbb-effe-4e95-93b3-113b3d1ca135" />


Command: ping 8.8.8.8 from CORP-PC-1
What it shows: Successful ICMP echo replies from Google’s DNS server (8.8.8.8) originating from a Corp workstation with private IP 192.168.10.x.
What it proves: NAT/PAT overload is correctly translating private RFC 1918 addresses to R1-EDGE’s public WAN IP (192.168.255.112) for internet egress.

### 8. NAT Translation Table
File: <img width="1914" height="877" alt="2 Tier NAT translations verification " src="https://github.com/user-attachments/assets/dc704ee0-abca-4b11-8b6a-ff74c3e01091" />



Command: show ip nat translations on R1-EDGE
What it shows: Active NAT translation entries showing the mapping between inside local IPs (192.168.10.x) and the inside global IP (192.168.255.112) with unique port assignments per session.
What it proves: PAT overload is functioning correctly — multiple internal hosts are sharing one public IP address distinguished by unique TCP/UDP port numbers.

### 9. SSH Verification
File: <img width="1913" height="908" alt="Screenshot 2026-04-13 181530" src="https://github.com/user-attachments/assets/54ffd3fc-bdbb-4d26-81a4-a9bebdd596d6" />


Command: show ip ssh on R1-EDGE
What it shows: Output confirming SSH Enabled - version 2.0 with RSA key size 2048 bits and authentication timeout/retries configured.
What it proves: SSH v2 is enabled on all network devices providing encrypted management access. Telnet is disabled (transport input ssh only) preventing cleartext credential exposure.

### 10. SNMP Configuration
File: <img width="1906" height="822" alt="2 tier sh snmp community" src="https://github.com/user-attachments/assets/57b8deec-235e-4d44-92cb-07fc927d5326" />


Command: show snmp community on SW1-CORE
What it shows: Community string LAB-READ configured as Read-Only (RO) with nonvolatile storage type and active status.
What it proves: SNMP v2c is correctly configured on all devices with read-only community access for PRTG polling. Write access is intentionally not configured — monitoring systems should never have write access to production devices.

### 11. PRTG — R1-EDGE Live Monitoring
File: <img width="1919" height="1079" alt="PRTG monitoring tool" src="https://github.com/user-attachments/assets/c6d2169a-7659-44fc-8236-af2478515d23" />


What it shows: PRTG dashboard displaying R1-EDGE device with green sensors — Ping v2 (2ms response time), DNS v2 (Up), HTTP v2 (Up) all showing OK status with live response time graphs.
What it proves: PRTG Network Monitor is successfully monitoring R1-EDGE via the management network with live SNMP and ping sensors. Real-time response time data confirms continuous polling is operational.

### 12. Wireshark — NAT Validation Capture
File: <img width="1913" height="503" alt="2 Tier Wireshark Capture" src="https://github.com/user-attachments/assets/50ea7ef0-3581-4c32-8938-e17e7588466f" />

Capture point: R1-EDGE Gi0/0 (WAN interface toward Internet)
What it shows: Wireshark capture on R1-EDGE’s WAN interface showing ICMP echo requests with source IP 192.168.255.112 (R1-EDGE’s public WAN IP) destined for 8.8.8.8 — no RFC 1918 private addresses visible on the WAN side.
What it proves: NAT translation is confirmed at the packet level — private IP addresses are completely hidden from the public internet and only the translated public IP appears on the WAN segment.

### Skills Demonstrated
	- Layer 2: 802.1Q trunking, VTP server/client, Rapid PVST+, PortFast, BPDU Guard, port security
	- Layer 3: Inter-VLAN routing via SVIs, static routing, ip routing on L3 switch
	- Services: DHCP scopes with exclusions, Option 150 (VoIP TFTP), NAT/PAT overload
	- Monitoring: SNMP v2c, PRTG sensor configuration, alert threshold tuning
	- Design: 2-tier collapsed core architecture, VLAN pruning, OOB management plane separation 


### Tools & Platform
	- Cisco Modeling Labs (CML) 2.x — simulation platform
	- IOSv 15.9 — router image (R1-EDGE)
	- IOSvL2 — Layer 2/3 switch image (SW1–SW4)
	- Ubuntu 22.04 — PRTG monitoring server
	- CML Desktop (Alpine Linux) — endpoint nodes
	- Wireshark — packet capture and analysis
	- PRTG Network Monitor — real-time monitoring






