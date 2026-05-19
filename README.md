# usg-wifi-lte-upstreams

Two Raspberry Pi WAN bridges in front of a UniFi Security Gateway (USG), plus
the glue script that fixes the USG's unreliable WAN failback.

## The situation

A home network is fronted by a UniFi USG with two WAN ports. The two
upstreams are not your usual "two ISPs" pair:

- **Primary WAN — `rk-wifi-bridge`** (Raspberry Pi). Connects via Wi-Fi to
  `Trailbus`, which is a **Starlink Wi-Fi hotspot living inside a campervan**.
  When the van is parked at home, Trailbus is in range and the house gets the
  fast Starlink uplink. When the van drives away, Trailbus simply disappears
  from the air.
- **Secondary WAN — `rk-lte-bridge`** (Raspberry Pi). USB-tethers a
  TP-Link M7000 4G hotspot. Always present, slower and metered, used only
  when the campervan is away.

Each Pi runs its own DHCP server on a `192.168.50.0/24` link toward its USG
WAN port and MASQUERADEs out the upstream. From the USG's point of view it
is just a normal dual-WAN setup; from the upstream's point of view it is a
single client (double NAT — accepted on purpose).

```
 ┌─────────── van home (Trailbus in range) ──────────┐
 │                                                   │
 │   Starlink ── Wi-Fi(Trailbus) ─▶ rk-wifi-bridge ──┼─▶ USG WAN1  (primary, active)
 │                                                   │
 │   4G ─────── USB tether ──────▶ rk-lte-bridge ────┼─▶ USG WAN2  (secondary, idle)
 │                                                   │
 └───────────────────────────────────────────────────┘

 ┌─────────── van away (no Trailbus) ─────────────────────────┐
 │                                                            │
 │   rk-wifi-bridge has no upstream → drops its eth0 carrier ─┼─▶ USG WAN1  (dead)
 │                                                            │
 │   4G ─────── USB tether ──────▶ rk-lte-bridge ─────────────┼─▶ USG WAN2  (now active)
 │                                                            │
 └────────────────────────────────────────────────────────────┘
```

## The problem this repo solves

The UniFi USG fails **over** to the secondary WAN within seconds when WAN1
carrier disappears. Failing **back** to the primary WAN, however, is famously
unreliable: even after WAN1 reports healthy again the USG often happily
stays on WAN2 indefinitely. (Search the Ubiquiti community forums — this
has been a known complaint across many firmware releases.)

So when the van returns home, the house technically *has* fast Starlink
available again, but the USG keeps everyone routed through the slow LTE
hotspot until something forces its hand.

This repo automates that nudge.

## How the failback push works

1. `rk-wifi-bridge` runs `wan-link-check`, a small bash service that pings
   `1.1.1.1` (primary probe target) and `8.8.8.8` (secondary) out `wlan0`
   every 5 s. After 3 consecutive double-failures (~15 s) it forces `eth0`
   link DOWN. The USG sees carrier loss on WAN1 and fails over to WAN2.
2. While the van is away, `rk-lte-bridge` keeps carrying the household.
3. When `wlan0` reaches the internet again, `wan-link-check` brings `eth0`
   back up. The USG now sees WAN1 carrier — and ignores it.
4. `wan-link-check` then watches for a **stability window**: 180 s of clean
   *primary* probes (`1.1.1.1` via `wlan0`). Any single primary failure in
   that window resets the timer.
5. Once 180 s of clean primary is observed, `wan-link-check` opens an SSH
   connection (over Tailscale) to `rk-lte-bridge` and asks it to run
   `force-usg-failback`. That script drops `rk-lte-bridge`'s `eth0` for
   30 s.
6. From the USG's point of view, **the currently active WAN just went
   down**. It fails over — back to WAN1. The home is now on Starlink again
   without anyone touching it.
7. After the 30 s drop, `rk-lte-bridge`'s `eth0` comes back up. WAN2 is
   healthy and idle, ready for the next time the van leaves.

## Why Tailscale

Both Pis are joined to a [Tailscale](https://tailscale.com/) tailnet, and
SSH access is via Tailscale hostnames (`rk-wifi-bridge`, `rk-lte-bridge`)
rather than LAN IPs.

Reason: the management path can't share fate with the WAN it manages.

- When `rk-wifi-bridge` deliberately drops its own `eth0`, the LAN-side
  route into it disappears.
- When `rk-lte-bridge` is asked to drop `eth0`, the same is true for it.
- During an active WAN outage, half the house may be unreachable from the
  other half.
- Plugging a keyboard and monitor into a headless Pi in a closet to
  recover from a botched change is not practical.

Tailscale gives each Pi an out-of-band management plane that survives all
of the above as long as the box has any working upstream (Wi-Fi, LTE, or
even another Tailscale node relaying for it). All the SSH commands in this
repo assume that path.

## Why the TP-Link M7000

Two practical constraints picked this specific 4G hotspot:

- **It exposes USB tethering (RNDIS) out of the box.** The Pi drives the
  modem over a USB data cable and gets a clean `usb0` network interface —
  no Wi-Fi client role, no reassociation churn, no contention with the
  Trailbus-side `wlan0`. Hardware revisions V2 and V3 of the M7000 expose
  RNDIS; V1 does not. If you're buying one for this build, get a V2 or
  newer.
- **The SIM costs nothing on top of an existing O₂ (CZ) mobile tariff.**
  O₂'s [O2 Connect](https://www.o2.cz/osobni/volani/o2-connect) add-on
  lets a NEO+ or YOU mobile plan share its calls / SMS / data pool with a
  separate SIM in a supported "smart device" — and the first connected
  device receives a 99 Kč monthly discount that fully offsets its fee, so
  the extra SIM is effectively free. The M7000 is explicitly named on
  O₂'s allow-list of devices that may carry an O2 Connect SIM, alongside
  the TP-Link M7200 and the ZTE MF833U1 USB modem (the other allowed
  categories are notebooks/tablets, cameras and trail cameras,
  sensors/controllers, and wearables — phones are excluded). Picking a
  hotspot O₂ already blesses means there's no IMEI-class risk hanging
  over this WAN.

  ![O₂ Connect supported devices — screenshot from O₂ self-service
  help, listing TP-Link M7200/M7000 and ZTE MF833U1 as the allowed
  portable modems](docs/o2-connect-supported-devices.png)

## Repo layout

```
rk-wifi-bridge/                       # Pi — Wi-Fi(Trailbus) to USG WAN1
  usr/local/sbin/wan-link-check       # the probe + failback orchestrator
  etc/systemd/system/wan-link-check.service
  etc/sysctl.d/99-router.conf         # IPv4+IPv6 forwarding
  etc/iptables/rules.v4               # MASQUERADE out wlan0
  etc/dnsmasq.d/router.conf           # DHCP for the 192.168.50.0/24 link
  etc/NetworkManager/system-connections/router-lan.nmconnection.example

rk-lte-bridge/                        # Pi — M7000 USB tether to USG WAN2
  usr/local/sbin/force-usg-failback          # drops eth0 for 30 s
  usr/local/sbin/force-usg-failback-trigger  # restricted-key dispatcher
  usr/local/sbin/router-activate             # flip to router mode + reboot
  usr/local/sbin/router-deactivate           # revert to plain DHCP client
  etc/sysctl.d/99-router.conf
  etc/iptables/rules.v4               # MASQUERADE out usb+
  etc/dnsmasq.d/router.conf
  etc/NetworkManager/system-connections/router-lan.nmconnection.example
  etc/NetworkManager/system-connections/wan-usb-tether.nmconnection.example
  etc/sudoers.d/010_johndoe.example      # NOPASSWD sudo for the trigger
```

Each host directory mirrors the absolute paths the files live at on the Pi.
Files ending in `.example` are templates — copy, drop the suffix, fill in
UUIDs / passwords / etc. locally and never commit those copies back.

## Setup, in order

The host-specific READMEs walk through each step:

1. [`rk-wifi-bridge/README.md`](rk-wifi-bridge/README.md) — bring up the
   Wi-Fi bridge with WAN-aware `eth0` toggling.
2. [`rk-lte-bridge/README.md`](rk-lte-bridge/README.md) — bring up the LTE
   bridge and install the `force-usg-failback` trigger.
3. Generate an SSH keypair on `rk-wifi-bridge` (as root, since
   `wan-link-check.service` runs as root) and add the public key to
   `rk-lte-bridge`'s `~/.ssh/authorized_keys` with the forced-command
   restriction described in `rk-lte-bridge/README.md`.

## On the USG side

Nothing in this repo touches the USG itself. Plain dual-WAN failover is
all that's required there:

- Configure WAN1 (Pi-wifi-bridge side) and WAN2 (Pi-lte-bridge side) each
  with a DHCP client.
- Set the WAN role to **Failover Only** (not Load Balancing).
- Default ping targets are fine; the failback push doesn't depend on them.

## Not in this repo

- Wi-Fi passphrase / SSID config for `wlan0` — set this up with your own
  preferred tool (Network Manager, netplan, `wpa_supplicant`).
- The actual SSH public/private keys — generate fresh ones at deploy time.
- Operational extras kept in personal infrastructure: Discord notify hooks,
  Home Assistant bandwidth telemetry, auto-update jobs, Tailscale exit-node
  configuration. They're useful but orthogonal to the WAN-bridge purpose
  of this repo.
