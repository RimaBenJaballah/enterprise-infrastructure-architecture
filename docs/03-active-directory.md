# 03 — Active Directory & Identity

## Overview

Active Directory is the identity backbone of this infrastructure. 
Every user, every computer, every permission, and every service 
authentication flows through it. Getting this layer right is what 
separates a real enterprise environment from a basic lab — so this 
phase was built with production habits in mind: clean OU structure, 
role-based group membership, redundant domain controllers, and 
proper delegation of services.

**Domain:** `company.local`

---

## Domain Controllers

Rather than running a single DC (which most labs do), 
this infrastructure runs **two domain controllers** — 
mirroring real enterprise high-availability design.

| Server | IP | Roles |
|--------|-----|-------|
| DC1 | 10.10.30.10 | AD DS + DNS + DHCP |
| DC2 | 10.10.30.11 | AD DS + DNS |

DC2 continuously replicates the AD database and DNS records 
from DC1. If DC1 goes down, authentication and name resolution 
continue through DC2 without interruption — no single point 
of failure for identity services.

**DHCP is intentionally kept on DC1 only** — centralizing 
IP management in one place keeps scope administration clean 
and avoids split-scope conflicts in a lab environment.

**DFS (Distributed File System)** was enabled across both 
DCs to provide file redundancy — ensuring shared resources 
remain available even if one server becomes unreachable.

---

## DC1 Configuration
```
Rename PC       → DC1
Static IP       → 10.10.30.10 / 255.255.255.0
Gateway         → 10.10.30.1 (OPNsense SERVER gateway)
DNS (temp)      → 1.1.1.1 (only during initial setup/updates)
```

Roles installed: **AD DS + DNS**
→ Promoted to DC, created new forest: `company.local`
→ After promotion, DNS updated to point to itself: `10.10.30.10`

---

## DC2 Configuration
```
Rename PC       → DC2
Static IP       → 10.10.30.11 / 255.255.255.0
Gateway         → 10.10.30.1
DNS             → 10.10.30.10 (DC1)
```

Roles installed: **AD DS + DNS**
→ Joined `company.local` domain first
→ Promoted as additional DC to existing domain
→ DNS + Global Catalog enabled
→ Credentials used during promotion: `company\administrator` 
  (same admin across both DCs — unified management)

---

## OU Structure

A clean Organizational Unit hierarchy was built under a 
single top-level OU `COMPANY` — keeping all company 
objects separate from default AD containers and making 
the structure GPO-ready for Phase 2.
```
company.local
└── COMPANY/
    ├── Users/
    │   ├── HR
    │   └── IT
    ├── Computers/
    │   ├── HR
    │   └── IT
    ├── Servers
    └── Groups
```

Separating Users, Computers, and Servers into distinct OUs 
means GPOs can be applied precisely — a policy targeting 
HR users won't accidentally apply to IT machines or servers.

---

## Security Groups

Created inside `COMPANY\Groups`:

| Group | Scope | Purpose |
|-------|-------|---------|
| HR_Users | Global / Security | Access to HR file shares |
| IT_Admins | Global / Security | Elevated access, IT shares |
| VPN_Users | Global / Security | Authorized remote access (Phase 2) |
| FTP_Users | Global / Security | FTP service access on FILE1 |

Using security groups rather than assigning permissions 
directly to users is a fundamental AD best practice — 
adding or removing a user from a group instantly 
updates all their access across every service that 
references that group.

---

## Users

| User | OU | Groups | Role |
|------|----|--------|------|
| Rima.HR | COMPANY\Users\HR | HR_Users | HR employee |
| Syrin.HR | COMPANY\Users\HR | HR_Users | HR employee |
| Admin.IT | COMPANY\Users\IT | IT_Admins | IT administrator |
| Remote.IT | COMPANY\Users\IT | IT_Admins, VPN_Users, FTP_Users | Remote IT worker |

User logon format: `firstname.department@company.local`  
Password: `user@123` *(intentionally weak — see security notes)*

---

## DHCP Scopes (on DC1)

After installing and authorizing the DHCP role 
(authorization = AD formally trusts this server to 
assign IPs), three scopes were created:

| Scope | Range | Gateway | DNS |
|-------|-------|---------|-----|
| HR | 10.10.10.100–200 | 10.10.10.1 | DC1 + DC2 |
| IT | 10.10.20.100–200 | 10.10.20.1 | DC1 + DC2 |
| GUEST | 10.10.40.100–200 | 10.10.40.1 | 10.10.40.1 (firewall only) |

**GUEST scope exception:** DNS is deliberately set to the 
OPNsense gateway instead of DC1/DC2 — guests can resolve 
public internet names but cannot resolve any internal 
hostname. A small but meaningful isolation detail.

OPNsense was then configured to disable its built-in DHCP 
server on HR/IT/GUEST interfaces and enable **DHCP relay** 
pointing to DC1 — so a single centralized server handles 
all IP assignment across all segments.


