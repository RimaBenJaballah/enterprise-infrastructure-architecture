```mermaid
%%{init: {"flowchart": {"nodeSpacing": 90, "rankSpacing": 110}}}%%
flowchart LR
  Internet([INTERNET]) --> WAN["WAN (NAT)\nOPNsense em0\n10.0.2.15"]
  WAN --> FW["OPNsense\nFirewall / Router"]

  FW --> HR["HR_NET\n10.10.10.0/24\nGW 10.10.10.1"]
  FW --> IT["IT_NET\n10.10.20.0/24\nGW 10.10.20.1"]
  FW --> GUEST["GUEST_NET\n10.10.40.0/24\nGW 10.10.40.1"]
  FW --> SNET

  subgraph SNET["SERVER_NET (10.10.30.0/24)"]
    DC1["DC1\n10.10.30.10\nAD DS / DNS / DHCP"]
    DC2["DC2\n10.10.30.11\nAD DS / DNS"]
    FILE1["FILE1\n10.10.30.20\nSMB / Roaming Profiles / IIS FTP"]
  end

  FW -. "DHCP Relay" .-> DC1
