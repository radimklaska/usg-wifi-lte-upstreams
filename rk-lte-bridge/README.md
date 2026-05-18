# rk-lte-bridge

Raspberry Pi. TP-Link M7000 4G hotspot over USB tether → `eth0` → USG WAN2.

Secondary WAN — only carries traffic when `rk-wifi-bridge` (Trailbus) is
unavailable. Also hosts the `force-usg-failback` trigger that
`rk-wifi-bridge` SSHes into to push the USG back to its primary WAN once
Trailbus has been stable again for 180 s.

## Files and where they live

| Path in this repo | Path on the Pi | Mode |
|---|---|---|
| `usr/local/sbin/force-usg-failback` | `/usr/local/sbin/force-usg-failback` | 0755 root:root |
| `usr/local/sbin/force-usg-failback-trigger` | `/usr/local/sbin/force-usg-failback-trigger` | 0755 root:root |
| `usr/local/sbin/router-activate` | `/usr/local/sbin/router-activate` | 0755 root:root |
| `usr/local/sbin/router-deactivate` | `/usr/local/sbin/router-deactivate` | 0755 root:root |
| `etc/sysctl.d/99-router.conf` | `/etc/sysctl.d/99-router.conf` | 0644 root:root |
| `etc/iptables/rules.v4` | `/etc/iptables/rules.v4` | 0644 root:root |
| `etc/dnsmasq.d/router.conf` | `/etc/dnsmasq.d/router.conf` | 0644 root:root |
| `etc/NetworkManager/system-connections/router-lan.nmconnection.example` | `/etc/NetworkManager/system-connections/router-lan.nmconnection` | 0600 root:root |
| `etc/NetworkManager/system-connections/wan-usb-tether.nmconnection.example` | `/etc/NetworkManager/system-connections/wan-usb-tether.nmconnection` | 0600 root:root |
| `etc/sudoers.d/010_johndoe.example` | `/etc/sudoers.d/010_johndoe` | 0440 root:root |

## Setup

Assumes Debian/Raspberry Pi OS on a Pi with the `johndoe` user
created, Tailscale installed and logged in, and an M7000 hotspot plugged
into a USB port via a **data** cable (charge-only USB cables don't
enumerate the RNDIS device).

1. **Forwarding** (IPv4 + IPv6, same reasoning as `rk-wifi-bridge`):
   ```sh
   sudo install -m 0644 etc/sysctl.d/99-router.conf /etc/sysctl.d/99-router.conf
   sudo sysctl --system
   ```
2. **iptables-persistent**:
   ```sh
   sudo apt-get install -y iptables-persistent
   sudo install -m 0644 etc/iptables/rules.v4 /etc/iptables/rules.v4
   sudo iptables-restore </etc/iptables/rules.v4
   ```
3. **dnsmasq**:
   ```sh
   sudo apt-get install -y dnsmasq
   sudo install -m 0644 etc/dnsmasq.d/router.conf /etc/dnsmasq.d/router.conf
   # leave dnsmasq DISABLED for now — router-activate will turn it on.
   sudo systemctl disable dnsmasq.service
   ```
4. **Network Manager profiles** — install both examples (drop `.example`).
   Pin each by `interface-name` so the stock "Wired connection 1" profile
   can't accidentally capture `usb0` when the M7000 enumerates first.
   ```sh
   sudo install -m 0600 \
     etc/NetworkManager/system-connections/router-lan.nmconnection.example \
     /etc/NetworkManager/system-connections/router-lan.nmconnection
   sudo install -m 0600 \
     etc/NetworkManager/system-connections/wan-usb-tether.nmconnection.example \
     /etc/NetworkManager/system-connections/wan-usb-tether.nmconnection
   sudo nmcli connection reload
   ```
   Make sure the stock `Wired connection 1` profile is `interface-name=eth0`
   and `autoconnect=yes` (it is the "home-LAN mode" used before
   `router-activate` is run).
5. **NOPASSWD sudo** for the trigger. The `force-usg-failback-trigger`
   wrapper invokes `force-usg-failback` via `sudo -n`, so the SSH user
   needs passwordless sudo for at least that command. The example file
   grants the broader `NOPASSWD: ALL` that this deployment uses; tighten
   if you prefer least-privilege:
   ```sh
   sudo install -m 0440 etc/sudoers.d/010_johndoe.example \
                        /etc/sudoers.d/010_johndoe
   sudo visudo -cf /etc/sudoers.d/010_johndoe
   ```
   *Least-privilege variant — recommended if you don't otherwise want
   blanket NOPASSWD:*
   ```
   johndoe ALL=(root) NOPASSWD: /usr/local/sbin/force-usg-failback
   ```
6. **Install the failback scripts**:
   ```sh
   sudo install -m 0755 usr/local/sbin/force-usg-failback \
                        /usr/local/sbin/force-usg-failback
   sudo install -m 0755 usr/local/sbin/force-usg-failback-trigger \
                        /usr/local/sbin/force-usg-failback-trigger
   ```
7. **Authorise the rk-wifi-bridge key** with a forced-command restriction
   so the key can only do failback or `ping`, never a general shell. Append
   to `~johndoe/.ssh/authorized_keys`:
   ```
   command="/usr/local/sbin/force-usg-failback-trigger",restrict ssh-ed25519 AAAA... wan-link-check@rk-wifi-bridge
   ```
   The `restrict` keyword bundles `no-port-forwarding`, `no-X11-forwarding`,
   `no-agent-forwarding`, `no-pty`, etc. The dispatcher reads
   `$SSH_ORIGINAL_COMMAND` to choose between `ping` (no-op) and `failback`.
8. **Install the activate / deactivate helpers** and flip into router mode:
   ```sh
   sudo install -m 0755 usr/local/sbin/router-activate /usr/local/sbin/router-activate
   sudo install -m 0755 usr/local/sbin/router-deactivate /usr/local/sbin/router-deactivate
   sudo router-activate          # toggles NM profiles, enables dnsmasq, reboots
   ```

After reboot, plug `eth0` into the USG's WAN2 port. The USG should pick up
a `192.168.50.x` DHCP lease from this Pi.

## Manual test

From `rk-wifi-bridge`, as root:

```sh
sudo ssh -i /root/.ssh/id_failback_ed25519 \
         -o BatchMode=yes -o ConnectTimeout=10 \
         johndoe@rk-lte-bridge ping
```

Expected output: `pong`. No carrier change on `eth0`.

To actually exercise the eth0 drop:

```sh
sudo ssh -i /root/.ssh/id_failback_ed25519 \
         johndoe@rk-lte-bridge failback
```

This drops `eth0` for 30 s and brings it back. The SSH session rides
Tailscale (which is carried over `usb0` / M7000), so dropping `eth0` does
not kill the session.

## Why a separate trigger wrapper

The forced-command in `authorized_keys` runs *one fixed command*, not the
client's command. Without the dispatcher, the key would either always run
`force-usg-failback` (no way to test connectivity without dropping eth0)
or be a plain shell key (much broader blast radius if it ever leaks). The
wrapper reads `$SSH_ORIGINAL_COMMAND` so the same key can do `ping` for
health checks *and* `failback` for the real thing — and nothing else.
