# Cluster

## Current State

Single-node cluster:

- **Server node** (`matt-nix`) — runs k3s control plane and all workloads

## Planned: Raspberry Pi 5 Agent Node

Adding a Raspberry Pi 5 (16 GB RAM) as a dedicated worker (agent) node. Primary purpose: run game server workloads (Minecraft) so they don't compete with homelab services on the main node.

### Hardware

- Raspberry Pi 5 (16 GB)
- **HAT**: Waveshare PCIe to M.2 Adapter
- **SSD**: Samsung OEM NVMe — 256 GB, M.2 2242 (22mm × 42mm), PCIe Gen 3 x4 (MZ-ALQ256B)

## TODOs

### Pi 5 Hardware Setup

- [ ] Enable PCIe in `/boot/firmware/config.txt` — add `dtparam=pciex1_gen=3` and reboot
- [ ] Verify NVMe drive shows up as `nvme0n1` (`lsblk` / `sudo nvme list`)
- [ ] Format and mount the NVMe SSD
- [ ] (Optional) Configure Pi to boot from NVMe instead of SD card

### OS Decision

- [ ] Decide whether to keep Pi OS or move to NixOS
  - **Pi OS**: faster to get running, SSD HAT documentation is plentiful, less consistent with rest of homelab
  - **NixOS**: consistent with rest of homelab (fully managed as code), Pi 5 support is solid via `nixos-hardware` raspberry-pi-5 module, PCIe/NVMe works well since Pi 5 has native PCIe

### Joining the Cluster

- [ ] Install k3s agent on Pi and join to the cluster
  - Pi OS: `curl -sfL https://get.k3s.io | K3S_URL=<server-url> K3S_TOKEN=<token> sh -s - agent`
  - NixOS: add k3s agent module to host config in `homelab` repo
- [ ] Add a Makefile target in `homelab` for the join operation (token must not be committed to source control)
- [ ] Label the Pi node for workload targeting (e.g., `node-role: gameserver`)

### Workload Scheduling

- [ ] Add `nodeSelector` to Minecraft deployment to schedule only on Pi node
- [ ] Confirm other services continue scheduling on `matt-nix` (no unintended migration)
