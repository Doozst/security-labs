## Objective
Deploy Wazuh agents on the Windows endpoints (WIN-DC01, WIN-CLIENT01) of the AD lab, enroll them 
with the Wazuh manager, and confirm detection of a real event (repeated failed logons). Several 
infrastructure faults had to be fixed first.

## Ticket 1 — Kali (Musashi) Lost Its Host-Only IP
**Symptom:** `eth0` had no IPv4 address (only IPv6 link-local) and SSH to 192.168.56.105 was dead. 
`eth1` (NAT) was fine.

**Root cause:** While setting up the Wazuh VM, the `vboxnet0` host-only adapter was changed so the 
host itself held 192.168.56.105, Musashi's own address. Musashi's NetworkManager profile was also 
on DHCP (`ipv4.method=auto`), so it had nothing stable to fall back on.

**Resolution:** Set `vboxnet0` back to 192.168.56.1/24 in VirtualBox. On Musashi: 
`nmcli con mod eth0 ipv4.method manual ipv4.addresses 192.168.56.105/24 ipv4.never-default yes`, 
then `nmcli con up eth0`.

**Verification:** `ip -br a` shows 192.168.56.105/24 on `eth0`, and the default route stays on `eth1`.

**Lesson:** The .1 address on a vboxnet adapter belongs to the host. A VM takes its own address in 
the same /24, never .1 and never another VM's IP. Use `never-default` on a second NIC so it 
cannot take the default route.

## Ticket 2 — Wazuh Dashboard "API is down" (Error 3099)
**Symptom:** The health check failed with "3002 Request failed with status code 500" and "3099 Some 
Wazuh daemons are not ready yet in node node01 (wazuh-modulesd->failed)". The page loaded but had 
no API connection.

**Root cause (probable, not proven):** The manager's core daemons (analysisd, remoted, syscheckd, 
logcollector, modulesd, monitord) were not running, with stale PID entries in the log. That 
pattern points to a slow start or an unclean previous shutdown.

**Resolution:** `sudo systemctl restart wazuh-manager`, then waited for startup.

**Verification:** `wazuh-control status` showed the daemons running, `ossec.log` showed modulesd, 
the vulnerability scanner, syscollector, FIM and the indexer connectors initializing, and the 
dashboard API check went green.

**Lesson:** Read the daemon list before restarting anything. The manager needs a few minutes 
after boot, so judge it after 3 minutes. Shut Wazuh down with `sudo shutdown now`, not a hard 
power-off. The same failure returned on Oct 18 (agent error 1208, ports 1514/1515 closed) and 
cleared once the manager was healthy.

## Ticket 3 — Pin the Wazuh Server to a Static IP
**Symptom:** The server (192.168.56.103) held a DHCP lease from `vboxnet0`. Agents enroll against 
the manager's address, so a changed lease would silently break every agent.

**Root cause:** `/etc/systemd/network/20-eth0.network` matches `Type=ether` with `DHCP=ipv4`, so 
every Ethernet interface, including the host-only `eth1`, was on DHCP.

**Resolution:** Added a more specific file, `/etc/systemd/network/10-eth1.network`, matching 
`Name=eth1` with `Address=192.168.56.30/24`, `DHCP=no`, and no gateway. Applied with 
`sudo networkctl reload`.

**Verification:** The dashboard is reachable at https://192.168.56.30 and the manager daemons 
are still running.

**Lesson:** `systemd-networkd` uses the first matching file in lexical order, so a lower-numbered, 
narrower match wins. Do not give a lab VM a gateway on the host-only NIC when NAT already holds 
the default route.

## Ticket 4 — Capacity Planning: Two VMs Would Not Run Together
**Symptom:** With Wazuh and WIN-DC01 both set to 8000 MB, the second VM would not start (host: 
15 GB total, about 12 GB available).

**Root cause:** The allocations (16 GB across two VMs) exceeded available host RAM. The DC was 
over-allocated, since a lab DC runs on about 3 GB.

**Resolution:** Right-sized the VMs: Wazuh about 6-7 GB and 2 vCPU, WIN-DC01 3 GB, WIN-CLIENT01 
2 GB, and Kali off while Wazuh work runs.

**Verification:** Wazuh, the DC and the client all booted.

**Lesson:** Check the sum of allocations against `free -h` on the host before powering VMs on. Do 
not trim Wazuh much below 6 GB, because its indexer is memory-hungry.

## Ticket 5 — Wazuh Agent Install and Enrollment on WIN-CLIENT01
**Symptom:** The agent was not installed ("Cannot find any service with service name WazuhSvc"). 
Once the service existed, the agent log showed error 1208 (unable to connect to the enrollment 
service at 192.168.56.30:1515), and `Test-NetConnection` failed on both 1514 and 1515.

**Root causes (several, in order):**
1. A typo in the install command: manager address `92.168.56.30` and a missing space before 
   `WAZUH_AGENT_NAME`.
2. On retry, the `$` was lost from `$msi`, so `msiexec` received the literal text "msi" and `/q` 
   hid the failure.
3. The MSI had been downloaded under the LabUser profile, and the Administrator window had a 
   different temp folder.
4. The first download needed TLS 1.2 set explicitly.
5. After the install succeeded, the service was Stopped (fresh installs do not auto-start), and 
   ports 1515 and 1514 were closed because the manager daemons were down (see Ticket 2).

**Resolution:** Ran from the correct temp folder: 
`msiexec.exe /i wazuh-agent.msi /q /l*v wazuh-install.log WAZUH_MANAGER=192.168.56.30 WAZUH_AGENT_NAME=WIN-CLIENT01`, 
started `WazuhSvc`, and restarted it once the manager was healthy.

**Verification:** `Test-NetConnection 192.168.56.30 -Port 1515` returned True. WIN-CLIENT01 shows 
Active as agent 003 (192.168.56.50), and WIN-DC01 is agent 002 (192.168.56.10).

**Lesson:** The server address in the install command is the manager, not the endpoint being 
monitored. The manager identifies each agent by name plus source address. Always install with 
`/l*v` so a silent install cannot fail invisibly. Agent warnings 6950/1789 (trust relationship 
failed) appeared while WIN-DC01 was powered off. That is the likely cause, but the secure-channel 
check was not run.

## Ticket 6 — Detection of Repeated Failed Logons (Rules 60122 and 60204)
**Event:** Repeated failed interactive logons at WIN-CLIENT01 (agent 003) on 2026-10-19. This was 
a deliberate test, not an attack.

**Evidence (exported CSV, 11 events):**
- 10 alerts of rule 60122, "Logon Failure - Unknown user or bad password", level 5.
- 1 alert of rule 60204, "Multiple Windows Logon Failures", level 10, about 207 seconds after the 
  first failure, on the 8th failure. Three more failures followed without a second 60204.

**Why it fired:** Rule 60204 has frequency 8, timeframe 240 seconds, grouped by source address 
(`win.eventdata.ipAddress`). Event fields showed ipAddress 127.0.0.1, logonType 2 (interactive 
console), status `0xc000006d` with subStatus `0xc000006a` (account exists, wrong password).

**Mapped controls:** PCI DSS 10.2.4, 10.2.5, 11.4. MITRE Brute Force (Credential Access) for 60204.

**Verification:** Dashboard Threat Hunting, Events, agent 003, with the CSV exported.

**Lesson:** One failure is low-level noise (60122, level 5) and the burst is the signal (60204, 
level 10). Because the rule groups by source address, an attack spread across many addresses 
would not trigger it, which is a detection gap. The "Account Access Removal" MITRE tag on 60122 
does not describe a failed login accurately.

## Overall Lesson
Most of this session's faults were upstream of Wazuh itself: addressing, memory, a stopped 
service, a mistyped command. The agents only enrolled once each layer was verified separately 
(network, manager daemons, ports, agent service) instead of retrying the install. A silent 
installer and a healthy-looking dashboard both hid failures until logs and port tests were checked.
