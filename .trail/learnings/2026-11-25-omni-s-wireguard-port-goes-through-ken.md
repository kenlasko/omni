---
title: "Omni's WireGuard port goes through a Traefik UDP entrypoint; nodes join through the machine config, not the old kernel arg"
author: Ken
date: 2026-11-25
branch: main
cost: { minutes: 7, tokens: 39174 }
id: l-8a614c51
scope: repo
status: active
replaces: l-e4cc7253
src: agent
touches:
  - patches/controlplane.yaml
---

## Decided
- `omni.laskonet.com` resolves to Traefik's own LAN address on the QNAP qnet network (192.168.1.100), not the NAS (192.168.1.3). WireGuard (UDP 50180) therefore goes through a Traefik UDP entrypoint to the Omni container, with `udp.timeout` raised to 300s (the 3s default expires sessions between Talos keepalives). Fixed 2026-11-25; all nodes reconnected without reboots.
- Cluster nodes join through the `SideroLinkConfig` document Omni pushes in each machine config (`omni.laskonet.com`). In Talos's SideroLink controller the machine-config API URL wins over the `siderolink.api=` kernel arg, and the same holds for the events sink. So the old `omni.ucdialplans.com` kernel arg left on GRUB-booted nodes is ignored once the config loads. Moving them to `grubUseUKICmdline` is not needed for joining. Only an unallocated machine in maintenance mode (no machine config) still depends on the old name.

## Tried and ruled out
- Starting the old Omni again on the old host: two instances on one state would fight over nodes.

## Watch out
- After a move, nodes drop one by one. Omni contacts each node first from the NAS address and the tunnel follows it there; a node that reconnects from scratch looks the hostname up again and goes dark. A few nodes still working proves nothing about inbound UDP.
- A node booted in UKI mode ignores `install.extraKernelArgs`, so `net.ifnames=0` does not apply and its NIC keeps a predictable name (nuc3: enp3s0). A control-plane patch keyed on `interface: eth0` then silently skips it, VIP included. Prefer `deviceSelector` over interface names.
- Through Traefik, Omni shows Traefik's internal address (172.30.0.2) as every node's last endpoint. That is expected.

In the code:
- Traefik: `--entryPoints.omni-wg.address=:50180/udp`, `--entryPoints.omni-wg.udp.timeout=300s`; Omni labels `traefik.udp.routers.omni-wg.*` with service port 50180.
- Talos precedence: internal/app/machined/pkg/controllers/siderolink/config.go `apiEndpoint()` (read on main, not the v1.14 tag).
- Audit: `omnictl get redactedclustermachineconfig <id> -o yaml` (pushed SideroLinkConfig/EventSinkConfig/KmsgLogConfig), `omnictl get machinestatus -o json` (`network.networklinks[].linuxname`, schematic), `omnictl get clustermachinestatus` (configuptodate), `omnictl get links`.
- patches/controlplane.yaml uses `interface: eth0` with VIP 192.168.1.11.
