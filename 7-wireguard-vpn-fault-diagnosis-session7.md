## Objective
Build a WireGuard VPN tunnel between Kali (192.168.56.105 / 10.10.10.1) and ticket-01 
(192.168.56.20 / 10.10.10.2), break it in controlled ways, and diagnose each failure with 
evidence at the packet, firewall, and application layers.

## Ticket 1 — Handshake Failure: Public Key Mismatch
**Symptom:** No handshake forming. `wg show` had no `latest handshake` field. `tcpdump` confirmed 
UDP/51820 packets arriving at the destination, but no replies were sent.

**Root cause:** The `[Peer] PublicKey` in Kali's `wg0.conf` did not match ticket-01's actual 
public key, a transcription error during a manual key copy.

**Resolution:** Corrected the `PublicKey` value and reloaded with `wg-quick down/up`.

**Verification:** The fix alone was insufficient and the handshake was still absent (see Ticket 3).

**Lesson:** Never trust a manually copied base64 key without diffing it against the source 
machine's `cat publickey`. There is no checksum to catch a silent transcription error.

## Ticket 2 — VM Clock Desync Corrupting Command Output
**Symptom:** `ping` reported `invalid tv_usec`, `time of day goes back`, and garbage RTT values. 
Separately, `.zsh_history` was reported as corrupt on Kali.

**Root cause:** The guest clock was stale after a snapshot restore. No time sync was running 
(`VBoxService` not running, `NTP service: inactive`), so the clock jumped abruptly mid-session 
with nothing reconciling it gradually.

**Resolution:** `sudo timedatectl set-ntp true`, enabling continuous correction via 
`systemd-timesyncd`.

**Verification:** `timedatectl status` showed `System clock synchronized: yes`. The corrupted 
`.zsh_history` was recovered via `strings` extraction, dropping the unparseable lines.

**Lesson:** Run `timedatectl status` immediately after every snapshot restore, before any timed 
or logged work. A snapshot is untrusted for restore until that check is confirmed.

## Ticket 3 — Handshake Failure: Stale Public Key File (Recurrence)
**Symptom:** The handshake was still absent after Ticket 1's fix. The firewall was ruled out 
first (`iptables -S` on both VMs showed only default ACCEPT policies).

**Root cause:** ticket-01's private key had been regenerated after its `publickey` file was last 
written. The saved file was stale, so `wg show`'s live public key no longer matched `cat publickey`. 
The "corrected" key from Ticket 1 still pointed at a key that no longer existed.

**Resolution:** Re-derived the current public key live (`wg pubkey < privatekey`) and updated 
Kali's `[Peer]` block.

**Verification:** `wg show` returned `latest handshake: 2 seconds ago`. Bidirectional ping 
confirmed (10.10.10.1 ↔ 10.10.10.2, 0% loss).

**Lesson:** `wg show` is ground truth and a saved key file is not. Verify live interface state 
before trusting a stored file, especially after any key regeneration.

## Ticket 4 — Deliberate VPN Outage: iptables Block on UDP/51820
**Symptom:** The tunnel went down after a controlled test. Pings across the tunnel failed, the 
`received` counter on Kali stopped increasing, and the handshake was not renewed.

**Root cause:** A deliberately injected rule on ticket-01 dropped all inbound WireGuard traffic: 
`iptables -A INPUT -p udp --dport 51820 -j DROP`. ticket-01 never received Kali's handshake or 
keepalive packets, so it never replied.

**Diagnosis:**
- `iptables -L INPUT -n -v` showed the DROP rule matching real traffic (61 packets, 7260 bytes).
- A Wireshark capture on Kali (`udp.port == 51820`) showed one-way traffic. Kali kept sending, 
  and ticket-01's last reply was at packet 66.
- A `LOG` rule placed ahead of the DROP 
  (`-I INPUT 1 -p udp --dport 51820 -m limit --limit 5/min -j LOG --log-prefix "WG-DROP: "`) 
  recorded the drop events in the kernel log. `journalctl -k -f | grep WG-DROP` showed entries 
  from 12:49:51 to 12:53:40. The two `LEN=156` entries at the start were handshake-sized packets, 
  and the `LEN=100` entries after them were keepalives arriving about every 10s.

**Resolution:** Deleted both rules with `iptables -D` and enabled `wg-quick@wg0` on both VMs 
with `systemctl enable` so the tunnel survives reboots.

**Verification:** `iptables -S` on ticket-01 shows only default ACCEPT policies (LOG and DROP 
rules removed). `wg show` on Kali shows a live handshake (1m38s) and the received counter 
increasing (9.54 KiB), confirming ticket-01 is replying again. `wg-quick@wg0` is enabled on both VMs.

**Lessons:**
1. Confirm firewall state with the packet counter before interpreting any capture. A capture 
   only proves the moment it was taken.
2. Isolate layer by layer: packets arrive (tcpdump), then firewall (counters, LOG), then 
   application (`wg show`).
3. Wireshark cannot show a DROP event, only the resulting silence. A `LOG` rule placed before 
   the DROP records the event itself.
4. Make services persistent. An interface brought up by hand disappears on reboot.
5. Keep screenshots of evidence as you go. The first LOG output was lost and the test had to 
   be re-run.

## Overall Lesson
Three of the four faults hid behind a trusted assumption: a copied key, a saved file, a healthy 
clock. The pattern that closed each one was checking live state (`wg show`, `timedatectl`, 
packet counters) instead of trusting what was written down.
