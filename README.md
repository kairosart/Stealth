# Quick summary (high level)

- The box is a Windows host running several services (SMB, RDP, multiple HTTP ports). The interesting attack surface is a web app called _PowerShell Script Analyser_ accessible on 8080/8443 that exposes an uploads area. [System Weakness+1](https://systemweakness.com/stealth-tryhackme-write-up-aa684e97575a)
    

## **Step-by-step walkthrough**

1. **Recon / port scan**
    
    - Run a full nmap scan to discover open services (notably 8080 & 8443 show the PowerShell Script Analyser web app). [System Weakness](https://systemweakness.com/stealth-tryhackme-write-up-aa684e97575a)
        
2. **Web enumeration → find upload area / scripts**
    
    - Inspect the web app and uploads directory. You’ll find existing `.ps1` files (including `vulnerable.ps1` / `rev.ps1`) and a `log.txt` that records uploads/types. This indicates the app will serve or execute PowerShell files from `C:\xampp\htdocs\uploads`. [System Weakness+1](https://systemweakness.com/stealth-tryhackme-write-up-aa684e97575a)
        
3. **Initial foothold: use a PowerShell reverse shell**
    
    - Modify the found reverse-shell `.ps1` (change your IP/port), upload it to the uploads folder and **remove**/clean the `log.txt` so the simulated blue-team/AV doesn’t flag you. Start a listener locally and trigger the shell (PowerShell execution / the app will load/execute the uploaded script). You get a user shell as `evader` and can capture the user flag. [System Weakness+1](https://systemweakness.com/stealth-tryhackme-write-up-aa684e97575a)
        
4. **Post-exploit enumeration**
    
    - From the shell, run common Windows enumeration (or a script like `PrivescCheck.ps1`) — find that the user has `SeImpersonatePrivilege`/`SeImpersonatePrivilege` (impersonation) enabled. That points to token-stealing or impersonation-style LPEs (e.g., EfsPotato / JuicyPotato family). [System Weakness+1](https://systemweakness.com/stealth-tryhackme-write-up-aa684e97575a)
        
5. **Privilege escalation: use EfsPotato (or similar) to get SYSTEM / admin**
    
    - Transfer a compiled exploit (EfsPotato binary) to the host, build if needed via `csc.exe`, run it to abuse `SeImpersonatePrivilege` and obtain SYSTEM / create an administrator account. Once you have credentials, you can RDP in or otherwise gain full control and capture the root flag. [System Weakness+1](https://systemweakness.com/stealth-tryhackme-write-up-aa684e97575a)
        

## **Practical tips & gotchas**

- The room simulates active defensive controls (logs/AV). Delete/clean evidence files like `log.txt` if the exercise requires stealth. [System Weakness](https://systemweakness.com/stealth-tryhackme-write-up-aa684e97575a)
    
- Use `powershell -ep bypass -c` to run downloaded scripts when needed. [System Weakness](https://systemweakness.com/stealth-tryhackme-write-up-aa684e97575a)
    
- Compile exploits on the target if required (`csc.exe` is commonly available) to avoid AV detection by running unsigned binaries pulled from elsewhere. [System Weakness](https://systemweakness.com/stealth-tryhackme-write-up-aa684e97575a)
    

## Flow

- Recon → upload PS reverse shell → get user → PrivescCheck → EfsPotato → admin/RDP.


---

# Defense measures
Defensive playbook to harden a Windows host like the TryHackMe **Stealth** box (focus: stop malicious PowerShell uploads, block “Potato” LPEs that abuse SeImpersonatePrivilege, and reduce attack surface).

I’ll split this into focused controls (what to change), _why_ it helps, and short practical steps you can apply.


## Web application / file-upload hardening

Why: the lab’s initial foothold is an uploaded `.ps1` served/executed from `uploads` — preventing or inspecting uploads closes that vector. [cheatsheetseries.owasp.org](https://cheatsheetseries.owasp.org/cheatsheets/File_Upload_Cheat_Sheet.html?utm_source=chatgpt.com)

Controls & steps

- **Reject/validate file types server-side** (never rely on client checks). Accept only the exact MIME/bytes you expect; block scripts (`.ps1`, `.exe`, `.dll`) outright unless needed. [cheatsheetseries.owasp.org](https://cheatsheetseries.owasp.org/cheatsheets/File_Upload_Cheat_Sheet.html?utm_source=chatgpt.com)
    
- **Store uploads outside the webroot and never serve/execute them directly.** Serve processed/converted content via a safe pipeline. Rename files on save so original names/extensions aren’t usable. [cheatsheetseries.owasp.org](https://cheatsheetseries.owasp.org/cheatsheets/File_Upload_Cheat_Sheet.html?utm_source=chatgpt.com)
    
- **Scan uploads in-line** with antivirus / malware-scanning engines (local AV or cloud scanning / WAF content-scanning) before any processing. Consider integration with services that scan content for known malware signatures and suspicious heuristics. [cheatsheetseries.owasp.org+1](https://cheatsheetseries.owasp.org/cheatsheets/File_Upload_Cheat_Sheet.html?utm_source=chatgpt.com)
    
- **Enforce size limits, content sniffing, and sanitization.** Log and quarantine suspicious uploads for manual review. [cheatsheetseries.owasp.org](https://cheatsheetseries.owasp.org/cheatsheets/File_Upload_Cheat_Sheet.html?utm_source=chatgpt.com)
    


## PowerShell / Script control

Why: the attacker used a PowerShell reverse shell. Hardening PowerShell blocks or detects such abuse. [Microsoft Learn](https://learn.microsoft.com/en-us/powershell/scripting/security/security-features?view=powershell-7.5&utm_source=chatgpt.com)

Controls & steps

- **Enable PowerShell logging and telemetry**: Script Block Logging, Module Logging, and Transcription (via Group Policy). These create high-fidelity artifacts for detection and forensic review. [Microsoft Learn+1](https://learn.microsoft.com/en-us/answers/questions/620492/configuring-powershell-to-always-use-constrained-l?utm_source=chatgpt.com)
    
- **Enforce Constrained Language Mode** where appropriate and block untrusted scripts using AppLocker or Windows Defender Application Control (WDAC). CLM reduces script capabilities (no arbitrary .NET, COM). Use AppLocker rules to block `.ps1` execution from non-approved locations. [Patch My PC+1](https://patchmypc.com/blog/constrained-language-mode-custom-detection/?utm_source=chatgpt.com)
    
- **Harden ExecutionPolicy via GPO** and combine with AppLocker/WDAC — ExecutionPolicy alone is insufficient, but useful as part of layered controls. [Microsoft Learn+1](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.security/set-executionpolicy?view=powershell-7.5&utm_source=chatgpt.com)
    


## Privilege / Local rights hardening (SeImpersonatePrivilege & “Potato” mitigations)

Why: Potato-style exploits (Juicy/Efs/Rotten/…Potato) abuse impersonation rights (SeImpersonatePrivilege). Reducing exposure prevents these LPE chains. [Microsoft Learn+1](https://learn.microsoft.com/en-us/troubleshoot/windows-server/windows-security/seimpersonateprivilege-secreateglobalprivilege?utm_source=chatgpt.com)

Controls & steps

- **Minimize assignments of SeImpersonatePrivilege** — ensure only required service accounts and SYSTEM have this right. Use Group Policy → Computer Configuration → Windows Settings → Security Settings → Local Policies → User Rights Assignment to review and tighten. [Microsoft Learn](https://learn.microsoft.com/en-us/troubleshoot/windows-server/windows-security/seimpersonateprivilege-secreateglobalprivilege?utm_source=chatgpt.com)
    
- **Avoid running services as high-privilege service accounts** — prefer Managed Service Accounts or least-privileged service accounts. Don’t give unnecessary accounts Local Admin. [Shuciran Pentesting Notes+1](https://shuciran.github.io/posts/SeImpersonatePrivilege/?utm_source=chatgpt.com)
    
- **Block common exploit behaviors with EDR**: monitor token manipulation, suspicious use of `csc.exe`, `rundll32`, attempts to call COM objects or LSASS-like behaviors. Modern EDRs detect Potato patterns and token-stealing attempts. [SentinelOne](https://www.sentinelone.com/labs/relaying-potatoes-another-unexpected-privilege-escalation-vulnerability-in-windows-rpc-protocol/?utm_source=chatgpt.com)
    


## RDP / network & access controls

Why: escalation often leads to lateral movement / RDP access; hardening prevents post-exploit takeover. [security.berkeley.edu](https://security.berkeley.edu/education-awareness/securing-remote-desktop-rdp-system-administrators?utm_source=chatgpt.com)

Controls & steps

- **Disable RDP if not needed.** If needed: restrict to known management IPs via Windows Firewall, require NLA (Network Level Authentication), and use MFA for remote access. [security.berkeley.edu](https://security.berkeley.edu/education-awareness/securing-remote-desktop-rdp-system-administrators?utm_source=chatgpt.com)
    
- **Deny direct RDP logon for local Administrator** (set “Deny logon through Remote Desktop Services” for the local admin) and require separate admin jump hosts for privileged sessions. [Server Fault+1](https://serverfault.com/questions/598278/how-to-disable-rdp-access-for-administrator?utm_source=chatgpt.com)
    
- **Use secure admin hosts** (dedicated, locked-down machines for admin work), and restrict which hosts can manage domain/controller operations. [Microsoft Learn](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/plan/security-best-practices/implementing-secure-administrative-hosts?utm_source=chatgpt.com)
    


## Detection, logging, and response

Why: prevention will fail sometimes — good detection limits impact and speeds response.

Controls & steps

- **Centralize logs to a SIEM** (Windows Event logs, PowerShell ScriptBlock logs, webserver logs) and create alerts for: uploaded script executions, new accounts, token impersonation events, `csc.exe` launches, and suspicious network callbacks. [Microsoft Learn+1](https://learn.microsoft.com/en-us/answers/questions/620492/configuring-powershell-to-always-use-constrained-l?utm_source=chatgpt.com)
    
- **Enable Windows Defender / EDR** with runtime protection, attack-surface reduction rules, and exploit protection. Configure rules to block known malicious tools and suspicious child processes. [Microsoft Learn+1](https://learn.microsoft.com/en-us/powershell/scripting/security/security-features?view=powershell-7.5&utm_source=chatgpt.com)
    
- **Monitor for account/privilege changes** and create alerts for assignment of SeImpersonatePrivilege or new high-privilege service accounts. [Microsoft Learn](https://learn.microsoft.com/en-us/troubleshoot/windows-server/windows-security/seimpersonateprivilege-secreateglobalprivilege?utm_source=chatgpt.com)
    


## Patching, configuration, and least privilege

Why: basic hygiene stops many chains.

Controls & steps

- **Keep Windows up to date** and ensure all server apps (web servers, frameworks) are patched.
    
- **Run services with least privilege** (avoid local admin). Use service accounts with only required rights. [Microsoft Support](https://support.microsoft.com/en-us/topic/security-configuration-guidance-support-ea9aef24-347f-15fa-b94f-36f967907f2f?utm_source=chatgpt.com)
    
- **Remove/lockdown developer tools** on production boxes (e.g., `csc.exe`, build tools) or restrict who can execute them; attackers often compile on-target to evade AV. Use WDAC/AppLocker to block unexpected compilers. [Microsoft Learn+1](https://learn.microsoft.com/en-us/powershell/scripting/security/security-features?view=powershell-7.5&utm_source=chatgpt.com)
    


## Quick prioritized checklist (apply in this order)

1. Move upload directory outside webroot + block script extensions from uploads. [cheatsheetseries.owasp.org](https://cheatsheetseries.owasp.org/cheatsheets/File_Upload_Cheat_Sheet.html?utm_source=chatgpt.com)
    
2. Enable PowerShell ScriptBlock logging, Module logging, and Transcription via GPO. [Microsoft Learn+1](https://learn.microsoft.com/en-us/answers/questions/620492/configuring-powershell-to-always-use-constrained-l?utm_source=chatgpt.com)
    
3. Implement AppLocker or WDAC policies to block unapproved script/binary execution. [Microsoft Learn](https://learn.microsoft.com/en-us/powershell/scripting/security/security-features?view=powershell-7.5&utm_source=chatgpt.com)
    
4. Audit and remove unnecessary accounts from SeImpersonatePrivilege; apply least privilege. [Microsoft Learn](https://learn.microsoft.com/en-us/troubleshoot/windows-server/windows-security/seimpersonateprivilege-secreateglobalprivilege?utm_source=chatgpt.com)
    
5. Deploy or tune EDR to detect token-stealing and suspicious child-process behaviors; centralize alerts. [SentinelOne](https://www.sentinelone.com/labs/relaying-potatoes-another-unexpected-privilege-escalation-vulnerability-in-windows-rpc-protocol/?utm_source=chatgpt.com)
    
6. Restrict RDP (or require NLA+MFA) and deny local admin RDP logon.

