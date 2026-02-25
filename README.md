# 🏢 Enterprise IT Infrastructure Lab — Phase 1: Baseline

> **"You can't secure what you don't understand."**  
> This project builds a realistic company IT infrastructure — 
> intentionally mirroring the misconfigurations and weak practices 
> found in real enterprises — as a foundation for 
> **Phase 2: Zero Trust hardening.**

---

## 🎯 Project Goal

Most cybersecurity labs start with attacking or defending an 
already-built environment. This project takes a different approach:

**I built the company first.**

From scratch. The network, the servers, the identity system, 
the file sharing, the remote access — everything a real IT 
department would deploy. And like many real companies, this 
infrastructure was built prioritizing *functionality over security*, 
leaving behind the exact kinds of vulnerabilities that make 
enterprises targets.

Phase 2 will layer a full Zero Trust architecture on top of this, 
turning every weakness listed here into a hardened control.

---

## 🗺️ Infrastructure Overview

A multi-segment enterprise network running on VirtualBox + OPNsense, 
with a full Windows Server environment behind it.
```
                        [ INTERNET ]
                             |
                        [ OPNsense ]
                        (Firewall/Router)
                             |
                ┌──────┬──────┬──────┬
                │      │      │      │    
             HR_NET  IT_NET SERVER GUEST
             /10     /20    /30    /40
```

| Segment   | Subnet        | Purpose                          |
|-----------|---------------|----------------------------------|
| HR_NET    | 10.10.10.0/24 | HR department endpoints          |
| IT_NET    | 10.10.20.0/24 | IT department endpoints          |
| SERVER_NET| 10.10.30.0/24 | Internal servers (DC, File, FTP) |
| GUEST_NET | 10.10.40.0/24 | Untrusted / guest devices        |

---

## 🔧 What Was Built

### 🔥 Firewall — OPNsense
- Full routing between all 4 segments + WAN
- Each segment on a **dedicated adapter** (no VLAN tagging — 
  physical isolation in VirtualBox)
- DHCP relay configured so centralized DHCP on DC1 
  serves HR, IT, and GUEST across segments
- Basic rule sets per segment (detailed in `/docs/02-firewall.md`)

### 🪪 Identity — Active Directory
- **Domain:** `company.local`
- **DC1:** AD DS + DNS + DHCP — primary
- **DC2:** AD DS + DNS — replicates from DC1 for high availability
- **DFS** enabled across DC1 and DC2 for file redundancy
- Clean OU structure built for GPO-readiness:
- Security Groups: `HR_Users`, `IT_Admins`, `VPN_Users`, `FTP_Users`
- Users assigned to groups with role-based access in mind

### 📁 File Server — FILE1 (10.10.30.20)
- Domain-joined Windows Server in SERVER_NET
- SMB shares: **HR**, **IT**, **Public**
- Permissions controlled at NTFS level using AD groups
  (inheritance disabled — clean, explicit permissions per share)
- **Roaming Profiles** — user desktops/settings/documents 
  follow them across any domain PC via `\\FILE1\RoamingProfiles\%username%`

### 📤 FTP Server — on FILE1 (IIS)
- FTP site serving `C:\FTP\Shared`
- AD group authentication — only `FTP_Users` can connect
- Passive mode configured (ports 50000–50100)
- Firewall rules allow FTP access from IT_NET and HR_NET only

### 🌐 DHCP Architecture
- Single DHCP server on DC1 (centralized management)
- OPNsense acts as **DHCP relay** for cross-segment IP assignment
- Scopes: HR / IT / GUEST with correct gateways and DNS per segment
- GUEST DNS deliberately pointed to the firewall (10.10.40.1) — 
  not the internal DC, preventing internal name resolution by guests

---

## ⚠️ Baseline Security Gaps (Intentional)

This Phase 1 build prioritizes enterprise functionality first, 
leaving common "pre-hardening" gaps that Phase 2 will close:

- **Over-permissive inter-zone access** — limited micro-segmentation; 
  no strict default-deny east/west policy
- **Legacy auth exposure** — NTLM not fully restricted; 
  no certificate-based auth yet
- **Cleartext file transfer** — FTP enabled today; 
  planned upgrade to FTPS (TLS)
- **Directory traffic not enforced as secure** — 
  LDAP → LDAPS not enforced yet
- **File services not cryptographically enforced** — 
  SMB hardening/encryption/signing policies not fully enforced
- **No centralized detection layer** — 
  SIEM + advanced auditing comes in Phase 2
- **Admin model not tiered yet** — 
  no PAW / Tier 0-1-2 separation in Phase 1

---

## 🔜 Phase 2 — Zero Trust Hardening (Next)

Phase 2 will harden the baseline with identity + policy-driven controls:

- **AD CS / PKI** + certificate-based authentication (VPN/services)
- **MFA** for VPN + privileged access (RADIUS/NPS or equivalent)
- **Default-deny micro-segmentation** in OPNsense using 
  aliases + explicit allowlists
- **FTP → FTPS (TLS)**, LDAP → LDAPS, 
  and stronger SMB security posture
- **Tiered administration** — separate privileged accounts, 
  admin segmentation, PAW concept
- **SIEM + auditing** — firewall logs, AD events, 
  file access telemetry

→ Phase 2 repo link will be added once published.

---

## 🛠️ Technologies Used

| Tool                | Role                           |
|---------------------|--------------------------------|
| VirtualBox          | Virtualization platform        |
| OPNsense            | Firewall, routing, DHCP relay  |
| Windows Server 2019 | DC1, DC2, FILE1                |
| Active Directory DS | Identity & access management   |
| DNS / DHCP          | Name resolution, IP management |
| DFS                 | Distributed file redundancy    |
| IIS + FTP           | File transfer service          |
| SMB                 | Internal file sharing          |

---

## 📐 Diagrams & Screenshots

- Network topology: `/diagrams/network-topology.png`
- Full screenshots by component: `/screenshots/`

---

## 📁 Repository Structure
```
enterprise-infrastructure-architecture/
├── README.md
├── docs/
│   ├── 01-network-architecture.md
│   ├── 02-firewall-segmentation.md
│   ├── 03-active-directory.md
│   ├── 04-dhcp-infrastructure.md
│   └── 05-file-ftp-servers.md
├── diagrams/
│   └── network-topology.png
└── screenshots/
    ├── firewall/
    ├── active-directory/
    ├── file-server/
    └── ftp/
```
