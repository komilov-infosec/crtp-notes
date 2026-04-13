
# 🔍Stealth AD Enumeration: Operational Security (OPSEC) Playbook for EDR Evasion 

> **For lab environments, authorized penetration testing only.**
> 
> _Focus: Living off the Land (LotL), ADSI, LDAP Filters, and Evasion over Noise._

---

## 1. True Living off the Land (ADSI & .NET)

_Executing `net.exe` or `whoami.exe` leaves process creation logs (Event ID 4688). Using native .NET Active Directory Service Interfaces (ADSI) sends raw LDAP traffic directly from memory, completely bypassing process monitoring._


```PowerShell
# [OPSEC] Get current user domain and name without whoami.exe
[System.Security.Principal.WindowsIdentity]::GetCurrent().Name

# [ADSI] Enumerate all Domain Admins silently
$Group = [ADSI]"LDAP://CN=Domain Admins,CN=Users,DC=domain,DC=local"
$Group.Member

# [ADSI] Find all users in the domain via raw LDAP search
$Searcher = [adsisearcher]"(&(objectCategory=person)(objectClass=user))"
$Searcher.FindAll() | ForEach-Object { $_.Properties.name }

# [ADSI] Find specific SPN (Kerberoastable) without PowerView
$Searcher = [adsisearcher]"(&(objectCategory=person)(objectClass=user)(servicePrincipalName=*))"
$Searcher.FindAll() | ForEach-Object { $_.Properties.samaccountname }
```

---

## 2. Advanced PowerView (In-Memory Execution)

_Always run an AMSI bypass in memory before loading PowerView. Focus on specific, targeted queries rather than dumping the entire domain at once._

```PowerShell
# Import silently
Import-Module .\PowerView.ps1 -DisableNameChecking

# --- Targeted User Hunts ---
# Find Kerberoastable users and their SPNs
Get-DomainUser -SPN | Select-Object samaccountname, serviceprincipalname
# Find AS-REP Roastable users
Get-DomainUser -PreauthNotRequired | Select-Object samaccountname
# Hunt for passwords stored in user descriptions
Get-DomainUser | Where-Object {$_.description -ne $null} | Select-Object samaccountname, description

# --- Advanced Delegation Hunts ---
# Find hosts allowing Unconstrained Delegation
Get-DomainComputer -Unconstrained | Select-Object name, dnshostname
# Find Resource-Based Constrained Delegation (RBCD) targets
Get-DomainComputer | Where-Object {$_.msds-allowedtoactonbehalfofotheridentity} | Select-Object name

# --- High-Value ACL & GPO Hunts ---
# Find users with local admin rights pushed via GPO
Get-DomainGPOLocalGroup | Select-Object GPODisplayName, GroupName
# Find objects where our compromised user has dangerous rights (GenericAll, WriteDacl, etc.)
Find-InterestingDomainAcl -ResolveGUIDs | Where-Object {$_.IdentityReferenceName -eq "TargetUser"}
```

---

## 3. BloodHound / SharpHound (OPSEC Modes)

_Running `SharpHound.exe -c All` from a standard workstation creates a massive spike in LDAP queries and SAMR requests that SOCs look for. Use targeted collection methods._


```PowerShell
# [OPSEC] Stealth Collection - Only runs LDAP collection, avoids touching workstations/SAMR
Invoke-BloodHound -CollectionMethod DCOnly -OutputDirectory C:\Windows\Temp\

# [OPSEC] Targeted Session Collection - Only enumerate sessions on specific high-value targets (e.g., File Servers)
Invoke-BloodHound -CollectionMethod Session -ComputerFile targets.txt

# Session Enumeration via native API (No SharpHound needed)
# Requires NetSessionEnum privileges (often restricted in modern AD)
Get-NetSession -ComputerName TARGET_FILE_SERVER
```

---

## 4. Local Administrator Enumeration

_Finding where you have local admin rights without generating noisy Event 4624 (Logon) logs across the entire network._


```PowerShell
# Check if current user has local admin on a specific machine via RPC (avoids touching SMB/Admin$)
Find-LocalAdminAccess -ComputerName TARGET

# Map out local admins across the domain natively (Noisy - Requires high privileges)
Invoke-EnumerateLocalAdmin -Verbose
```

---

## 🛡️ OPSEC Notes & Telemetry Matrix

|**Tool / Technique**|**Detection Risk**|**Artifact Left / EDR Telemetry**|**OPSEC Mitigation**|
|---|---|---|---|
|`net.exe` / `whoami.exe`|**High**|Event 4688 (Process Creation). Highly monitored by EDRs.|Use `[ADSI]` and `[System.Security.Principal.WindowsIdentity]`.|
|SharpHound (`-c All`)|**High**|Massive LDAP traffic spike. Event 4624/4634 across all hosts.|Use `-c DCOnly` or targeted session enumeration.|
|PowerView|**Medium**|Event 4104 (PowerShell Script Block Logging).|Reflectively load PowerView via Invoke-ReflectivePEInjection or modify the script to rename suspicious function names like Get-Net* to Get-File* to evade keyword-based EDR detections.|
|Native ADSI (`[adsi]`)|**Very Low**|Blends in with normal domain LDAP queries.|N/A - The ultimate stealth enumeration method.|

---

_Created for Active Directory Red Teaming Operations._
