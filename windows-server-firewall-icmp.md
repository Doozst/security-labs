## Objective
Verify connectivity between Windows Server VM and lab peers after static IP config.

## Recon / Methodology
Confirmed static IP correctly assigned. Peers unreachable via ICMP despite matching subnet.

## Obstacle
Windows Firewall default-denies inbound ICMPv4 Echo Request — silent failure, no error, just no response.

## Resolution
Located and enabled the relevant NetFirewallRule via PowerShell (Enable-NetFirewallRule -DisplayName "File and Printer Sharing (Echo Request - ICMPv4-In)").

## Verification
Bidirectional ping confirmed. Ran dcdiag post-domain-promotion — healthy.

## Lesson
Windows Firewall's default inbound posture is a common false-positive for "network misconfigured" — check firewall rules before re-checking IP/subnet config.
