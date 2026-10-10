# Active Directory Home Lab: Phase 2 (Attack Simulation & Detection)

![Active Directory](https://img.shields.io/badge/Active_Directory-Security-0078D4?style=for-the-badge)
![Kali Linux](https://img.shields.io/badge/Kali_Linux-Attacker_VM-557C94?style=for-the-badge&logo=kalilinux&logoColor=white)
![Status](https://img.shields.io/badge/Phase_2-Complete-2ea44f?style=for-the-badge)

**Tools:** Responder · Hashcat · Impacket · bloodhound-python · Kali Linux
**Attacks:** LLMNR poisoning · Kerberoasting · Pass-the-Hash · BloodHound enumeration
**MITRE ATT&CK:** T1557.001, T1558.003, T1550.002, T1087, T1069
**Phase:** 2 of 2, builds on [Phase 1](https://github.com/MrBSykes/active-directory-home-lab-phase-1-environment-building-and-IT-administartion)
**Date completed:** May 27, 2026

Full write-up: [`docs/active-directory-phase-2-documentation.pdf`](docs/active-directory-phase-2-documentation.pdf)

---

## Project Overview

Phase 2 turns the Active Directory environment from Phase 1 into an attack simulation lab. A Windows 10 workstation (WS01) was domain-joined as the target, and Kali Linux was added as the attacker machine. Four common Active Directory attack techniques were run against the `apexdefense.local` domain, each one documented with what happened, the Windows Event ID signatures that would detect it, and the prevention controls that stop it.

The goal is to show both sides of the same environment: Phase 1 is how it's built and administered, Phase 2 is how it gets attacked and defended.

---

## Lab Network

| VM | OS | IP address | Role |
|---|---|---|---|
| DC01 | Windows Server 2022 | 192.168.1.10 | Domain controller, `apexdefense.local` |
| WS01 | Windows 10 Pro | 192.168.1.20 | Domain-joined workstation, attack target |
| Kali Linux | Kali 2025.2 | 192.168.1.100 | Attacker machine |

All three VMs run on an isolated VirtualBox internal network (`apexdefense-lab`) with no internet access.

---

## Attacks Executed

Full exploitation steps, command syntax, and cracked-credential detail are in the PDF. Summary:

| # | Attack | MITRE | Tool | Result |
|---|---|---|---|---|
| 1 | LLMNR poisoning | T1557.001 | Responder | Captured NTLMv2 hashes from WS01 broadcast traffic; cracked offline with Hashcat |
| 2 | Kerberoasting | T1558.003 | Impacket GetUserSPNs | Requested a TGS ticket for the `sqlservice` SPN and cracked its hash offline |
| 3 | Pass-the-Hash | T1550.002 | Impacket secretsdump / psexec | Dumped domain hashes via DRSUAPI, then authenticated to WS01 with a hash alone and got a SYSTEM shell |
| 4 | BloodHound enumeration | T1087, T1069 | bloodhound-python | Mapped the domain's users, groups, and ACLs; identified a path from a low-privilege user to Domain Admin |

**Full attack chain:** LLMNR poisoning gave an initial credential → Kerberoasting compromised a service account → Pass-the-Hash reached a SYSTEM shell on WS01 → BloodHound mapped the path from there to Domain Admin.

### Detection and prevention, by attack

| Attack | Detect with | Prevent with |
|---|---|---|
| LLMNR poisoning | Event ID 4625/4648; network monitoring for LLMNR/NBT-NS responses from unexpected hosts | Disable LLMNR and NBT-NS via GPO, enable SMB signing, enforce strong passwords |
| Kerberoasting | Event ID 4769 with RC4 encryption (etype 0x17) | Group Managed Service Accounts, audit SPN-registered accounts, enforce AES Kerberos encryption |
| Pass-the-Hash | Event ID 4624 (NTLM network logon), Event ID 7045 (new service installed) | Credential Guard, Protected Users group, LAPS for unique local admin passwords |
| BloodHound enumeration | High-volume LDAP queries from a workstation in a short window; Event ID 4662 | Microsoft Defender for Identity, tiered administration, clean up excess ACLs, LDAP signing |

---

## Key Troubleshooting — Clock Skew

The main challenge in Phase 2 was a persistent Kerberos clock-skew error from a 5+ hour offset between Kali and DC01 (Kerberos requires clocks within 5 minutes to prevent replay attacks). Root causes: VirtualBox Guest Additions kept overriding manual time changes on Kali, DC01 had Daylight Saving Time disabled, and DC01's hardware clock was being read as UTC when Windows expected local time. Fixed by disabling the host time sync, correcting the DST setting, manually syncing the clocks, and adding DC01 to Kali's `/etc/hosts`.

---

## Screenshots

| # | Screenshot | What it shows |
|---|---|---|
| 00 | ![](screenshots/00-ws01-VirtualBox-ISO-configuration.png) | WS01 VM created from the Windows 10 ISO |
| 01 | ![](screenshots/01-windows10-desktop.png) | Windows 10 installed |
| 02 | ![](screenshots/02-ws01-static-ip.png) | WS01 given a static IP on the lab network |
| 03 | ![](screenshots/03-ws01-domain-joined.png) | WS01 joined to `apexdefense.local` |
| 04 | ![](screenshots/04-ws01-domain-login-jcarter(APEXDEFENSE).png) | Domain login as jcarter |
| 05 | ![](screenshots/05-ws01-domain-membership-login.png) | Domain membership confirmed |
| 06a-06b | ![](screenshots/06a-ws01-appears-in-ad-computers.png) | WS01 listed in Active Directory Users and Computers |
| 07 | ![](screenshots/07-ws01-moved-to-it-ou.png) | WS01 moved into the IT OU |
| 08 | ![](screenshots/08-ws01-firewall-disabled.png) | WS01 firewall disabled for the lab |
| 09 | ![](screenshots/09-sqlservice-spn-registered.png) | `sqlservice` SPN registered |
| 10 | ![](screenshots/10-kali-network-connectivity.png) | Kali on the lab network |
| 11 | ![](screenshots/11-responder-listening.png) | Responder listening on Kali |
| 12 | ![](screenshots/12-llmnr-poisoning-active.png) | LLMNR poisoning in progress |
| 13 | ![](screenshots/13-ntlmv2-hashes-captured.png) | NTLMv2 hashes captured |
| 14 | ![](screenshots/14-hashcat-exhausted.png) | Initial Hashcat run against the hash list |
| 15 | ![](screenshots/15-new-ntlmv2-hash-captured.png) | A further NTLMv2 hash captured |
| 16 | ![](screenshots/16-hashcat-password-cracked.png) | A captured hash cracked offline |
| 17 | ![](screenshots/17-kerberoast-hash-captured.png) | Kerberoast TGS hash captured |
| 18 | ![](screenshots/18-kerberoast-hashcat-exhausted.png) | Initial crack attempt on the Kerberoast hash |
| 19 | ![](screenshots/19-kerberoast-cracked.png) | Kerberoast hash cracked |
| 20 | ![](screenshots/20-secretsdump-hashes.png) | Domain hashes dumped with secretsdump |
| 21 | ![](screenshots/21-secretsdump-complete.png) | secretsdump run complete |
| 22 | ![](screenshots/22-pass-the-hash-shell.png) | Shell obtained on WS01 via Pass-the-Hash |
| 23 | ![](screenshots/23-pass-the-hash-whoami.png) | `whoami` confirming SYSTEM access |
| 24 | ![](screenshots/24-pass-the-hash-ipconfig.png) | `ipconfig` confirming the session is on WS01 |
| 25 | ![](screenshots/25-bloodhound-collection-complete.png) | BloodHound data collection complete |
| 26 | ![](screenshots/26-bloodhound-domain-analysis.png) | BloodHound domain analysis showing an escalation path |

---

## Skills Demonstrated

LLMNR/NBT-NS poisoning and hash capture, offline hash cracking, Kerberoasting (SPN enumeration and TGS extraction), domain credential dumping via DRSUAPI, Pass-the-Hash lateral movement, Active Directory enumeration and attack-path analysis with BloodHound, MITRE ATT&CK technique mapping, Windows Event ID detection mapping, multi-VM troubleshooting (clock sync, hypervisor settings, DNS), and attack-chain documentation for both technical and executive audiences.

---

## Repository Layout

```
.
├── README.md
├── docs/
│   └── active-directory-phase-2-documentation.pdf
└── screenshots/            # 28 numbered screenshots, 00 to 26 (06 split into a/b)
```

---

## Notes

- All attacks ran against a personal, isolated lab environment. No live systems were targeted.
- `apexdefense.local` is a fictional domain built for this project.
- Some lab accounts used deliberately weak passwords so a crack could be demonstrated. Those are documented in the PDF, not in this README, since this repo is public.
- BloodHound's GUI wasn't used because Neo4j wasn't installed on the isolated lab network; the collected JSON was analyzed directly instead, which demonstrates the same enumeration.
- A Phase 1 snapshot was taken before Phase 2 began, as a rollback point.

---

## Related Projects

- [Active Directory Home Lab: Phase 1, Environment Build & IT Administration](https://github.com/MrBSykes/active-directory-home-lab-phase-1-environment-building-and-IT-administartion)
- [Security Onion Home Lab: Detection & Incident Response](https://github.com/MrBSykes/security-onion-home-lab-detection-incident-response)

---

*Bryan Sykes | Home Lab Portfolio | 2026*
