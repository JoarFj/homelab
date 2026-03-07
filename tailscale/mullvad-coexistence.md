# Mullvad + Tailscale Coexistence on Linux

## Goal

Run Mullvad VPN and Tailscale simultaneously on a Linux desktop, with Pi-hole handling DNS for ad blocking even when Mullvad is active.

## Problem

Mullvad takes over all routing when connected, which causes Tailscale traffic (range `100.64.0.0/10`) to be sent to Mullvad's servers and dropped. Mullvad's built-in split tunneling (app exclusion list) does not work reliably for Tailscale on Linux.

## Solution: nftables traffic marking

Mark Tailscale-bound traffic so the kernel bypasses Mullvad's routing and hands it back to Tailscale instead.

### 1. Add rules to `/etc/nftables.conf`

Append the following table to the existing file (do not replace existing content):

```
table inet mullvad_tailscale {
  chain output {
    type route hook output priority 0; policy accept;
    ip daddr 100.64.0.0/10 ct mark set 0x00000f41 meta mark set 0x6d6f6c65;
  }

  chain input {
    type filter hook input priority -100; policy accept;
    ip saddr 100.64.0.0/10 ct mark set 0x00000f41 meta mark set 0x6d6f6c65;
  }
}
```

### 2. Load the rules

```bash
sudo nft -f /etc/nftables.conf
```

### 3. Persist on boot

```bash
sudo systemctl enable --now nftables
```

## Pi-hole DNS via Mullvad

With Tailscale now reachable while Mullvad is active, set Pi-hole as the DNS in Mullvad:

**Mullvad app → Settings → VPN Settings → DNS → Use custom DNS server**

Enter your Pi's Tailscale IP (`100.x.x.x`) — Pi-hole will handle DNS and ad blocking even when Mullvad is on.

## Behaviour Summary

| Mullvad | Tailscale | Pi-hole active? |
|---------|-----------|----------------|
| Off | On/Off | Yes (via router DNS) |
| On | On | Yes (via Pi Tailscale IP as custom DNS) |
| On | Off | No (Mullvad default DNS) |

## Notes

- Tailscale MagicDNS hostnames may not resolve when Mullvad is active — use Tailscale IPs directly
- Rules are applied at kernel level via nftables, which is more reliable than Mullvad's split tunneling UI on Linux
- Credit: https://theorangeone.net/posts/tailscale-mullvad/
