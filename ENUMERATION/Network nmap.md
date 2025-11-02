# First enumeration

#Attacking_machine
Use *nmap* to see the services and ports on the victim machine.

```
nmap -A  <MACHINE IP> -Pn
```

![[Screenshot_2025-10-23_04-54-13.png]]

## Quick summary / high-value findings

- **SMB (139, 445) open** — Windows host called `HOSTEVASION`.
    
    - `smb2-security-mode` shows **message signing enabled but not required** → **SMB relay** and credential relay attacks are possible if you can capture NTLM auth.
        
- **RDP (3389) open** — RDP service present; `rdp-ntlm-info` leaked host/domain info and system version **Windows 10 / Server 2019 (10.0.17763)**. RDP may accept NTLM auth or have NLA; further checks required.
    
- **HTTP services on 8000, 8080, 8443** — two pages titled **PowerShell Script Analyser** (8080,8443). 8080 appears to be an Apache/PHP server; 8443 has an **expired cert** (2009–2019) — odd/leftover. 8080 may act as an **open proxy** (nmap suggested).
    
- **Time/stamp** from RDP/SSL matches your scan time → timing good.
    
- **Overall risk**: multiple attack vectors — SMB relaying, web app weaknesses, RDP brute/credential reuse, and lateral movement if you can capture/ crack creds.

**Next step:** [[Web Server]]

