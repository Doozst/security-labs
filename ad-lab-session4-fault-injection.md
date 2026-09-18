## Objective
Run four fault-injection/diagnostic scenarios against the AD lab (WIN-DC01, WIN-CLIENT01): 
domain join, account lockout, GPO Security Filtering misapplication, and a DNS resolver fault — 
diagnose and resolve each via standard tooling, ticket each one.

## Scenario 1 — Domain Join Failure
**Obstacle:** Client had never actually been domain-joined.
**Resolution:** Verified DNS pointed at the DC, then joined via `sysdm.cpl`. Cleared up a 
SamAccountName vs. display name mix-up along the way.
**Verification:** Client confirmed domain-joined and authenticating against WIN-DC01.

## Scenario 2 — Account Lockout
**Obstacle:** Needed to simulate and diagnose an account lockout under a real policy.
**Resolution:** Configured lockout thresholds in Default Domain Policy, triggered the lockout, 
unlocked manually via ADUC.
**Verification:** Account unlocked and authenticating again.
**Lesson:** Lockout policy has to be set at the domain-root GPO — OU-level won't apply it correctly.

## Scenario 3 — GPO Misapplication via Security Filtering
**Obstacle:** Created "Lab-Restriction-Test" GPO linked to the Lab OU, deliberately swapped 
"Domain Admins" into Security Filtering in place of "Authenticated Users" to break the GPO's 
application to the intended target.
**Resolution:** Tested the misapplication, confirmed the GPO wasn't reaching the right accounts, 
then reverted Security Filtering back to "Authenticated Users."
**Verification:** GPO confirmed applying correctly post-revert.

## Scenario 4 — DNS Resolver Fault
**Obstacle:** Client's DNS resolver was pointed at 8.8.8.8 instead of the DC.
**Recon:** Used `nslookup` to isolate the fault — explicit query to 192.168.56.10 resolved 
`labnoron.local` correctly, proving the DC's DNS zone was healthy the whole time. The "unknown 
server" message was a red herring caused by a missing reverse lookup zone, not a real fault.
**Resolution:** Reassigned client DNS to 192.168.56.10 via `ncpa.cpl`.
**Verification:** Confirmed clean with `dcdiag` and `repadmin /replsummary` (empty result — 
correct for a single-DC environment).

## Lesson
Across all four scenarios, the pattern that mattered most was not assuming a symptom points to 
its most obvious cause — the DNS "unknown server" message looked like a broken zone but was 
actually a missing reverse lookup zone; the real fault was a resolver misconfiguration one layer 
away. Isolating each layer independently (DNS zone health vs. client resolver setting) before 
acting is what kept the fix accurate instead of just plausible.
