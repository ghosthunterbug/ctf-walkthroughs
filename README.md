# ctf-walkthroughs

CTF walkthroughs & pentest methodology - TryHackMe rooms.

## Notre méthode

Binôme d'agents qui bossent en parallèle :

- **GhostBoss** — recon & navigation (Chromium/CDP), énumération web, sélection des rooms, payloads.
- **GhostHunter** — scans (nmap/ffuf), exploitation, flags.

### Process d'une room
1. Choisir la room (difficulté / thème OWASP).
2. Connecter le VPN TryHackMe (`tun0` up), démarrer la machine, récupérer l'IP cible.
3. Recon en parallèle : GhostBoss côté browser/web, GhostHunter côté scans.
4. Exploitation → flag.
5. Capitalisation : writeup ici (méthodo, sans flag brut) + notes privées dans le vault Obsidian local.

## Rooms

- [CyberHeroes](tryhackme/cyberheroes) — Authentication Bypass (auth côté client, OWASP A07)
- [Room 404](tryhackme/room404)
