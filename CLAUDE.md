# CLAUDE.md

Context for any AI assistant working in this repo.

## What this project is

Two Raspberry Pi WAN bridges feeding a UniFi USG's WAN1 and WAN2 ports:

- **`rk-wifi-bridge`** (Pi) — Wi-Fi → eth0. Wi-Fi upstream is `Trailbus`,
  a Starlink Wi-Fi hotspot that lives in a campervan and is only in range
  when the van is parked at home.
- **`rk-lte-bridge`** (Pi) — USB tether (M7000 4G hotspot) → eth0.
  Always-on, metered, secondary.

Both Pis MASQUERADE their upstream and run dnsmasq on `192.168.50.1/24` for
the downstream USG WAN port. Both are on Tailscale so they remain
reachable for SSH regardless of which WAN is currently active.

## Why the failback orchestration exists

The USG fails over within seconds when WAN1 carrier drops, but it does not
reliably fail **back** to WAN1 once it recovers. The workaround in this
repo is to have `rk-wifi-bridge` SSH into `rk-lte-bridge` after a 180 s
"primary is stable again" window and ask it to drop its own `eth0` for
30 s. The USG sees the *currently active* WAN go down and is forced to
flip back to the primary, which is healthy again.

## Where to make changes

- The probe loop and orchestration live in
  `rk-wifi-bridge/usr/local/sbin/wan-link-check`. State (needs_failback,
  stability window) is in-memory bash variables; restarting the service
  resets them. That is by design.
- The "drop eth0 for 30 s" mechanism lives in
  `rk-lte-bridge/usr/local/sbin/force-usg-failback`. It is intentionally
  small. A `flock` guards against overlapping runs.
- The SSH boundary uses a forced-command wrapper
  (`force-usg-failback-trigger`) so the restricted key can only invoke the
  failback (or `ping` → `pong` for connectivity tests). Do not loosen this
  to a general shell.

## Things that look weird but are intentional

- Both Pis use the same `192.168.50.0/24` for the link to the USG. That
  is fine: each is a separate L2 segment plugged into a different USG WAN
  port, so the overlap is invisible to everything but the human reader.
- The setup is double-NAT (upstream NATs the Pi, the Pi NATs the USG,
  the USG NATs the home LAN). Bridging was considered and rejected for
  both upstreams; in particular the M7000 tether hands out a single DHCP
  lease and cannot pass it through to a downstream router.
- `wan-link-check` resets its "stable" timer on **any single primary
  probe failure**, even one where the secondary probe immediately
  succeeds. The whole point is to not fail back onto a flaky primary.
- The wlan0 Wi-Fi profile (where the Trailbus passphrase lives) is
  managed by netplan, not by Network Manager — that's why no `wlan0`
  connection file ships in this repo. NM only owns `eth0`.

## Tailscale assumption

Every "SSH to the other Pi" in this codebase resolves via Tailscale
MagicDNS, not via the home LAN. This is deliberate:

- When `rk-wifi-bridge` drops its own `eth0` (or `rk-lte-bridge` drops
  theirs), the LAN-side IP becomes unreachable.
- Tailscale survives because the daemon's connectivity rides whichever
  upstream is still up (Wi-Fi or LTE), and falls back to DERP relays if
  needed.
- This means an AI assistant editing live can ssh into either Pi at any
  time as long as the box has *any* working internet — no need for the
  USG-side LAN to be working.

If a script in this repo ever references the other Pi via a 192.168.x
address, that's a regression — fix it to use the Tailscale hostname.

## Safe to make public

This repo is structured so that nothing committed reveals credentials,
specific Tailscale IPs, Wi-Fi passphrases, or other deployment secrets.
Hostnames are intentionally in the open. When adding new files:

- Strip UUIDs from any Network Manager profile examples (NM regenerates
  on first import).
- Replace any concrete Tailscale IP with a placeholder like
  `<TAILSCALE_IP>` if you must reference one at all.
- Do not commit `.env` files, real SSH keys, or actual webhook URLs.
- Operational extras tied to one deployment (Discord webhooks, HA
  long-lived tokens, auto-update timers, exit-node configuration) belong
  in a separate private location, not here.

## How to test changes safely

The realistic ways to drive the system through its states:

- Manual failback: from `rk-wifi-bridge` as root,
  `ssh -i /root/.ssh/id_failback_ed25519 johndoe@rk-lte-bridge ping`
  → expect `pong`. Then `... failback` to actually trigger the 30 s
  eth0 drop.
- Simulated outage: temporarily disable Wi-Fi on `rk-wifi-bridge`
  (disconnect from Trailbus or block the AP) and watch
  `journalctl -t wan-link-check -f`. The state machine should drive
  through: probes fail → eth0 down → reconnect → eth0 up → 180 s
  stability window → SSH trigger.
- Dry-run of the SSH path: the `ping` subcommand exists exactly so that
  the key + forced-command wrapper can be exercised without touching
  `eth0`.

When a change is shipped, copy the script into the right path on the live
Pi (the in-repo path mirrors the absolute filesystem path), then
`systemctl restart wan-link-check.service` on `rk-wifi-bridge`. There is
no service to restart on `rk-lte-bridge`; the trigger script is invoked
on demand by SSH.
