## Objective
Turn ticket-01 into a router between two lab segments and diagnose routing faults with packet 
capture on the router. Topology: Kali (192.168.57.4/24) → ticket-01 (enp0s9 192.168.57.3 / 
enp0s3 192.168.56.20) → WIN-DC01 (192.168.56.10). Kali's NAT adapter was kept down so it could 
not mask routing faults. Capture method on the router: 
`sudo tcpdump -ni any icmp and host 192.168.56.10`.

## Ticket 1 — Forwarding Disabled on the Router
**Symptom:** Ping from Kali to WIN-DC01 hung with 100% loss. Ping to the router's own leg 
(192.168.57.3) worked.

**Root cause:** `net.ipv4.ip_forward = 0` on ticket-01. Packets arrived on enp0s9 but were not 
forwarded to enp0s3.

**Resolution:** `sudo sysctl -w net.ipv4.ip_forward=1`

**Verification:** Ping succeeded. tcpdump showed the full four-line pattern per ping: request 
in on enp0s9, request out on enp0s3, reply in on enp0s3, reply out on enp0s9. Reply TTL was 127 
(Windows 128 minus one router hop).

**Lessons:**
- A silent timeout with packets arriving on the router but not leaving makes the router the break point.
- Early in the lab, Kali's pings "worked" with no route to the 56.x network. Traceroute hop 1 was 
  10.0.3.2, the VirtualBox NAT gateway, so traffic bypassed the router entirely. A working ping 
  does not prove the intended path. Check `traceroute` and `ip route get`.
- `sysctl -w` does not survive a reboot. The persistent setting belongs in 
  `/etc/sysctl.d/99-lab-forward.conf`.

## Ticket 2 — Route Removed on the Sender
**Symptom:** Ping to 192.168.56.10 failed immediately with "Network is unreachable". tcpdump on 
the router captured 0 packets.

**Root cause:** Kali had no route to 192.168.56.0/24 and no default route. `ip route get` returned 
`RTNETLINK answers: Network is unreachable`, and `ip route` showed only the connected 
192.168.57.0/24.

**Resolution:** `sudo ip route add 192.168.56.0/24 via 192.168.57.3 dev eth0`

**Verification:** `ip route get 192.168.56.10` returned `via 192.168.57.3 dev eth0`. Ping 0% loss, TTL 127.

**Lessons:**
- An instant local error plus zero packets on the router means the failure is on the sender's side.
- Triage rule: an instant error means a local routing problem; a silent timeout means something 
  further along the path.
- The NAT adapter had to be down for this fault to show. With it up, the default route via 
  10.0.3.2 would have masked the missing route.

## Ticket 3 — Wrong Subnet Mask (/30) Makes the Gateway Unusable
**Symptom:** Adding the route via 192.168.57.3 failed with `Error: Nexthop has invalid gateway`. 
`ip -br a` showed `eth0 192.168.57.4/30`.

**Root cause:** A /30 puts Kali in block 192.168.57.4–7. The router at .3 sits in the neighbouring 
block (.0–.3), so the gateway is off-link and Linux refuses it as a next hop.

**Resolution:**

sudo ip addr flush dev eth0
sudo ip addr add 192.168.57.4/24 dev eth0
sudo ip route add 192.168.56.0/24 via 192.168.57.3 dev eth0


**Verification:** `ip -br a` showed only 192.168.57.4/24, `ip route get` returned the correct 
gateway, and ping showed 0% loss with TTL 127.

**Lessons:**
- A host decides "local or via gateway" from its own IP plus mask. A gateway outside that 
  network cannot be used.
- When a gateway is rejected, check the mask (`ip -br a`) before the route table.
- The first injection attempt failed: after `ip addr del`, NetworkManager re-added the DHCP /24, 
  so the fault did not exist. The fix was `nmcli device set eth0 managed no` followed by the flush. 
  Always verify the fault state before diagnosing.
- A stray `192.168.56.4/30` address appeared on eth0 during that check. Its origin is unknown, 
  and the flush removed it.
- On a /30, the network (.0) and broadcast (.3) addresses cannot be host or gateway addresses.

## Ticket 4 — Missing Return Route on the Destination
**Symptom:** Ping from Kali timed out. tcpdump showed the request in on enp0s9 and out on 
enp0s3 for every ping, but no reply lines.

**Root cause:** WIN-DC01 had no route to 192.168.57.0/24 (deleted with `route delete`). The DC 
is multi-homed, so it sent replies out its NAT adapter's default gateway (10.0.3.2), which 
cannot reach 57.x. `Find-NetRoute -RemoteIPAddress 192.168.57.4` showed only the NAT address 
10.0.3.15. This is asymmetric routing: the request came via the router and the reply left via 
a different path.

**Resolution:** `route add 192.168.57.0 mask 255.255.255.0 192.168.56.20 -p` on WIN-DC01 (persistent).

**Verification:** `Find-NetRoute` selected the 56.x adapter with next hop 192.168.56.20. Ping 
returned 3/3 replies, and tcpdump showed the full four-line pattern again.

**Lessons:**
- A request that leaves the router with no reply points at the destination's return path, not the router.
- Routing must work in both directions. Check the destination's route table, especially on 
  multi-homed hosts whose default gateway is on another adapter.
- The router's own address never appears in the IP headers of forwarded packets. It only shows 
  up in traceroute.

## Ticket 5 — Router Interface on a DHCP Lease
**Symptom:** The new router leg enp0s9 came up as 192.168.57.3/24 with `scope global dynamic` and 
`valid_lft 560sec`. No netplan entry existed for it.

**Root cause:** The address came from the VirtualBox host-only DHCP server. If the lease changed, 
Kali's gateway (192.168.57.3) would silently break.

**Resolution:** Created a separate file `/etc/netplan/60-lab-segment.yaml` (chmod 600) with 
`dhcp4: false`, the static address 192.168.57.3/24, and no gateway. Applied with 
`sudo netplan try`. The existing enp0s3 config was left untouched.

**Verification:** `ip -4 addr show enp0s9` shows no `dynamic` flag and `valid_lft forever`. 
`ip route` shows connected routes for both segments, and the default route is still via 10.0.3.2 
on enp0s8.

**Lessons:**
- Infrastructure that other hosts use as a gateway must have a static address.
- Put no gateway on the second leg, because a router with two default routes causes its own failures.
- `netplan try` rolls back automatically if a change locks you out.

## Comparison: Where Each Fault Breaks
| Fault | Where it breaks | tcpdump on router |
|---|---|---|
| Forwarding off | Router | `In` only |
| Kali route removed | Sender | 0 packets |
| Kali /30 mask | Sender | 0 packets |
| DC return route missing | Destination | `In` + `Out`, no reply |

## Overall Lesson
The same complaint, "I can't reach it", pointed to three different locations in the path: the 
sender, the router, and the destination. Counting what the router sees (nothing, inbound only, 
or outbound with no reply) tells you which one before you touch any of them.
