# 02 — Firewall & Segmentation

## Overview

OPNsense is the single control point for all traffic in this 
infrastructure. Every packet crossing a segment boundary passes 
through it — making rule design one of the most critical parts 
of this project.

OPNsense processes rules **top to bottom, first match wins**. 
Rule order is not cosmetic — it is the logic itself. 
A misplaced rule can silently break a service or silently 
open access that should be blocked. Every rule set below 
was designed with this in mind.

---

## Interface Assignment

`Interfaces → Assignments → add each NIC → rename + enable + set Static IPv4`

| Interface | NIC | Gateway IP | Type |
|-----------|-----|-----------|------|
| WAN | em0 | 10.0.2.15 | NAT |
| MGMT | em1 | 192.168.56.10 | Host-Only |
| OPT1 → HR_NET | em2 | 10.10.10.1 | Internal |
| OPT2 → IT_NET | em3 | 10.10.20.1 | Internal |
| OPT3 → SERVER_NET | em4 | 10.10.30.1 | Internal |
| OPT4 → GUEST_NET | em5 | 10.10.40.1 | Internal |

Each interface was renamed from the default OPT label to its 
segment name — this keeps rule management readable and 
unambiguous across multiple interfaces.

---

## Rule Design Per Segment

### HR_NET / IT_NET / SERVER_NET — Temporary Baseline Rules

> ⚠️ **These rules are intentionally broad and temporary.**
> They exist purely to establish basic communication between 
> segments so the infrastructure functions as a working 
> enterprise environment. They will be completely replaced 
> in Phase 2 with strict, service-specific allowlists 
> under a default-deny policy.

| Field | Value |
|-------|-------|
| Action | Pass |
| Protocol | IPv4 any |
| Source | \<interface\> net |
| Destination | any |

This mirrors exactly how many real enterprises operate 
before any hardening — which is precisely why lateral 
movement is so easy after an initial compromise.

---

### GUEST_NET — Isolated Internet-Only Access

The GUEST rule set is where deliberate firewall thinking 
starts. Guests need internet access but must be completely 
blind to the internal network. This required **4 explicit 
rules in strict order:**

| # | Action | Protocol | Destination | Port | Purpose |
|---|--------|----------|-------------|------|---------|
| 1 | Pass | UDP | GUEST address | 67 | DHCP via relay |
| 2 | Pass | TCP/UDP | GUEST address | 53 | DNS via firewall |
| 3 | Block | any | HR/IT/SERVER/MGMT | any | Block internal access |
| 4 | Pass | any | any | any | Allow internet |

**Why order matters here:** Rule 4 allows everything — 
if it came before Rule 3, guests would reach internal 
networks before the block fires. Rules 1 and 2 must 
come first or guests would get no IP and no DNS even 
for internet browsing.

**Why DNS points to the firewall (10.10.40.1) and not DC1:**
Giving guests DC1 as DNS would let them resolve internal 
hostnames like `FILE1.company.local` — exposing the 
internal naming structure. OPNsense as DNS resolver 
gives guests public resolution only.

---

## DHCP Relay Rules

When DHCP relay is active, OPNsense intercepts client 
broadcast requests and forwards them as unicast to DC1. 
This two-hop flow requires two explicit rules:

**Rule 1 — Client → Firewall** (on HR/IT/GUEST interfaces)

| Field | Value |
|-------|-------|
| Protocol | UDP |
| Source | \<interface\> net |
| Destination | This Firewall / Port 67–68 |

**Rule 2 — Firewall → DC1** (on SERVER_NET interface)

| Field | Value |
|-------|-------|
| Protocol | UDP |
| Source | This Firewall |
| Destination | 10.10.30.10 / Port 67 |

A detail that catches many people off guard: the 
firewall's own forwarded traffic is subject to its 
own rules. Without Rule 2, the relay silently fails 
even though the service is enabled.

---

## FTP Access Rules

FTP uses **two separate port ranges** — port 21 for 
control and a passive range for data transfer. 
Allowing only port 21 lets users authenticate but 
breaks all file operations.

A named alias was created to manage the passive range cleanly:

`Firewall → Aliases → FTP_PASSIVE → Port(s) → 50000–50100`

If the range ever changes, it updates in one place 
and all rules reflect it automatically.

Rules applied on IT_NET (and HR_NET if needed):

| Rule | Protocol | Destination | Port |
|------|----------|-------------|------|
| FTP Control | TCP | 10.10.30.20 (FILE1) | 21 |
| FTP Data | TCP | 10.10.30.20 (FILE1) | FTP_PASSIVE |


