```mermaid
flowchart TB
  Internet([INTERNET]) --> WAN[WAN (NAT)<br/>OPNsense em0<br/>10.0.2.15]

  WAN --> FW[OPNsense Firewall/Router]

  FW --> HR[HR_NET<br/>10.10.10.0/24<br/>GW: 10.10.10.1]
  FW --> IT[IT_NET<br/>10.10.20.0/24<br/>GW: 10.10.20.1]
  FW --> GUEST[GUEST_NET<br/>10.10.40.0/24<br/>GW: 10.10.40.1]
  FW --> SERVER[SERVER_NET<br/>10.10.30.0/24<br/>GW: 10.10.30.1]

  subgraph SERVER["SERVER_NET (10.10.30.0/24)"]
    DC1[DC1<br/>10.10.30.10<br/>AD DS / DNS / DHCP]
    DC2[DC2<br/>10.10.30.11<br/>AD DS / DNS]
    FILE1[FILE1<br/>10.10.30.20<br/>SMB Shares / Roaming Profiles / IIS FTP]
  end

  FW -. "DHCP Relay" .-> DC1
