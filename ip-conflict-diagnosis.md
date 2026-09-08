## Objective
Inject a controlled client-side network fault (IP conflict) into the AD lab, diagnose it 
methodically, and document via ticket.

## Recon / Methodology
Before the deliberate fault could be trusted, had to fix real unplanned issues in the host-only 
network first: two separate host-only networks (vboxnet0/vboxnet1) both using 192.168.56.x — 
adding a host-only adapter creates a new isolated switch, not a join to an existing one. 
Standardized on vboxnet0. Client had no host-only adapter at all (NAT only) — added it. Client 
was falling back to APIPA (169.254.x.x) due to an expired DHCP lease with no working DHCP server 
behind it — moved to static addressing instead. Hit IP collisions during static config (first 
against the host's own IP, then targeted the wrong adapter — "Ethernet 2" was NAT, not host-only). 
Windows' modern network settings UI rejected valid subnet input in multiple formats — dropped to 
classic `ncpa.cpl` dialog instead.

## Obstacle
With the network actually stable, deliberately set the client to 192.168.56.10 — the DC's live 
address — to force a real conflict.

## Resolution
Diagnosed symptom-first: `(Duplicate)` tag in `ipconfig` → Windows Network Diagnostics alert → 
inconsistent ping loss (0% to .10, 25% to 8.8.8.8) → Event Viewer confirmation. Found Event ID 
4199 (Source: Tcpip) reporting the conflict against MAC 08-00-27-26-D4-51, cross-referenced and 
confirmed as WIN-DC01's NIC. Reassigned client to 192.168.56.50/24, DNS 192.168.56.10. Also had 
to disable inbound ICMP block on the client's Windows Firewall to allow DC→client ping.

## Verification
Clean ping both directions, confirmed no further Event ID 4199 entries in the System log. 
Logged as osTicket #491490 with full Symptom/Root Cause/Resolution/Verification structure.

## Lesson
Nearly every fault today traced back to an unverified assumption — "I put them on the same 
network," "they ping now," "I'm setting the host-only IP" — each wrong until checked with a 
command. Precision over familiar-sounding answers, same gap flagged in Security+ study, showing 
up here in the lab instead of on a quiz.
