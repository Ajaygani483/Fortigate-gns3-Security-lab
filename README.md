# FortiGate GNS3 Security Labs

Hands-on FortiGate labs I built in GNS3 while working through NSE4 material. Each folder is a separate lab — topology, what I configured, and how I checked it actually worked.

I'm keeping this honest rather than polished: some labs below are fully tested with real traffic, others are configuration walkthroughs I've done but haven't fully verified yet. Each lab's README says which is which.

## Labs

| # | Lab | What it covers |
|---|---|---|
| 01 | [High Availability](./01-High-Availability-HA/) | Active-Passive FortiGate clustering, failover |
| 02 | [MAC Address Filtering](./02-Mac-Address-Filtering/) | Restricting a policy to one device by MAC |
| 03 | [Central NAT / DMZ](./03-Central-NAT-DMZ/) | Publishing a DMZ server with a static VIP |
| 04 | [DHCP Server + Reservation](./04-DHCP-Server-Reservations/) | FortiGate as DHCP server, one fixed lease |
| 05 | [DHCP Relay](./05-DHCP-Relay-Agent/) | Forwarding DHCP across a routed boundary |
| 06 | [OSPF Routing](./06-OSPF-Routing-LAN-DMZ/) | Dynamic routing through the FortiGate |
| 07 | [Site-to-Site IPsec VPN](./07-Site-to-Site-IPsec-VPN/) | Encrypted tunnel between two sites |

## Background

I'm a NOC/Network Engineer (CCNA, working through CCNP ENCOR) doing daily FortiGate health monitoring at work. These labs are me going past "monitor and check status" into actually configuring the features I look after — partly for NSE4 prep, partly because I wanted to actually build the things I've only watched other engineers configure.

## Tools
GNS3, FortiGate VM (FortiOS 7.x), Cisco IOS routers/switches for the surrounding topology.

# Lab 01 – Active-Passive FortiGate High Availability (HA)

## Objective
Set up two FortiGate units in an Active-Passive HA cluster (FGCP), so that if the primary unit fails, the secondary takes over without manual intervention.

## Topology
<img width="1519" height="640" alt="Active-Passive HA" src="https://github.com/user-attachments/assets/9e0dd85f-1de4-4cd8-b991-76d2d842e552" />

## Configuration

**Primary unit** — System > High Availability:
- Mode: Active-Passive
- Device Priority: 200 (higher priority wins the primary role)
- Group Name: Corporate-HA-Cluster
- Password: (set a shared cluster password)
- Heartbeat Interface: port4, priority 50

**Secondary unit** — same settings, except:
- Device Priority: 100

**Override** (CLI on the primary unit):
config system ha
set override enable
end

# Lab 02 – Layer 2 Access Control via MAC Address Objects

## Objective
Restrict a firewall policy to a specific device by MAC address, instead of allowing an entire subnet.

## Topology
<img width="1546" height="556" alt="Adress for policy for specific user" src="https://github.com/user-attachments/assets/7a657f33-cda3-4d4d-b12d-9371b348a7d2" />

## Configuration

**Step 1 — Create a MAC address object**
Policy & Objects > Addresses > Create New > Address:
- Name: LAN-PC1-MAC
- Type: Device (MAC Address)
- MAC address: 00:50:79:66:68:00
- Interface: port2

**Step 2 — Use it in a firewall policy**
Policy & Objects > Firewall Policy > Create New:
- Incoming Interface: port2 (LAN)
- Outgoing Interface: port3 (WAN)
- Source: LAN-PC1-MAC
- Destination: all
- Action: ACCEPT
- NAT: Enabled

## Verification
! Ping/browse from the matching PC — should work
! Ping/browse from a different PC on the same LAN with a different MAC — should be blocked
FortiGate # diagnose firewall iprope list

# Lab 03 – Central NAT & Static VIP for a DMZ Service

## Objective
Publish an internal DMZ server using a static Virtual IP (VIP), and manage outbound source NAT centrally instead of per-policy.

## Topology
<img width="1556" height="630" alt="CNAT lab" src="https://github.com/user-attachments/assets/4ecb2e6e-97ca-4be2-9ca9-8d8c482e4fcd" />

## Configuration

**Step 1 — Enable Central NAT**
System > Feature Visibility > enable Central NAT > Apply.

**Step 2 — Create the inbound VIP**
Policy & Objects > Virtual IPs > Create New > Virtual IP:
- Name: DMZ-Public-VIP
- Interface: port3 (WAN)
- External IP Address: 192.168.1.10
- Mapped IP Address: 20.0.0.2 (internal DMZ server)

Check: confirm 192.168.1.10 isn't already assigned to the WAN interface itself elsewhere in the topology.

**Step 3 — Central SNAT for outbound traffic**
Policy & Objects > Central SNAT > Create New:
- Incoming Interface: port2
- Outgoing Interface: port3
- Source / Destination: all / all

## Verification
! From outside: connect to 192.168.1.10 and confirm it reaches 20.0.0.2
! Packet capture on port3 showing translated source/destination

# Lab 04 – FortiGate DHCP Server with Static MAC Reservation

## Objective
Run a DHCP server directly on a FortiGate LAN interface, and reserve a fixed IP for one specific client by MAC address.

## Topology
<img width="843" height="618" alt="DHCP LAB" src="https://github.com/user-attachments/assets/0a1d7d78-cbc2-444c-9c7b-09b527ba6f3f" />

## Configuration

**Step 1 — Interface + DHCP pool**
Network > Interfaces > port2 > Edit:
- Addressing Mode: Manual
- IP/Netmask: 172.16.1.10/255.255.255.0
- DHCP Server: Enabled
- Address Range: 172.16.1.20 – 172.16.1.100

**Step 2 — Static MAC reservation**
Same interface, DHCP server Advanced settings > MAC Reservation > Create New:
- MAC Address: 00:50:79:66:68:01
- IP Address: 172.16.1.11

## Verification
! Client with that MAC gets exactly 172.16.1.11 on renew
! Network > DHCP Monitor showing the leased/reserved client

# Lab 05 – DHCP Relay Across a Routed Boundary

## Objective
Forward DHCP requests from a client on one subnet to a centralized DHCP server on a different subnet, via the FortiGate acting as a relay agent.

## Topology
<img width="1371" height="657" alt="DHCP Relay LAB" src="https://github.com/user-attachments/assets/688326db-203b-4594-8ca7-b9ae6075410b" />

## Configuration
Network > Interfaces > port2 (client-facing interface) > Edit:
- DHCP Server mode: switch from Server to Relay
- DHCP Server IP: 192.168.1.1 (the actual central DHCP server)

## Verification
! Client on port2's subnet receives a lease from the remote DHCP server
! Packet capture on port2 showing DHCP discover/offer being relayed

# Lab 06 – OSPF Routing Through a FortiGate

## Objective
Enable OSPF on the FortiGate so LAN and WAN-side subnets are learned dynamically rather than via static routes.

## Topology
<img width="1557" height="640" alt="OSPF" src="https://github.com/user-attachments/assets/395a8630-9e35-41b8-a21b-bab759ab6e1e" />

## Configuration
Network > Feature Visibility > enable Advanced Routing > Apply.

Network > OSPF:
- Router ID: 1.1.1.1
- Area: 0.0.0.0
- Networks advertised into area 0: 172.16.10.0/24, 192.168.1.0/24

## Verification
FortiGate # get router info ospf neighbor
FortiGate # get router info routing-table ospf

# Lab 07 – Site-to-Site IPsec VPN (Wizard-Based)

## Objective
Build an encrypted tunnel between two FortiGate sites, so traffic between each site's LAN subnet is encrypted across the WAN.

## Topology
<img width="1288" height="412" alt="Site to Site VPN" src="https://github.com/user-attachments/assets/d4c1d7f8-1775-46a1-a30b-ad4477d4e1b9" />

## Configuration
VPN > IPsec Wizard:
- Name: HQ-to-Branch-Tunnel
- Template: Site to Site
- Remote Device Type: FortiGate
- Remote Gateway IP: 192.168.229.135
- Outgoing Interface: port1 (WAN)
- Pre-shared Key: (set a strong key)
- Local Interface: port2
- Remote Subnet: (the branch site's actual LAN subnet)

Check: confirm the subnets used here actually match your real topology rather than wizard defaults.

## Verification
FortiGate # diagnose vpn tunnel list
! Ping/traceroute from a LAN host at Site A to a LAN host at Site B, through the tunnel

## Knowledge & Concepts (FortiGate NSE4 Coursework)

The list below covers what the NSE4 course covers topic by topic. Where I've built and tested a hands-on lab for something, I've linked it. Everything else here is conceptual understanding from coursework and daily firewall monitoring at work — not yet verified in a lab, and labeled that way on purpose.

### Security Policies
FortiGate policies match traffic by a 5-tuple (incoming/outgoing interface, source, destination, service) and apply an action — accept, deny, or IPsec. Policies are evaluated top-down, first match wins, and every ruleset ends in an implicit deny. I understand policy ordering, object-based source/destination (address objects, address groups, MAC-based objects), and how NAT and security profiles attach at the policy level rather than being separate standalone rules. Hands-on: covered in [MAC Address Filtering](./02-Mac-Address-Filtering/) and every other lab, since every lab needs a policy to actually pass traffic.

### Administration & Day-to-Day Management
This covers the operational side — admin profiles and access control, firmware upgrade procedure, configuration backup/restore, system time and DNS settings, and the difference between GUI and CLI management. I do this daily at work (device uptime, HA status, port status checks across shifts), so this is genuinely hands-on from the job, just not something I've built a dedicated GNS3 lab around.

### Deployment Modes
FortiGate can run as a traditional Layer 3 NAT/Route gateway (the mode all my labs use), or in Transparent mode, where it acts as a Layer 2 bridge with no routing — useful for dropping a firewall into an existing network without renumbering anything. There's also virtual-wire-pair mode for inline inspection without even switching. I understand the trade-offs (Transparent mode is easier to insert but loses some routing-based features) but all my labs so far use NAT/Route mode — Transparent mode is something I want to build a lab for next.

### Routing
Covers static routes, policy-based routing, and dynamic routing (OSPF, BGP) on a FortiGate, plus how routes interact with SD-WAN rules if configured. Hands-on: [OSPF Routing](./06-OSPF-Routing-LAN-DMZ/).

### NAT Technologies
FortiGate supports policy-based NAT (NAT settings live inside each firewall policy) and Central NAT (translation rules managed separately from policies, better for scale). SNAT covers outbound translation; DNAT/VIPs cover publishing internal services inbound. Hands-on: [Central NAT / DMZ](./03-Central-NAT-DMZ/).

### SSL Decryption & SSL Inspection
FortiGate can inspect encrypted traffic two ways: flow-based (inline, lower overhead, some detection limits) or proxy-based (full decrypt/re-encrypt, deeper inspection, more resource-intensive). Full SSL inspection requires FortiGate's CA certificate to be trusted on client devices, since it's actively terminating and re-establishing the TLS session — without that, users get certificate warnings on every HTTPS site. This breaks certificate-pinned apps unless specifically exempted. I understand this conceptually from coursework and daily monitoring context, but haven't configured and tested SSL inspection in a lab yet — it's next on my list.

### VPN Technologies
Covers IPsec (site-to-site and remote access), SSL VPN (web mode and tunnel mode), and the phase 1 (IKE, authentication + key exchange) / phase 2 (IPsec SA, actual encrypted tunnel) structure underneath every IPsec connection. Hands-on: [Site-to-Site IPsec VPN](./07-Site-to-Site-IPsec-VPN/).

### Security Profiles (Antivirus, Web Filter, DNS Filter, DoS, IPS)
These attach to a firewall policy and inspect traffic already permitted through it:
- **Antivirus** — scans file transfers against signature databases, with optional sandbox/heuristic analysis for unknown files
- **Web Filter** — blocks or allows by category (gambling, malware-hosting, social media, etc.), either by local database or FortiGuard cloud lookup
- **DNS Filter** — blocks lookups to known-bad domains before a connection is even attempted, which stops malware callbacks earlier than web filtering alone
- **DoS Policy** — interface-level thresholds that detect and drop flood-style traffic (SYN flood, ICMP flood, port scans) based on packets-per-second rather than content matching
- **IPS** — matches traffic against known exploit/attack signatures, independent of the port or protocol carrying it

I do daily health checks confirming these profiles are attached and their signature databases are current on production FortiGates at work, but haven't built a lab specifically testing what each one catches in practice — that's a good next lab.

### Troubleshooting
Covers reading FortiGate logs (traffic log, event log, security profile logs), using `diagnose sniffer packet` for live packet capture on the CLI, session table inspection (`diagnose sys session list`), and narrowing down whether a problem is routing, policy, NAT, or profile-related by testing each layer in order. This is the skill I use most at work day-to-day, just applied to monitoring/escalation rather than initial configuration.

### User Authentication
FortiGate can tie firewall policies to identity instead of just IP/subnet, checking against a local user database, LDAP/Active Directory, RADIUS, or picking up existing AD logon sessions transparently via FSSO (so a user doesn't have to log in twice). This is what makes policies like "only the Finance group can reach the Finance server" possible without hardcoding IPs. Conceptual knowledge from coursework — haven't configured directory integration in a lab yet.

### Traffic Analysis & Logging
FortiGate logs can go to local disk, FortiAnalyzer, or a syslog server, and every policy has a log setting (log all sessions, or only security events). I use FortiGate's traffic logs and Wireshark packet captures at work to trace connectivity issues, and understand log severity levels and how to correlate a traffic log entry back to the specific policy that handled it.

### High Availability & Redundancy
Covers FGCP active-passive and active-active clustering, heartbeat interfaces, session synchronization, and failover behavior. Hands-on: [High Availability](./01-High-Availability-HA/).

**What's knowledge-only right now, not yet lab-tested:** Transparent/virtual-wire deployment modes, SSL/SSH deep inspection, security profile behavior under real traffic, and directory-based user authentication. These are the natural next labs as I keep working through NSE4.
