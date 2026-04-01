# Pi-hole + InkyPi + Tailscale Setup

## Hardware

- Raspberry Pi Zero 2W connected via 2.4GHz WiFi (Zero 2W does not support 5GHz)
- Static IP reserved via router DHCP reservation (tied to Pi's MAC address)

## Router (TP-Link Archer)

- **Primary DNS:** Pi's reserved local IP
- **Secondary DNS:** A fallback DNS in case Pi-hole goes down (e.g. Cloudflare `1.1.1.1`, Google `8.8.8.8`, or your preferred provider)
- Address reservation configured under: Advanced → Network → DHCP Server → Address Reservation

## Pi-hole

- Installed on Raspberry Pi Zero 2W
- Upstream DNS: Cloudflare `1.1.1.1`

### Blocklists

| List | URL |
|------|-----|
| StevenBlack hosts | https://raw.githubusercontent.com/StevenBlack/hosts/master/hosts |
| Hagezi Pro | https://cdn.jsdelivr.net/gh/hagezi/dns-blocklists@latest/adblock/pro.txt |

After adding lists, update gravity:

```bash
pihole -g
```

## Tailscale

- Pi is a Tailscale node
- Pi's Tailscale IP (`100.x.x.x`) set as custom nameserver in Tailscale admin console (DNS tab)
- "Override local DNS" enabled → Pi-hole filters DNS for all Tailscale-connected devices even when away from home

## Known Behaviour

| Scenario | Pi-hole active? |
|----------|----------------|
| Device on home network, no VPN | Yes |
| Brave browser with Shields on | No — Brave blocks at browser level instead |
| VPN active on device | No — VPN takes over DNS |
| Connected via Tailscale (away from home) | Yes |
| Smart TVs / phones / stock browsers | Yes |

### Brave Browser

Brave bypasses Pi-hole when Shields are on. To make Brave use Pi-hole:

- Settings → Privacy and security → Security → Use secure DNS → **Off**

### VPN

When a VPN is active on a device, DNS is handled by the VPN and Pi-hole is bypassed on that device. This is expected behaviour.

## Local DNS for k8s Services

Services exposed via the Tailscale operator get a `100.x.x.x` Tailscale IP. To access them by hostname on your home network, add a DNS record in Pi-hole:

**Local DNS → DNS Records:**

| Domain | IP |
|--------|----|
| `mealie.home` | Tailscale IP of Traefik service |

To find the IP:
```bash
kubectl get svc -n kube-system traefik
```

Add new services here as they're exposed. All devices connected to Tailscale (or on the home network with Pi-hole as DNS) will resolve the hostname.

> **Note:** These are Tailscale IPs — only reachable from devices with Tailscale connected.

---

## Useful Commands

```bash
pihole status        # Check Pi-hole status
pihole enable        # Enable Pi-hole
pihole disable       # Disable Pi-hole
pihole -g            # Update gravity (blocklists)
sudo reboot          # Reboot Pi (Pi-hole auto-starts on boot)
tailscale status     # Check Tailscale status
tailscale ip         # Show Pi's Tailscale IP
```
