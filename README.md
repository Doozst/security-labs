Enterprise IT Troubleshooting Lab

A VirtualBox lab that simulates a small enterprise (Active Directory, Windows client, Linux, Kali, Wazuh SIEM). I injected faults on purpose, diagnosed them with real tools (Wireshark, tcpdump, journalctl, PowerShell), and documented each one as a ticket in osTicket.

The goal is to show diagnostic method, not just installs: symptom, evidence, root cause, fix, proof.

Topology
VM	OS	Role	IP
WIN-DC01	| Windows Server 2022	Domain controller + DNS, labnoron.local	| 192.168.56.10
WIN-CLIENT01	| Windows 10/11	Domain-joined client	| 192.168.56.50
ticket-01	Ubuntu | Server	osTicket, Apache/MariaDB/PHP; acts as router in Session 8	| 192.168.56.20 / 192.168.57.3
Musashi (Kali)	Kali Linux	| Attacker/diagnostic client	| 192.168.56.105 / 192.168.57.4
Wazuh server	| Amazon Linux	SIEM	192.168.56.30

Host: Lenovo Y520, 15 GB RAM. Not every VM can run at once, so sessions were sequenced.

Method

Every fault follows the same ticket format: Symptom → Investigation → Root cause → Resolution → Verification → Lesson. A snapshot is taken before each fault injection. Tickets were written after the session, so the osTicket timestamps are not resolution times; times shown are marked approximate.

Ticket index
Session	Area	Fault	Root cause	Status	Detail
4	AD / Windows	Domain join failure	Client never domain-joined	Resolved	index only
4	AD / Windows	Account lockout	Policy configured at domain root	Resolved	index only
4	AD / GPO	GPO not applying	Authenticated Users removed from Security Filtering	Resolved	index only
4	DNS	Client could not resolve domain	Client resolver set to 8.8.8.8, not the DC	Resolved	index only
5	Linux	Apache won't start	Port 80 held by a Python http.server	Resolved	index only
5	Linux	403 on web root	/var/www/html at mode 000	Resolved	index only
5	Linux	Disk full	Filler file on a 200 MB loop volume	Resolved	index only
6	DNS	Multi-homed DC registering unreachable IP	NAT adapter registered 10.0.3.15	Resolved	index only
6	DNS	nslookup timeouts	DNS Server service stopped	Resolved	S06-T02
6	DNS	NXDOMAIN	A record deleted	Resolved	index only
6	DNS	Resolves, but connection fails	Round-robin A record with a dead target	Resolved	S06-T04
7	VPN	WireGuard tunnel down	Wrong peer public key (twice), VM clock desync, iptables DROP on UDP/51820	Resolved	index only
8	Routing	Cross-segment connectivity broken (4 faults)	ip_forward=0, missing route, bad /30 mask, missing return route	Resolved	S08-T04
9	Wazuh	Network / manager / agent faults	Lost IP after VirtualBox change, manager daemons down, install error	Resolved	index only
9	Wazuh	Detect brute-force logons	Rules 60122, then 60204	Resolved	S09-T06
9	Wazuh	No FIM alert for hosts file	Undetermined	Open	S09-T07
Featured case studies
DNS: resolves fine, connection still fails
Routing: asymmetric path from a missing return route
Wazuh: detecting brute-force logons
Open issue

The hosts-file FIM alert never fired. It is documented honestly as open: S09-T07.

Lessons
Silence and a negative answer are different failures (timeout vs NXDOMAIN).
A resolved name is not a reachable host; check the layer below (ARP, route).
Capture on the middle device (tcpdump on the router) to see asymmetric paths.
Verify with before/after evidence, not with "it works now."
