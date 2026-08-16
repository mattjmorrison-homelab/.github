# Networking

> **Note:** This document describes both the current state and the planned future architecture. Sections marked "Planned" are aspirational and not yet in place.

## Current State

### Morrison WiFi (Google Wifi)
- Subnet: `192.168.86.0/24`
- DHCP range: `192.168.86.20` – `192.168.86.250`
- Available for static assignment: `192.168.86.2` – `192.168.86.19` and `192.168.86.251` – `192.168.86.254`
- All homelab nodes and devices are currently on this network
- MetalLB IP pool: `192.168.86.2` – `192.168.86.19`

### Deco Network (TP-Link Deco)
- Subnet: `192.168.68.0/22` (covers `192.168.68.0` – `192.168.71.255`)
- DHCP range: `192.168.68.50` – `192.168.71.250`
- Currently a separate network, not yet carrying homelab traffic
- Backbone: Ethernet over Coax (EoC) connecting access points

## Planned Architecture

The goal is to replace the Google Wifi network with the Deco + OPNsense setup as the primary network for all homelab traffic.

### Hardware
- **Protectli** — will run OPNsense as the primary firewall/router
- **TP-Link Deco** — will serve as the primary WiFi/switching layer with EoC backbone
- **Google Wifi** — to be retired once migration is complete

### Migration Steps (not yet started)
1. Configure Protectli with OPNsense
2. Migrate homelab nodes and devices to the Deco network
3. Update MetalLB IP pool to use the `192.168.68.0/22` subnet
4. Update DNS records in NixOS host configs to reflect new IPs

## TODOs

- [ ] **Pihole** — deploy DNS ad-blocking once networking is settled on a single router. Pihole needs a stable DNS IP that all devices point to; setting it up mid-migration would require reconfiguring it multiple times. Deploy after Protectli/OPNsense migration is complete.
- [ ] **MetalLB IP pool update** — update pool from `192.168.86.x` to the appropriate range on the new network after migration
