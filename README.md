# tailscale-mullvad-linux

Use **Tailscale** and **Mullvad VPN** simultaneously on Linux, without conflicts.

Mullvad's kill switch uses `nftables` to block all traffic outside its tunnel — including Tailscale's `100.64.0.0/10` subnet. This guide fixes that by:

1. Adding the Tailscale subnet as a trusted source in **firewalld**
2. Adding a static **route** to force Tailscale traffic through `tailscale0` instead of `wg0-mullvad`

---

## How Mullvad breaks Tailscale

When Mullvad connects, it creates a `table inet mullvad` in nftables with `policy drop` on both `input` and `output` chains. The whitelist includes standard RFC-1918 ranges (`10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`) but **not** `100.64.0.0/10` (Tailscale's CGNAT range).

Additionally, Mullvad adds a routing rule that sends all traffic through `wg0-mullvad`, which takes priority over Tailscale's `tailscale0` interface.

Two fixes are required:
- **Firewall**: allow traffic to/from `100.64.0.0/10` and `tailscale0`
- **Routing**: add a route that forces Tailscale traffic through `tailscale0`

---

## Requirements

- Linux (tested on **Fedora**; see [Other Distros](#other-distros) for Debian/Ubuntu/Arch)
- [Tailscale](https://tailscale.com/download/linux) installed and logged in
- [Mullvad VPN](https://mullvad.net/en/download/linux) installed (GUI or CLI)
- `firewalld` active (default on Fedora/RHEL-based distros)

---

## Installation — Fedora / RHEL-based

### Step 1 — Configure firewalld

Allow Tailscale traffic through firewalld's trusted zone:

```bash
sudo firewall-cmd --permanent --zone=trusted --add-source=100.64.0.0/10
sudo firewall-cmd --permanent --zone=trusted --add-interface=tailscale0
sudo firewall-cmd --reload
```

These rules survive reboots automatically.

### Step 2 — Fix the routing (persistent)

Create a NetworkManager dispatcher script that adds the correct route every time a VPN connects:

```bash
sudo tee /etc/NetworkManager/dispatcher.d/99-tailscale-mullvad > /dev/null <<'SCRIPT'
#!/bin/bash
# Re-add Tailscale route after Mullvad connects
if [ "$2" = "vpn-up" ] || [ "$2" = "up" ] || [ "$2" = "connectivity-change" ]; then
    ip route add 100.64.0.0/10 dev tailscale0 2>/dev/null || true
fi
SCRIPT

sudo chmod +x /etc/NetworkManager/dispatcher.d/99-tailscale-mullvad
```

### Step 3 — Add the route for the current session

The dispatcher script handles future connections. For the current session, add the route now:

```bash
sudo ip route add 100.64.0.0/10 dev tailscale0
```

> **Note:** This manual command is only needed once per reboot (or after first Mullvad connection). The dispatcher script handles it automatically going forward.

### Step 4 — Test

Connect to Mullvad from the GUI (or via CLI), then:

```bash
ping <tailscale-ip-of-a-device>
# e.g. ping 100.122.214.102
```

You should receive replies. If not, see [Troubleshooting](#troubleshooting).

---

## Other Distros

### Debian / Ubuntu (uses `ufw` or raw `iptables`)

These distros typically don't use firewalld. The firewall fix is handled differently, but the routing fix is identical.

**Routing fix (same as Fedora):**

```bash
# Create dispatcher script
sudo mkdir -p /etc/NetworkManager/dispatcher.d/
sudo tee /etc/NetworkManager/dispatcher.d/99-tailscale-mullvad > /dev/null <<'SCRIPT'
#!/bin/bash
if [ "$2" = "vpn-up" ] || [ "$2" = "up" ] || [ "$2" = "connectivity-change" ]; then
    ip route add 100.64.0.0/10 dev tailscale0 2>/dev/null || true
fi
SCRIPT
sudo chmod +x /etc/NetworkManager/dispatcher.d/99-tailscale-mullvad

# Add route for current session
sudo ip route add 100.64.0.0/10 dev tailscale0
```

**Firewall fix with nftables directly** (if no firewalld):

```bash
# Add rules to Mullvad's nftables chains
sudo nft insert rule inet mullvad output oif "tailscale0" accept
sudo nft insert rule inet mullvad input iif "tailscale0" accept
sudo nft insert rule inet mullvad output ip daddr 100.64.0.0/10 accept
sudo nft insert rule inet mullvad input ip saddr 100.64.0.0/10 accept
```

> ⚠️ These nftables rules are **not persistent** — Mullvad wipes them on reconnect. Include them in the dispatcher script above for persistence.

### Arch Linux

Same approach as Debian/Ubuntu. If using `systemd-networkd` instead of NetworkManager, use a `systemd` service instead of a dispatcher script:

```ini
# /etc/systemd/system/tailscale-route.service
[Unit]
Description=Fix Tailscale route after Mullvad
After=mullvad-daemon.service tailscaled.service
Wants=mullvad-daemon.service tailscaled.service

[Service]
Type=oneshot
ExecStart=/sbin/ip route add 100.64.0.0/10 dev tailscale0
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl enable --now tailscale-route.service
```

---

## How it works

| Problem | Root Cause | Fix |
|---|---|---|
| Tailscale traffic blocked | Mullvad's nftables `policy drop` doesn't whitelist `100.64.0.0/10` | firewalld trusted zone bypasses Mullvad's nftables chains |
| Route goes to `wg0-mullvad` | Mullvad adds a catch-all routing rule in its own routing table | `ip route add 100.64.0.0/10 dev tailscale0` overrides with a more specific route |

---

## Troubleshooting

**`ping` still fails after setup:**

Check where the route actually goes:
```bash
ip route get 100.x.x.x
# Should say: dev tailscale0
# If it says: dev wg0-mullvad → the route wasn't added
```

**Route is missing after reboot:**

Check the dispatcher script is executable and in the right place:
```bash
ls -la /etc/NetworkManager/dispatcher.d/99-tailscale-mullvad
```

Trigger it manually or reboot and reconnect Mullvad.

**Tailscale connects via DERP only (no direct connection):**

DERP relay is used when direct peer-to-peer is not possible (NAT, firewall on the remote device). This is a separate issue from Mullvad coexistence — check the firewall on the other Tailscale device.

**Mullvad kill switch re-blocks after reconnect:**

Make sure the dispatcher script fires on `vpn-up`. Test it manually:
```bash
sudo /etc/NetworkManager/dispatcher.d/99-tailscale-mullvad wg0-mullvad vpn-up
ip route show | grep tailscale
```

---

## Arch Linux / systemd-networkd — reconnect-proof method (no firewalld)

The Fedora method inserts rules into Mullvad's own `inet mullvad` table, which **Mullvad wipes on every reconnect**. On systems without NetworkManager the dispatcher script never fires either, so the fix silently breaks after the first reconnect. This variant survives reconnects, reboots, and Mullvad's **lockdown mode (kill switch)**.

It works by:

- A **separate** nftables table that stamps Mullvad's own exclusion mark (`ct mark 0x00000f41`) onto Tailscale traffic. Mullvad's permanent `ct mark 0x00000f41 accept` rule (present when connected *and* in lockdown) then lets it through. Because the table is independent of Mullvad's, it is never wiped on reconnect.
- Setting **`rp_filter` to loose (`2`)** — without this, replies from Tailscale peers are silently dropped by strict reverse-path filtering once Mullvad's policy routing is active (symptom: traffic works one direction only).
- Loading everything via a dedicated systemd unit after `tailscaled`.

> This only sets `ct mark`, **not** `meta mark`/fwmark. Routing is handled by the explicit route below. Overwriting Tailscale's own `0x80000` fwmark breaks Tailscale's WireGuard handshake routing.

**1. Firewall mark — `/etc/nftables.d/mullvad_tailscale.conf`**

```
#!/usr/sbin/nft -f
table inet tsExclude
delete table inet tsExclude
table inet tsExclude {
  chain overlayOut {
    type filter hook output priority -151; policy accept;
    ip daddr 100.64.0.0/10 ct mark set 0x00000f41
    oifname "tailscale0" ct mark set 0x00000f41
  }
  chain overlayIn {
    type filter hook input priority -151; policy accept;
    ip saddr 100.64.0.0/10 ct mark set 0x00000f41
    iifname "tailscale0" ct mark set 0x00000f41
  }
}
```

**2. Reverse-path filter — `/etc/sysctl.d/99-tailscale-rpfilter.conf`**

```
net.ipv4.conf.all.rp_filter=2
net.ipv4.conf.default.rp_filter=2
```

**3. systemd unit — `/etc/systemd/system/tailscale-mullvad.service`**

```
[Unit]
Description=Tailscale + Mullvad coexistence (firewall mark + route)
After=tailscaled.service network-online.target
Wants=tailscaled.service network-online.target

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStartPre=/bin/bash -c 'for i in $(seq 1 30); do ip link show tailscale0 >/dev/null 2>&1 && exit 0; sleep 1; done; exit 0'
ExecStart=/usr/bin/nft -f /etc/nftables.d/mullvad_tailscale.conf
ExecStart=/usr/bin/ip route replace 100.64.0.0/10 dev tailscale0

[Install]
WantedBy=multi-user.target
```

Enable it:

```
sudo mkdir -p /etc/nftables.d
sudo sysctl --system
sudo systemctl daemon-reload
sudo systemctl enable --now tailscale-mullvad.service
```

Do **not** enable the distro's generic `nftables.service` if you weren't already using it — on Arch it loads a default drop-policy ruleset that blocks inbound Tailscale.

**Why it survives reconnect:** Mullvad re-creates its `ct mark 0x00000f41 accept` rules on every connect (and keeps them in lockdown mode). Since the `tsExclude` table only *marks* packets and lives outside Mullvad's table, Mullvad never touches it — the marked Tailscale traffic keeps passing.


## License

MIT
