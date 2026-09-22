## Objective
Run three fault-injection scenarios on a Kali Linux VM (kali-lab) running Apache: a port conflict, 
a permissions/403 failure, and a disk-full condition. Diagnose, resolve, verify, and ticket each.

## Scenario 1 — Apache Fails to Start (Port Conflict)
**Environment:** kali-lab, Apache 2.4.65 (Debian)
**Symptom:** `apache2` fails to start — `systemctl` reports `Active: failed (Result: exit-code)`, 
exit status 1. Site unreachable.

**Diagnosis:**
- `systemctl status apache2` → failed, status=1/FAILURE
- `journalctl -xeu apache2 --no-pager | tail -30` → `(98)Address already in use` binding to 
  `0.0.0.0:80`, no listening sockets available (the AH00558 ServerName warning was unrelated noise)
- `sudo apachectl configtest` → "Syntax OK" — ruled out config as the cause
- `sudo ss -tlnp 'sport = :80'` → port already held by another process; `curl localhost` returned 
  a Python directory listing, confirming a non-Apache process owned the port

**Root cause:** A Python `http.server` instance was already bound to port 80, blocking Apache 
from starting.

**Resolution:** Identified the PID via `sudo ss -tlnp 'sport = :80'`, killed the process, 
restarted Apache.

**Verification:** Port confirmed free before restart; `curl -I localhost` returned 
`HTTP/1.1 200 OK`, `Server: Apache/2.4.65 (Debian)`. Resolved in ~10 minutes.

**Lesson:** `configtest` passing only proves the config is valid — it says nothing about whether 
the port is actually free. A `200 OK` on its own doesn't confirm Apache is serving; the `Server:` 
header is what proves it.

## Scenario 2 — Web Root Returns 403 (Permissions)
**Environment:** kali-lab, Apache 2.4.65
**Symptom:** `curl -I localhost` first returned connection refused; after starting Apache, it 
returned `403 Forbidden`.

**Diagnosis:**
- Connection refused → nothing listening
- `namei -l /var/www/html/index.html` → mode `000` on the web root, blocking traversal
- `systemctl is-active` / `is-enabled` → inactive, disabled; `uptime -p` confirmed a recent reboot 
  explained the service being down
- After starting Apache: `403 Forbidden`, and the error log (`AH00035`) confirmed access denied 
  due to missing search permissions on a path component

**Root cause (two-part):** Apache was down after a VM reboot (never enabled at boot), and 
`/var/www/html` had mode `000`, blocking `www-data` from traversing the directory.

**Resolution:** Started Apache, then `sudo chmod 755 /var/www/html`.

**Verification:** `namei -l` showed `drwxr-xr-x`, no denial; `curl -I localhost` returned 
`200 OK` with the correct Server header.

**Lesson:** Connection refused means nothing's listening; 403 means it's listening but denying — 
reading which layer is failing is the actual diagnosis. The first symptom (service down) masked 
the second, real fault (permissions) until the service was restarted — fixing the obvious problem 
first can reveal a hidden one underneath.

## Scenario 3 — Disk Full
**Environment:** kali-lab, 200 MB ext4 volume loop-mounted at `/mnt/labdisk`
**Symptom:** Writes as a normal user failed with "no space left on device."

**Diagnosis:**
- `df -h /mnt/labdisk` → 100% used, 0 available
- `df -i /mnt/labdisk` → only 1% inodes used, ruling out inode exhaustion
- `sudo du -ah /mnt/labdisk | sort -rh | head` → single file `fill.bin` consuming 178 MB of the 
  182 MB volume
- Noted: writes via `sudo tee` succeeded, since root can use ext4's reserved blocks — a working 
  root session doesn't prove the volume is healthy for normal users/services

**Root cause:** Block exhaustion from a single large file.

**Resolution:** Removed the file (`sudo rm /mnt/labdisk/fill.bin`).

**Verification:** `df -h` showed 1% used, 168 MB available; a normal-user write/read succeeded.

**Lesson:** `df -h` separates "no space" from `df -i`'s "no inodes" — and `du` finds the actual 
culprit. Root's access to reserved blocks means an admin session working fine doesn't guarantee 
normal users or services are unaffected. On a production system, a file should be checked (age, 
owner, open handles via `lsof`) before deletion — this was skipped only because the file was 
lab-created.

## Overall Lesson
All three faults were the kind that don't announce their real cause up front: a passing 
`configtest`, a "service is down" symptom masking a permissions fault underneath it, and root's 
elevated access hiding a normal-user-facing problem. In each case, the fix came from checking the 
next layer down rather than stopping at the first plausible explanation.
