---
title: "Omni's WireGuard endpoint must point at the NAS itself, not Traefik's address, and the old omni.ucdialplans.com name must keep working"
author: Ken
date: 2026-11-25
branch: main
cost: { minutes: 18, tokens: 25486 }
id: l-4a25ef0f
scope: repo
status: active
replaces: l-98d4e592
src: agent
touches:
  - patches/controlplane.yaml
  - patches/worker-longhorn.yaml
---

## Decided
- Nodes join through the `siderolink.api` kernel arg from install time, which still names `omni.ucdialplans.com`. Keep that name in local DNS and on Traefik's 8090 router (with a cert) until the kernel args are changed. Changing them needs `machine.install.grubUseUKICmdline: true` plus an upgrade (it cannot coexist with `install.extraKernelArgs`; move `net.ifnames=0` into Omni's kernel args or `eth0` and the VIP break).
- WireGuard (UDP 50180) is published by the Omni container on the NAS's own address, 192.168.1.3. `omni.laskonet.com` resolves to 192.168.1.100, which is Traefik on the QNAP qnet network and has no UDP listener. Give WireGuard its own name pointing at 192.168.1.3 (public: CNAME to the WAN host, DNS-only), forward WAN UDP 50180 to 192.168.1.3, and set Omni's WireGuard advertised address to that name.

## Tried and ruled out
- Starting the old Omni again on the old host: two instances on one state would fight over nodes.
- Proxying UDP 50180 through Traefik: Traefik's UDP sessions expire quickly, and every node would appear to come from Traefik's address.

## Watch out
- The symptom is nodes dropping one by one after a move. Omni first contacts each node from the NAS address and the tunnel follows it there; a node that reconnects from scratch looks the hostname up again, gets Traefik's address, and goes dark. A few nodes still working proves nothing about inbound UDP.
- In tcpdump on the NAS, a `P` direction flag on packets to .100 means they were not addressed to the NAS's own MAC.

In the code:
- Diagnose on the NAS: `docker run --rm --net=host --cap-add NET_ADMIN --cap-add NET_RAW nicolaka/netshoot timeout 30 tcpdump -lnni any udp port 50180`, and the same with `--net=container:omni`. Container Station's docker: `$(getcfg container-station Install_Path -f /etc/config/qpkg.conf)/bin`.
- `omnictl get links` (Link gets a new nodepublickey but stays connected:false = API reached, WireGuard not), `omnictl get kernelargsstatus <id> -o yaml`, `omnictl get connectionparams`.
- Compose flag: `--siderolink-wireguard-advertised-addr`. Kernel-arg patches: patches/controlplane.yaml, patches/worker-longhorn.yaml.
