# 02 — Firewall & Segmentation

## Overview

OPNsense is the single control point for all traffic in this 
infrastructure. Every packet crossing a segment boundary passes 
through it — making firewall rule design one of the most critical 
parts of this project.

OPNsense processes rules **top to bottom, first match wins**. 
This means rule order is not cosmetic — it is the logic. 
A misplaced rule can silently break a service or silently 
open access that should be blocked. Every rule set below 
was designed with this in mind.

---

## Interface Assignment

Before any rule can be written, each virtual adapter must be 
assigned to a named interface inside OPNsense. This is done via:

`Interfaces → Assignments → add each NIC → rename + enable + set Static IPv4`

| Interface Name | NIC | IP (Gateway) | Type |
|---------------|-----|-------------|------|
| WAN | em0 | 10.0.2.15 | NAT |
| MGMT | em1 | 192.168.56.10 | Host-Only |
| OPT1 → HR_NET | em2 | 10.10.10.1 | Internal |
| OPT2 → IT_NET | em3 | 10.10.20.1 | Internal |
| OPT3 → SERVER_NET | em4 | 10.10.30.1 | Internal |
| OPT4 → GUEST_NET | em5 | 10.10.40.1 | Internal |

Each interface was renamed from the default OPT label to its 
segment name — this matters because OPNsense uses these names 
throughout the rule interface, aliases, and DHCP relay config. 
Keeping them meaningful avoids mistakes when managing multiple 
interfaces simultaneously.

---

## Rule Design Per Segment

### HR_NET / IT_NET / SERVER_NET — Baseline Connectivity

At this phase, internal company segments use a broad allow rule 
to establish full connectivity — simulating a typical enterprise 
before any hardening has been applied:

| Field | Value |
|-------|-------|
| Action | Pass |
| Protocol | IPv4 any |
| Source | \<interface\> net |
| Destination | any |

**Why this reflects reality:**
Many real enterprises run exactly this. IT teams add services 
incrementally over years and default to "allow all" internally 
to avoid breaking things. This is the exact posture that makes 
lateral movement trivial after an initial compromise — and it 
is intentional here as the baseline Phase 2 will eliminate.

---

### GUEST_NET — Isolated Internet-Only Access

The GUEST rule set is where real firewall thinking starts. 
Guests need internet access but must be completely blind to 
the internal network. This required **4 explicit rules in a 
specific order** — order that matters because of how OPNsense 
evaluates top-to-bottom:

| # | Action | Protocol | Source | Destination | Port | Purpose |
|---|--------|----------|--------|-------------|------|---------|
| 1 | Pass | UDP | GUEST net | GUEST address | 67 | DHCP requests reach the relay |
| 2 | Pass | TCP/UDP | GUEST net | GUEST address | 53 | DNS resolution via firewall |
| 3 | Block | IPv4 any | GUEST net | HR/IT/SERVER/MGMT | any | Block all internal access |
| 4 | Pass | IPv4 any | GUEST net | any | any | Allow internet |

**Why rule order is critical here:**

Rule 3 blocks GUEST from reaching internal networks. 
Rule 4 allows GUEST to reach anything. If rule 4 came 
before rule 3, guests would reach internal networks 
before the block could fire — the block would never 
be evaluated. Rules 1 and 2 must come first for the 
same reason: without explicit DHCP and DNS allow rules 
before any broader logic, guests would get no IP and 
no name resolution even for internet browsing.

**Why DNS points to the firewall (10.10.40.1) and not DC1:**

Giving guests DC1 as their DNS server would let them 
resolve internal hostnames like `FILE1.company.local` 
or `DC1.company.local` — exposing the internal naming 
structure. Pointing DNS to OPNsense itself means guests 
can resolve public internet names only. Simple, effective, 
and requires zero extra infrastructure.

---

## DHCP Relay Rules

When DHCP relay is enabled, OPNsense intercepts broadcast 
DHCP requests from clients and forwards them as unicast 
to DC1. This introduces a two-hop flow that requires 
two separate firewall rules — one on each side:

### Rule 1 — Client to Firewall (DHCP Discovery)

Applied on HR_NET, IT_NET, GUEST_NET interfaces:

| Field | Value |
|-------|-------|
| Action | Pass |
| Protocol | UDP |
| Source | \<interface\> net |
| Destination | This Firewall |
| Destination Port | 67–68 |
| Description | DHCP request to relay |

This allows the initial DHCP broadcast to reach OPNsense 
so the relay can pick it up.

**Note:** For HR_NET and IT_NET, the existing "allow any" 
rule already covers this. The explicit rule was noted 
for documentation purposes and will become mandatory in 
Phase 2 when "allow any" is removed and only specific 
traffic is permitted.

For GUEST_NET, rule 1 in the GUEST set (above) already 
covers this explicitly.

### Rule 2 — Firewall to DC1 (Relay Forward)

Applied on SERVER_NET interface:

| Field | Value |
|-------|-------|
| Action | Pass |
| Protocol | UDP |
| Source | This Firewall |
| Destination | 10.10.30.10 (DC1) |
| Destination Port | 67 |
| Description | DHCP relay to DC1 |

This allows OPNsense itself to forward the relayed 
request to DC1. Without this rule, the relay would 
silently fail — the firewall's own forwarded traffic 
is subject to its own rules, a detail that catches 
many people off guard.

---

## FTP Access Rules

FTP requires careful firewall handling because it uses 
**two separate port ranges** — port 21 for control 
(login, commands) and a passive port range for actual 
data transfer (file listing, upload, download). 
Allowing only port 21 will let users authenticate 
but fail on any file operation.

### Alias — FTP_PASSIVE

Rather than typing the port range in every rule, 
a named alias was created:

`Firewall → Aliases → Add`

| Field | Value |
|-------|-------|
| Name | FTP_PASSIVE |
| Type | Port(s) |
| Content | 50000–50100 |

This alias is referenced in all FTP rules — if the 
passive range ever changes, it is updated in one place 
and all rules reflect it automatically.

### Rules on IT_NET (FTP access to FILE1)

| Rule | Action | Protocol | Source | Destination | Port |
|------|--------|----------|--------|-------------|------|
| A | Pass | TCP | IT_NET net | 10.10.30.20 | 21 |
| B | Pass | TCP | IT_NET net | 10.10.30.20 | FTP_PASSIVE |

### Rules on HR_NET (if HR requires FTP access)

Same as above with HR_NET net as source. 
If HR does not need FTP, these rules are omitted — 
least privilege applied at the firewall level, 
not just at the application level.

---

## Rule Summary — Current State

| Segment | Policy | Notes |
|---------|--------|-------|
| HR_NET | Allow any → any | Baseline only, intentionally broad |
| IT_NET | Allow any → any | Baseline only, intentionally broad |
| SERVER_NET | Allow any → any | Baseline only, intentionally broad |
| GUEST_NET | 4-rule explicit policy | DHCP + DNS allowed, internal blocked, internet allowed |
| FTP (IT/HR → FILE1) | Explicit TCP 21 + passive | Alias-based, scoped to FILE1 only |
| DHCP Relay | Two-rule chain | Client → Firewall + Firewall → DC1 |

---

## Security Notes (Phase 1 Gaps)

- **No default-deny east/west policy** — internal segments 
  trust each other implicitly
- **No stateful inspection tuning** — OPNsense stateful 
  firewall is active but not configured beyond defaults
- **No IDS/IPS** — Suricata/Zenarmor not enabled; 
  malicious lateral traffic would pass undetected
- **FTP is cleartext** — port 21 rules allow unencrypted 
  credential and file transfer across the network
- **No logging policy** — rules do not have logging enabled; 
  there is no audit trail of allowed or denied connections

*All of the above are addressed in Phase 2.*
