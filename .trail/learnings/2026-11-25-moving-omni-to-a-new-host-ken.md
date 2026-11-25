---
title: "Moving Omni to a new host needs the old omni.ucdialplans.com name kept working, because every node's kernel args still join through it"
author: Ken
date: 2026-11-25
branch: main
cost: { minutes: 15, tokens: 95333 }
id: l-98d4e592
scope: repo
status: active
src: agent
touches:
  - patches/controlplane.yaml
  - patches/worker-longhorn.yaml
---

## Decided
- The Talos nodes join Omni through the `siderolink.api` kernel arg baked in at install time, and it still names the OLD domain `omni.ucdialplans.com`, not `omni.laskonet.com`. Talos upgrades carry the arg forward, so changing Omni's advertised URLs or the laskonet DNS does not reach the nodes.
- When Omni moves host, point local DNS for `omni.ucdialplans.com` at the new host too, and have Traefik's machine-API router (port 8090) accept that host name with a valid cert. Keep this alias until every node's kernel arg has been changed.
- To change the arg for good, Omni's per-machine kernel-args feature must be unlocked first: set `machine.install.grubUseUKICmdline: true` (it cannot be combined with `install.extraKernelArgs`), then upgrade. `net.ifnames=0` must then move into Omni's kernel args, or interfaces get renamed away from `eth0` and the control-plane VIP patch breaks.

## Tried and ruled out
- Adding a second `siderolink.api=` to `extraKernelArgs`: it is unclear which value Talos would use.
- Running the old Omni again on the old host as a fallback: two Omni instances sharing one state would fight over the nodes.

## Watch out
- After a move, nodes keep working for a while and then drop one by one. Omni restores each node's last WireGuard endpoint and contacts it first, so a connection the node has to start itself is what fails later. A node that still answers is not proof that the new host accepts UDP 50180 from outside.
- A node whose Link gets a new `nodepublickey` but keeps `connected: false` and its old `lastendpoint` did reach the API, but its WireGuard handshake on UDP 50180 never arrived. Open as of 2026-11-25: check for that traffic on the new host with tcpdump, and check that the router's 50180/udp forward and DHCP DNS (public DNS sends `omni.laskonet.com` to the WAN IP) no longer point at the old host.
- In Traefik labels, setting `tls.domains[0].main` twice keeps only the last value; use `tls.domains[0].sans` for the second name.

In the code:
- Old host 192.168.1.17 (Traefik left running, no Omni); new host 192.168.1.100.
- Check with `omnictl get links`, `omnictl get kernelargsstatus <machine-id> -o yaml` (shows the current cmdline and why kernel-args editing is blocked), and `omnictl get connectionparams`.
- Kernel-arg patches: patches/controlplane.yaml and patches/worker-longhorn.yaml (`install.extraKernelArgs: [net.ifnames=0]`).
