# 04 — DHCP Infrastructure

## Overview

Most basic labs run DHCP directly on the firewall — one server, 
one segment, no complexity. This infrastructure takes a more 
realistic enterprise approach: **a centralized DHCP server on DC1 
serving multiple network segments through OPNsense acting as a 
DHCP relay.**

This matters because in a real company, you want IP address 
management centralized — one place to see all leases, all scopes, 
all reservations. Not a separate DHCP service running per interface 
on the firewall.

---

## Architecture
```
[ HR-PC / IT-PC / GUEST ]
         |
    (broadcast DHCP Discover)
         |
    [ OPNsense ]  ← intercepts broadcast, forwards as unicast
         |
    [ DC1 - 10.10.30.10 ]  ← assigns IP, sends offer back
         |
    [ OPNsense ]  ← relays offer back to client
         |
[ Client receives IP ]
```

The key detail here: DHCP clients send a **broadcast** — 
they don't know where the server is. Broadcasts don't cross 
router boundaries. The relay solves this by having OPNsense 
intercept the broadcast on each client-facing interface and 
forward it as a **unicast packet** directly to DC1. 
DC1 responds to OPNsense, which relays it back to the client.
Without the relay, clients in HR_NET or IT_NET would never 
reach a DHCP server sitting in SERVER_NET.

---

## OPNsense Configuration

### Step 1 — Disable built-in DHCP server

OPNsense has its own DHCP server per interface. 
This must be disabled on HR/IT/GUEST so it doesn't 
conflict with DC1:

`Services → DHCPv4 → HR / IT / GUEST → uncheck Enable → Save`

### Step 2 — Enable DHCP Relay

`Services → DHCRelay → Add destination: DC1-DHCP → 10.10.30.10`

Relay enabled on interfaces: **OPT1 (HR), OPT2 (IT), OPT4 (GUEST)**

---

## Firewall Rules for Relay

This is where most people get caught — enabling the relay 
service is not enough. The firewall must explicitly allow 
the two-hop traffic flow, because **OPNsense applies its 
own rules to its own forwarded traffic.**

### Rule 1 — Client → Firewall
*(Applied on HR_NET, IT_NET, GUEST_NET interfaces)*

| Field | Value |
|-------|-------|
| Action | Pass |
| Protocol | UDP |
| Source | \<interface\> net |
| Destination | This Firewall |
| Port | 67–68 |
| Description | DHCP request to relay |

Allows the initial client broadcast to reach OPNsense 
so the relay service can pick it up.

### Rule 2 — Firewall → DC1
*(Applied on SERVER_NET interface)*

| Field | Value |
|-------|-------|
| Action | Pass |
| Protocol | UDP |
| Source | This Firewall |
| Destination | 10.10.30.10 (DC1) |
| Port | 67 |
| Description | DHCP relay forward to DC1 |

Allows OPNsense to forward the relayed request to DC1. 
Without this rule the relay silently fails — 
the service shows as running but no client ever gets an IP. 
This two-rule chain is a detail that reflects a real 
understanding of how stateful firewalls interact with 
relay services.

---

## DHCP Scopes on DC1

Three scopes configured after installing and **authorizing** 
the DHCP role in Active Directory. Authorization is a 
required step — it is AD formally registering this server 
as a trusted DHCP source, preventing rogue DHCP servers 
from operating on the domain.

`DHCP Console → right-click server → Authorize → Refresh`

| Scope | Range | Gateway | DNS |
|-------|-------|---------|-----|
| HR | 10.10.10.100–200 | 10.10.10.1 | 10.10.30.10, 10.10.30.11 |
| IT | 10.10.20.100–200 | 10.10.20.1 | 10.10.30.10, 10.10.30.11 |
| GUEST | 10.10.40.100–200 | 10.10.40.1 | 10.10.40.1 |

Each scope pushes the correct gateway and DNS to clients 
automatically — HR clients get HR's gateway, IT clients 
get IT's gateway. No manual configuration needed on endpoints.

**GUEST scope exception:**  
DNS is set to `10.10.40.1` (OPNsense) instead of DC1/DC2. 
Guests get public DNS resolution through the firewall but 
cannot resolve any internal hostname like 
`FILE1.company.local` or `DC1.company.local` — 
keeping the internal naming structure invisible to 
untrusted devices.

