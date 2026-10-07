## Objective
Diagnose four distinct DNS failure modes in the AD lab (Kali as client, WIN-DC01 as DNS server, 
192.168.56.10), using Wireshark packet capture throughout to confirm root cause at the wire level, 
not just by tool output.

## Ticket 1 — Stale DNS Record from Multi-Homed DC
**Symptom:** `WIN-DC01.labnoron.local` resolved to two A records — `192.168.56.10` (reachable) 
and `10.0.3.15` (VirtualBox NAT adapter, unreachable from the lab network).

**Root cause:** The DC's NAT adapter had "Register this connection's address in DNS" enabled, 
publishing an address no lab client could ever reach.

**Resolution:** Disabled DNS registration on the NAT adapter, ran `ipconfig /registerdns`, 
deleted the stale `10.0.3.15` A record in DNS Manager.

**Verification:** Packet capture before showed two A records in the response; after showed a 
single correct A record (`192.168.56.10`), response time <1ms.

**Lesson:** A multi-homed server registers every adapter's address in DNS by default — only the 
client-facing adapter should register. A fast, protocol-correct DNS answer can still point to the 
wrong, unreachable value.

## Ticket 2 — DNS Service Stopped (Silent Failure)
**Symptom:** `nslookup` from Kali timed out three times (~5s apart), then returned "no servers 
could be reached."

**Root cause:** The DNS Server service was stopped on WIN-DC01 (`Stop-Service DNS`).

**Evidence chain:** Wire capture showed three identical queries (same transaction ID), zero 
responses, zero ICMP — pure silence. `Get-Service DNS` confirmed `Stopped`. `netstat` showed 
nothing bound to port 53. UDP and TCP `nmap` scans against port 53 were both inconclusive — a 
generic port scanner can't distinguish a silently-dropped probe from a genuinely closed port, 
logged as a tool limitation rather than evidence.

**Resolution:** `Start-Service DNS`.

**Verification:** Post-fix query/response pair ~0.5ms apart.

**Lesson:** A timeout doesn't mean the host is down — ARP and ping already proved the host was 
alive, isolating the fault to one specific service. Protocol-specific tools (`nslookup`, `dig`) 
are the ground truth for confirming an application-layer service's health, not a generic port scan.

## Ticket 3 — Record Deleted (Explicit NXDOMAIN)
**Symptom:** `nslookup` for a test record returned instantly with NXDOMAIN.

**Root cause:** `Remove-DnsServerResourceRecord` deleted the A record while the DNS service 
itself remained fully operational.

**Evidence chain:** Baseline capture showed the record resolving correctly. Post-deletion showed 
an instant response (no 15s wait) with terminal NXDOMAIN. Decoded the DNS flags field (`0x8583`) 
at the packet level — AA=1 (authoritative), RCODE=3 (Name Error) — confirming the failure at the 
protocol layer, not just the terminal string.

**Resolution:** `Add-DnsServerResourceRecordA` re-created the record.

**Verification:** Clean re-resolution, no NXDOMAIN.

**Lesson:** The same user complaint ("can't reach it") can present as silence (Ticket 2, an 
infrastructure problem) or as an instant explicit no (a data problem). NXDOMAIN specifically means 
the server is authoritative and checked — ruling out a service or reachability issue immediately.

## Ticket 4 — Round-Robin DNS with a Dead Target
**Symptom:** Same hostname intermittently usable — not reproducible from a single test.

**Root cause:** Two A records existed for the same name — one live (`.50`), one pointing to an 
unassigned address (`.199`). DNS round-robin rotated which address appeared first in the response.

**Evidence chain:** `Get-DnsServerResourceRecord` confirmed two A records under one name. A 
10-run query loop showed every response contained *both* addresses — the rotation was in ordering, 
not in which addresses were returned. `ping` to the dead address returned "Destination Host 
Unreachable," sourced from Kali itself, confirming a local ARP resolution failure (no host 
answered) rather than any response from a real device. A related ICMP "Port unreachable" signature 
was also observed during testing — a third failure signature alongside silence and NXDOMAIN — 
logged as an artifact for future investigation rather than isolated in this session.

**Resolution:** Removed the stale `.199` record by `-RecordData`, kept `.50`.

**Verification:** Repeated queries confirmed only the live address returned going forward.

**Lesson:** DNS resolution succeeding is not proof of reachability — the failure can sit one 
layer up, at ARP/connectivity, with DNS behaving exactly as configured. A single successful test 
is insufficient evidence for an intermittent fault; repeated sampling is what actually detects 
the pattern.

## Overall Lesson
The same user-facing complaint — "I can't reach it" — produced three genuinely different wire 
signatures across this session: silence (infrastructure layer), explicit NXDOMAIN (data layer), 
and successful resolution with a downstream connection failure (a layer past DNS entirely). 
Timing, response code, and what happens *after* resolution succeeds are the actual diagnostic 
sequence — not just whether `nslookup` returns something.
