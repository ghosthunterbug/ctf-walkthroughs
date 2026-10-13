# ctf-walkthroughs

CTF walkthroughs & pentest methodology — TryHackMe rooms.

## Methodology

Each room follows the same end-to-end process:

1. **Recon** — port scan, service enumeration, web crawling.
2. **Enumeration** — directory brute-force, source review, API discovery.
3. **Exploitation** — identify the vulnerability, craft the payload.
4. **Privilege escalation** — lateral movement to user/root.
5. **Writeup** — methodology + commands (no raw flags, no internal IPs).

## Completed Rooms (22)

### Writeups available

| Room | Difficulty | Focus |
|---|---|---|
| [Bounty Hacker](tryhackme/bounty-hacker) | Easy | Boot2Root — FTP anon, hydra, sudo tar privesc |
| [OWASP Juice Shop](tryhackme/owaspjuiceshop) | Easy | Web / OWASP Top 10 — SQLi, broken auth, IDOR, XSS |
| [Poster](tryhackme/poster) | Easy | SQL / Boot2Root — PostgreSQL RCE |
| [CyberHeroes](tryhackme/cyberheroes) | Easy | Authentication Bypass (client-side auth) |
| [Room 404](tryhackme/room404) | Easy | Web enum, exposed `.git` |
| [Overpass](tryhackme/overpass) | Easy | Cookie forgery, SSH key cracking, cron + hosts hijack |

### ![](https://img.shields.io/badge/Completed-blue) (no writeup)

| Room | Difficulty |
|---|---|
| Mr Robot CTF | Medium |
| Pickle Rick | Easy |
| Vulnversity | Easy |
| Anonymous Playground | Hard |
| Internal | Hard |
| The Server From Hell | Medium |
| VulnNet: dotpy | Medium |
| Crocc Crew | Insane |
| Oh My WebServer | Medium |
| Bookstore | Medium |
| GLITCH | Easy |
| CyberLens | Easy |
| W1seGuy | Easy |
| Snowy ARMageddon | Insane |
| U.A. High School | Easy |
| Corp Website | Medium |

## Tooling

`nmap` · `ffuf` · `gobuster` · `sqlmap` · `hydra` · `nikto` · `responder` · `bloodhound` · Burp Suite · Metasploit · Python · Bash
