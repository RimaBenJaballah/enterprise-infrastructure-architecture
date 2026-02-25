# 05 — File Server, FTP & User Services

## Overview

FILE1 is the most service-rich server in this infrastructure. 
It handles SMB file sharing with AD-integrated permissions, 
roaming profiles so users carry their environment across any 
domain PC, and an FTP service for file transfer — all running 
on a single domain-joined Windows Server in SERVER_NET.

Consolidating FILE and FTP on one server is a lab practicality 
(RAM). In a real enterprise these would be separated — but the 
configuration, permissions model, and AD integration are 
identical either way.

**FILE1 — 10.10.30.20**

---

## Domain Join & Role Installation

Before FILE1 can use AD groups for permissions, it must be 
a trusted member of the domain:
```
Rename PC    → FILE1
Static IP    → 10.10.30.20 / 255.255.255.0
Gateway      → 10.10.30.1
DNS Primary  → 10.10.30.10 (DC1)
DNS Alt      → 10.10.30.11 (DC2)
```

`System → Rename this PC (advanced) → Domain: company.local`  
→ Authenticate with `company\Administrator`  
→ Restart

Once joined, the FILE1 computer object was moved from the 
default Computers container into `COMPANY\Servers` in ADUC — 
keeping the AD structure clean and ensuring any future 
server-targeted GPOs apply correctly.

**Role installed:** File and Storage Services → 
File and iSCSI Services → File Server

Adding the role formally is a professional habit — 
the server can technically share folders without it, 
but the role enables proper management through 
Server Manager and makes the server's purpose explicit 
in the infrastructure.

---

## SMB File Shares

Three shares created under `C:\Shares\`:

| Share | Path | Intended Users |
|-------|------|---------------|
| HR | C:\Shares\HR | HR_Users group |
| IT | C:\Shares\IT | IT_Admins group |
| Public | C:\Shares\Public | All domain users |

### Permissions Model

A deliberate two-layer permission model was applied — 
the same approach used in production environments:

**Share permissions (broad):**  
Set to `Authenticated Users = Full Control` on all shares.  
This layer is kept intentionally permissive.

**NTFS permissions (the real control):**  
This is where actual access is enforced using AD groups.

| Folder | Group | NTFS Permission |
|--------|-------|----------------|
| C:\Shares\HR | HR_Users | Modify |
| C:\Shares\IT | IT_Admins | Modify |
| C:\Shares\Public | Domain Users | Read |

**Why this model:**  
Effective access = the *more restrictive* of Share + NTFS. 
By keeping Share permissions broad and controlling 
everything at the NTFS level with AD groups, 
there is a single place to manage access. 
Adding a user to HR_Users instantly grants them 
access to the HR share — removing them instantly 
revokes it. No need to touch the server.

### Inheritance — Disabled on All Shares

Before applying any permissions, inheritance was 
disabled on each folder:

`Properties → Security → Advanced → Disable Inheritance`  
→ *Convert inherited permissions into explicit permissions*

Then the local `Users (FILE1\Users)` entry was removed.

**Why this matters:**  
Without disabling inheritance, the parent folder's 
permissions bleed into the share — local machine users 
could end up with unintended access. Disabling 
inheritance and converting gives a clean, 
explicit permission set that is fully under control.

---

## Roaming Profiles

Roaming profiles allow users to log into **any** 
domain-joined PC and have their full environment 
follow them — desktop, documents, settings, 
application configurations. Every change they make 
is written back to the file server on logout.

This simulates real enterprise mobility and is a 
detail that goes well beyond basic lab setups.

### Setup on FILE1

**1 — Create and share the folder:**
```
Folder:      C:\RoamingProfiles
Share name:  RoamingProfiles
```

Share permissions:  
`Authenticated Users → Change + Read`

NTFS permissions:  
`Authenticated Users → Full Control`  
*(users need full control to create and manage 
their own profile subfolder)*

**2 — Configure each user in AD (on DC1):**

`ADUC → open user → Profile tab → Profile path:`
```
\\FILE1\RoamingProfiles\%username%
```

The `%username%` variable automatically resolves to 
each user's logon name — no need to set individual 
paths per user. After first login and logout, 
a personal profile folder is created automatically 
under `C:\RoamingProfiles\`.

Applied to all HR and IT users.

---

## DFS — Distributed File System

DFS was enabled across DC1 and DC2 to provide 
**file redundancy** for shared resources. 

If FILE1 or DC1 becomes unavailable, files replicated 
through DFS remain accessible through DC2. 
This reflects real enterprise thinking — 
critical shared data should never have a single point 
of failure.

---

## FTP Server (IIS)

FTP runs on FILE1 via IIS — keeping file transfer 
and file storage on the same server while using 
AD group authentication to control who can connect.

### Installation

`Add Roles and Features → Web Server (IIS)`  
→ Under Role Services: check **FTP Service + FTP Extensibility**

### FTP Folder
```
C:\FTP\Shared
```

This is the root directory users land in when 
they connect. Access controlled by AD group.

### AD Group — FTP_Users

Only members of `FTP_Users` can authenticate 
to the FTP service. Remote.IT was added to 
this group specifically for FTP access:

`ADUC → COMPANY\Groups → FTP_Users → Members → Add → Remote.IT`

### FTP Site Configuration in IIS

| Field | Value |
|-------|-------|
| Site name | FTP-Shared |
| Physical path | C:\FTP\Shared |
| IP | 10.10.30.20 |
| Port | 21 |
| SSL | No SSL (Phase 1 — cleartext intentional) |
| Authentication | Basic (AD-integrated) |
| Authorization | FTP_Users group — Read + Write |

### Passive Mode — Port Range

FTP passive mode is essential for firewall 
compatibility. Without it, data connections 
fail silently after successful authentication.

`IIS Manager → FILE1 (server level) → FTP Firewall Support`  
→ Data Channel Port Range: **50000–50100**

Two rules added to Windows Defender Firewall on FILE1:

| Rule | Protocol | Port |
|------|----------|------|
| FTP Control | TCP Inbound | 21 |
| FTP Passive Data | TCP Inbound | 50000–50100 |

OPNsense rules were also added 
(see `/docs/02-firewall-segmentation.md`) to allow 
FTP traffic from IT_NET and HR_NET to FILE1 only — 
scoped by destination IP, not just port.

---

## Security Notes (Phase 1 Gaps)

- **FTP runs on port 21 — cleartext** — credentials 
  and file contents are transmitted unencrypted and 
  are sniffable on the network. Upgrade to FTPS (TLS) 
  is planned for Phase 2
- **Basic authentication over FTP** — AD passwords 
  exposed in cleartext during the FTP handshake
- **SMB signing not enforced** — shares are 
  vulnerable to NTLM relay attacks
- **Roaming profiles over unencrypted SMB** — 
  profile data including cached settings transferred 
  without encryption
- **No file access auditing** — who accessed or 
  modified what in the shares is not logged

*All of the above are addressed in Phase 2.*
