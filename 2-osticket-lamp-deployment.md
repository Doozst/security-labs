## Objective
Deploy osTicket on Ubuntu LAMP stack (ticket-01), from server install through working admin portal.

## Recon / Methodology
Full build from scratch: Ubuntu Server install → Apache/PHP/MariaDB → osTicket download → 
web installer → working portal.

## Obstacles & Resolutions

**DNS mirror failure during install** — installer couldn't resolve tn.archive.ubuntu.com mid-setup.
Skipped the failed mirror check, fixed full connectivity post-install.

**Package naming mismatches** — `libapache2-mob-php` (typo) → correct is `libapache2-mod-php`.
`php-imap` unavailable in default repos, dropped (not required for core osTicket).

**MariaDB tooling renamed** — `mysql_secure_installation` not found. Traced via 
`dpkg -L mariadb-client-core | grep bin` — MariaDB has migrated to `mariadb-secure-installation` 
as part of an ongoing mysql_* → mariadb_* naming transition.

**SQL syntax typos** — `FRANT ALL PRIVILEGES` and `FLUSH PRIVILEGS` both failed silently, leaving 
a user created with zero actual privileges until caught and corrected.

**Wrong GitHub release URL** — `wget` targeted a typo'd repo path, 404'd. Fixed by querying 
GitHub's API directly (`curl -s https://api.github.com/repos/osTicket/osTicket/releases/latest 
| grep browser_download_url`) instead of guessing the asset path.

**Path typo on unzip** — `/var/uuu/html/osticket` instead of `/var/www/html/osticket`, caught 
before deeper failure.

**MariaDB app-account auth failure (main incident)** — Apache's PHP error log showed 
`Access denied for user 'osticket_user'@'localhost'` during the web installer's DB connection 
step, despite the account existing in `mysql.user`. Root cause: unrecoverable password on the 
account. `ALTER USER` attempts failed (error 1396). Resolved by dropping and recreating the 
account cleanly (`DROP USER` → `CREATE USER` → `GRANT` → `FLUSH PRIVILEGES`), then validating 
via direct CLI login (`mariadb -u osticket_user -p osticket`) before resubmitting the installer.

**Unexpected GUI on headless server** — a desktop environment appeared on ticket-01 unexpectedly, 
inconsistent with a Server-edition install, no terminal easily accessible. Not fully root-caused — 
worked around by restoring the `ticket-01-osTicket-Installed-Clean` snapshot rather than debugging 
forward.

**Keyboard input not registering (Windows client VM)** — Ctrl+Alt+Del didn't reach the guest OS 
because the host intercepted it. Fixed via VirtualBox's Input → Keyboard → Insert Ctrl-Alt-Del 
menu command.

## Verification
Logged into osTicket admin panel — staff dashboard loads, Tickets tab functional, queue reachable.

## Lesson
Most failures here weren't conceptual — they were typos, renamed tools, and silent SQL failures 
that looked like success until checked. The MariaDB auth incident was the one true root-cause dig: 
matching an app-layer error (PHP/Apache log) back to a database-layer permission problem, and 
choosing to recreate the account cleanly rather than fight an unrecoverable password. 
Systematic service-by-service isolation (checking each layer instead of guessing) is what closed 
it — and what a snapshot revert doesn't always solve if the real cause isn't file-level.
