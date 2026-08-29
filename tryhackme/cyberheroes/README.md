# CyberHeroes — TryHackMe

- **Difficulty** : Easy
- **Category** : Web — Authentication Bypass (OWASP A07)
- **Time** : ~15 min

## Summary
A static site that invites you to "find the vuln on our login page and login to join us". The vulnerability is a fully client-side authentication: the credentials and the verification logic both live in the browser-side JavaScript of `login.html` — there is no server-side validation.

## Recon
```bash
# List linked pages
curl -s http://MACHINE_IP/ | grep -oE 'href="[^"]*"'

# Inspect the login page and its scripts
curl -s http://MACHINE_IP/login.html
```

## Vulnerability
Open `login.html` and read the inline `<script>`. The `authenticate()` function:

- compares the username against a hardcoded value;
- compares the password against a hardcoded string that is reversed at runtime via `[...str].reverse().join('')`;
- on success, fires an `XMLHttpRequest` to a hidden `.txt` path and injects the response (the flag) into the page.

Since the check happens entirely in the browser, the "login" is bypassable just by reading the source — the credentials are never verified server-side.

## Flag
Flag kept private per TryHackMe rules — format `THM{...}`.

## Takeaways
- Always read inline and external JS on login pages: hardcoded credentials and client-side checks are the classic A07 failure.
- "Authentication bypass" is not always SQLi — client-side-only auth is just as common and much faster to exploit.
- A single `&` (bitwise AND) between two boolean comparisons still behaves as a logical AND.
