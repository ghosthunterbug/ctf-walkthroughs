# ctf-walkthroughs

CTF walkthroughs & pentest methodology — TryHackMe rooms.

## Methodology

Each room follows the same end-to-end process:

1. **Recon** — port scan, service enumeration, web crawling.
2. **Enumeration** — directory brute-force, source review, API discovery.
3. **Exploitation** — identify the vulnerability, craft the payload.
4. **Privilege escalation** — lateral movement to user/root.
5. **Writeup** — methodology + commands (no raw flags, no internal IPs).

## Rooms

- [Bounty Hacker](tryhackme/bounty-hacker) — Boot2Root (FTP anon, hydra, sudo tar privesc)
- [OWASP Juice Shop](tryhackme/owaspjuiceshop) — Web / OWASP Top 10 (SQLi, broken auth, IDOR, XSS)
- [Poster](tryhackme/poster) — SQL / Boot2Root (PostgreSQL RCE)
- [CyberHeroes](tryhackme/cyberheroes) — Authentication Bypass (client-side auth, OWASP A07)
- [Room 404](tryhackme/room404) — Web enum, exposed `.git`

## Tooling

`nmap` · `ffuf` · `gobuster` · `sqlmap` · `hydra` · `nikto` · Burp Suite · Python · Bash
