
````
# 🔴 Advanced Active Directory Red Teaming & CRTP Notes

![Red Team](https://img.shields.io/badge/Category-Red_Teaming-darkred?style=flat-square)
![Active Directory](https://img.shields.io/badge/Target-Active_Directory-blue?style=flat-square)
![OPSEC](https://img.shields.io/badge/OPSEC-Strict-green?style=flat-square)

A comprehensive, OPSEC-focused repository detailing advanced Active Directory exploitation techniques, lateral movement strategies, and domain dominance paths. These notes were compiled during the preparation for the **Certified Red Team Professional (CRTP)** examination and have been upgraded to reflect modern EDR-evasion and Living off the Land (LotL) methodologies.

## 🎯 Repository Philosophy

Unlike standard cheat sheets that rely heavily on dropping noisy binaries (e.g., standard `Mimikatz`, `PsExec`) and triggering massive EDR telemetry, this repository focuses on:
* **Living off the Land (LotL):** Utilizing native Windows APIs, ADSI, and PowerShell to blend in with legitimate administrative traffic.
* **In-Memory Execution:** Completely fileless payload execution to bypass traditional AV scanning.
* **OPSEC Awareness:** Prioritizing AES-256 over RC4, avoiding Event ID 4688 (Process Creation) anomalies, and executing targeted enumeration rather than noisy domain-wide scans.

## 📂 Repository Structure

```text
crtp-notes/
├── README.md
└── AD-Cheatsheet/
    ├── 01-Enumeration.md          # Stealth ADSI, PowerView, BloodHound OPSEC
    ├── 02-Lateral-Movement.md     # DCOM, WinRM, WMI, Pass-the-Ticket
    ├── 03-Privilege-Escalation.md # LPE, ACL Abuse, Constrained/Unconstrained Delegation
    ├── 04-Kerberos-Attacks.md     # Targeted Roasting, Native .NET TGS Requests
    └── 05-Persistence.md          # Diamond/Golden Tickets (AES), DSRM, SDProp
````

## 🛠️ Core Toolkit Referenced

- **Native Windows:** `ADSI`, `WMI`, `DCOM`, `WinRM`
    
- **C2 & Memory:** `PowerView` (Heavily modified/In-memory), `Rubeus`
    
- **Analysis:** `BloodHound` / `SharpHound` (Targeted Collection)
    
- **Execution:** `.NET Reflection`, `API Unhooking Concepts`
    

## ⚠️ Disclaimer

> **For Educational and Authorized Testing Purposes Only.**
> 
> All techniques, scripts, and commands provided in this repository are intended strictly for use in closed laboratory environments or during authorized penetration testing / Red Team engagements where explicit written consent has been granted by the infrastructure owner. The author assumes no liability and is not responsible for any misuse or damage caused by the information provided.

## 📚 Acknowledgments

- Massive respect to [Altered Security](https://www.alteredsecurity.com/) for the exceptional **CRTP (Certified Red Team Professional)** boot camp and lab environment.
    
- Inspiration drawn from the global InfoSec community, BloodHound Gang, and various open-source Red Team developers.
    

---

_Stay stealthy. Enumerate deeply. Exploit logically._
