# 01 — Network Architecture

## Overview

The network is built inside VirtualBox using OPNsense as the central 
firewall and router. Instead of VLAN tagging on a single interface, 
each network segment is connected to OPNsense via a **dedicated 
virtual adapter** — giving true physical-level isolation between zones 
within the hypervisor.

This design mirrors how small-to-mid enterprise environments are often 
structured: separate broadcast domains per department, a single 
perimeter firewall handling all inter-zone routing, and a centralized 
server segment hosting shared services.

---

## Network Segments

| Segment | OPNsense Interface | Adapter | Subnet | Gateway |
|---------|-------------------|---------|--------|---------|
| WAN | em0 | NAT | 10.0.2.15 | VirtualBox NAT |
| MGMT | em1 | Host-Only | 192.168.56.10 | 192.168.56.1 |
| HR_NET | em2 (OPT1) | Internal | 10.10.10.0/24 | 10.10.10.1 |
| IT_NET | em3 (OPT2) | Internal | 10.10.20.0/24 | 10.10.20.1 |
| SERVER_NET | em4 (OPT3) | Internal | 10.10.30.0/24 | 10.10.30.1 |
| GUEST_NET | em5 (OPT4) | Internal | 10.10.40.0/24 | 10.10.40.1 |

### WAN
Provides internet access to internal segments via NAT through 
VirtualBox. OPNsense handles outbound routing for all zones.

### MGMT
Host-only adapter used exclusively to access the OPNsense web GUI 
from the host machine. Not routed to internal segments.

### HR_NET / IT_NET
Department endpoint networks. Machines in these segments are 
domain-joined and receive IPs from DC1 via DHCP relay.

### SERVER_NET
Hosts all internal servers (DC1, DC2, FILE1). Static IPs assigned 
manually — no DHCP on this segment. Servers here serve the rest 
of the infrastructure.

### GUEST_NET
Isolated network for untrusted devices. Blocked from reaching any 
internal segment. Internet access only. DNS points to the firewall 
itself — not the internal DC — to prevent internal name resolution.

---

## IP Address Map

| Host | Segment | IP |
|------|---------|-----|
| OPNsense (WAN) | WAN | 10.0.2.15 |
| OPNsense (MGMT) | MGMT | 192.168.56.10 |
| OPNsense (HR gateway) | HR_NET | 10.10.10.1 |
| OPNsense (IT gateway) | IT_NET | 10.10.20.1 |
| OPNsense (SERVER gateway) | SERVER_NET | 10.10.30.1 |
| OPNsense (GUEST gateway) | GUEST_NET | 10.10.40.1 |
| DC1 | SERVER_NET | 10.10.30.10 |
| DC2 | SERVER_NET | 10.10.30.11 |
| FILE1 | SERVER_NET | 10.10.30.20 |

---

## Design Decisions

### Why dedicated adapters instead of VLANs?
In a production environment, VLANs on a trunk port would be the 
standard approach. In this lab, dedicated VirtualBox internal 
networks achieve the same logical isolation while keeping the 
focus on firewall rules, routing, and server configuration — 
not on switch/trunk configuration. The segmentation logic is 
identical either way.

### Why put all servers in one segment?
SERVER_NET consolidates DC1, DC2, and FILE1 in one subnet, 
which is common in smaller enterprise environments. 
Access to this segment is controlled at the firewall level — 
clients in HR_NET and IT_NET can only reach specific 
services on specific servers, not the segment as a whole.  
*(In Phase 2, this will be enforced with strict firewall aliases 
and allowlists rather than the current broad rules.)*

### Why is GUEST DNS pointed at the firewall?
Pointing GUEST clients to the OPNsense gateway (10.10.40.1) 
as their DNS server — instead of DC1 — means guests can resolve 
public internet names but have no visibility into internal 
hostnames like `FILE1.company.local` or `DC1.company.local`. 
This is a simple but effective isolation layer.

---

## Topology Diagram

*(See `/diagrams/network-topology.png`)*
```
                        [ INTERNET ]
                              |
                         [ OPNsense ]
                         Firewall/Router
                              |
         ┌────────┬──────────┬────────┬────────┐
         │        │          │        │        │
      HR_NET   IT_NET   SERVER_NET  GUEST_NET  MGMT
      /10      /20        /30        /40     (host-only)
                        ┌───┴────┐
                     DC1/DC2   FILE1
```

---

## Security Notes (Phase 1 Gaps)

- Inter-zone routing is active with **no default-deny policy** — 
  a compromised endpoint in HR_NET can attempt lateral movement 
  toward SERVER_NET
- No IDS/IPS sitting on segment boundaries
- MGMT interface is not monitored or access-controlled beyond 
  being host-only

*These gaps are addressed in Phase 2.*
