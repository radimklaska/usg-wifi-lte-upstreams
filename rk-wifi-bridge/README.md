# rk-wifi-bridge

Raspberry Pi. Wi-Fi (`Trailbus`, a Starlink hotspot in a campervan) →
`eth0` → USG WAN1.

Always-on bridge. `wan-link-check` toggles `eth0` carrier based on
upstream reachability and, once the upstream has been clean for 180 s after
an outage, SSHes to `rk-lte-bridge` to force the USG back to this WAN.

## Files and where they live

| Path in this repo | Path on the Pi | Mode |
|---|---|---|
| `usr/local/sbin/wan-link-check` | `/usr/local/sbin/wan-link-check` | 0755 root:root |
| `etc/systemd/system/wan-link-check.service` | `/etc/systemd/system/wan-link-check.service` | 0644 root:root |
| `etc/sysctl.d/99-router.conf` | `/etc/sysctl.d/99-router.conf` | 0644 root:root |
| `etc/iptables/rules.v4` | `/etc/iptables/rules.v4` | 0644 root:root |
| `etc/dnsmasq.d/router.conf` | `/etc/dnsmasq.d/router.conf` | 0644 root:root |
| `etc/NetworkManager/system-connections/router-lan.nmconnection.example` | `/etc/NetworkManager/system-connections/router-lan.nmconnection` | 0600 root:root |

## Setup

Assumes Debian 13 (trixie) on a Pi, with the `johndoe` user already
created and a working Wi-Fi connection to `Trailbus` (managed by netplan on
`wlan0`, not by NetworkManager).

1. **Forwarding** — both IPv4 and IPv6 are required (IPv6 forwarding is
   needed for Tailscale exit-node / subnet relay even if your traffic is
   IPv4-only). Install `etc/sysctl.d/99-router.conf` and apply:
   ```sh
   sudo sysctl --system
   ```
2. **iptables-persistent** — install once, then drop `etc/iptables/rules.v4`
   in place:
   ```sh
   sudo apt-get install -y iptables-persistent
   sudo install -m 0644 etc/iptables/rules.v4 /etc/iptables/rules.v4
   sudo iptables-restore </etc/iptables/rules.v4
   ```
3. **dnsmasq** — DHCP and DNS for the 192.168.50.0/24 link to the USG WAN
   port:
   ```sh
   sudo apt-get install -y dnsmasq
   sudo install -m 0644 etc/dnsmasq.d/router.conf /etc/dnsmasq.d/router.conf
   sudo systemctl enable --now dnsmasq.service
   ```
4. **Network Manager profile for `eth0`** — copy the example, drop the
   `.example` suffix, and import. NM will fill in a fresh UUID on first
   load.
   ```sh
   sudo install -m 0600 \
     etc/NetworkManager/system-connections/router-lan.nmconnection.example \
     /etc/NetworkManager/system-connections/router-lan.nmconnection
   sudo nmcli connection reload
   sudo nmcli connection up router-lan
   ```
   Note: if your Pi was imaged via Pi Imager, there's a `netplan` YAML for
   `eth0` (DHCP client by default) — move it aside so netplan doesn't
   regenerate the DHCP profile on every boot. The Wi-Fi netplan YAML for
   `wlan0` stays.
5. **The probe service** — drops in place and start:
   ```sh
   sudo install -m 0755 usr/local/sbin/wan-link-check /usr/local/sbin/wan-link-check
   sudo install -m 0644 etc/systemd/system/wan-link-check.service \
                        /etc/systemd/system/wan-link-check.service
   sudo systemctl daemon-reload
   sudo systemctl enable --now wan-link-check.service
   ```
   Tail it:
   ```sh
   sudo journalctl -t wan-link-check -f
   ```
6. **SSH key for the failback push** — generate as root (the service runs
   as root):
   ```sh
   sudo install -d -m 0700 /root/.ssh
   sudo ssh-keygen -t ed25519 -N "" -f /root/.ssh/id_failback_ed25519 \
                   -C "wan-link-check@rk-wifi-bridge"
   sudo cat /root/.ssh/id_failback_ed25519.pub
   ```
   Add that public key to `rk-lte-bridge` as described in that host's
   README (note the forced-command restriction).
7. **Verify the SSH path** without touching anything:
   ```sh
   sudo ssh -i /root/.ssh/id_failback_ed25519 \
            -o BatchMode=yes -o ConnectTimeout=10 \
            johndoe@rk-lte-bridge ping
   ```
   Expected output: `pong`.

After that, the service is fully wired. Simulate an outage by disconnecting
Wi-Fi (e.g. `nmcli radio wifi off`) and watch the journal: probes fail →
`eth0` goes down → bring Wi-Fi back → probes recover → 180 s window → SSH
trigger fires.

## State machine, briefly

```
            ┌──────── probes failing ─────────┐
            │ (3 × ~5 s)                       │
            ▼                                  │
       drop eth0                               │
       needs_failback = 1                      │
            │                                  │
            ▼                                  │
   ┌─ probes recover ──────────────────┐       │
   │  bring eth0 up                    │       │
   │  open stability window (180 s)    │       │
   │  → SSH "failback" to rk-lte-bridge│       │
   │  needs_failback = 0               │       │
   └───────────────────────────────────┘       │
            │                                  │
            └──────── next outage ─────────────┘
```

Any single primary-probe failure inside the 180 s window resets the timer.
SSH failure defers the next attempt by another full window.
