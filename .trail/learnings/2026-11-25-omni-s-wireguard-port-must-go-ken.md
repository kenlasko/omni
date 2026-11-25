---
title: "Omni's WireGuard port must go through a Traefik UDP entrypoint, because the Omni hostname points at Traefik's own LAN address"
author: Ken
date: 2026-11-25
branch: main
cost: { minutes: 1, tokens: 7476 }
id: l-e4cc7253
scope: repo
status: active
replaces: l-4a25ef0f
src: agent
---

## Decided
- `omni.laskonet.com` resolves to Traefik's own LAN address on the QNAP qnet network (192.168.1.100), not the NAS (192.168.1.3). The Omni container's `50180/udp` publish lands on the NAS address, so nodes that looked the name up sent their WireGuard handshakes to Traefik and they were dropped. Fixed 2026-11-25 with a Traefik UDP entrypoint on `:50180/udp` and a UDP router/service to the Omni container on port 50180. All cluster nodes reconnected within a minute, without reboots.
- Raise the entrypoint's `udp.timeout` (300s). Traefik's default of 3s expires sessions between Talos keepalives, and Omni's replies are then dropped.
- Nodes still join through the `siderolink.api` kernel arg from install time, which names `omni.ucdialplans.com`. Keep that name in local DNS and on Traefik's 8090 router (with a cert) until the kernel args are changed. Changing them needs `machine.install.grubUseUKICmdline: true` plus an upgrade (it cannot coexist with `install.extraKernelArgs`; move `net.ifnames=0` into Omni's kernel args, or `eth0` and the VIP break).

## Tried and ruled out
- Starting the old Omni again on the old host: two instances on one state would fight over nodes.
- A separate WireGuard hostname pointing at the NAS address also works and keeps real node addresses in Omni, but needs new DNS records. Traefik matches how 8090 and 8100 are already routed.

## Watch out
- After a move, nodes drop one by one. Omni contacts each node first from the NAS address and the tunnel follows it; a node that reconnects from scratch looks the hostname up again and goes dark. A few nodes still working proves nothing about inbound UDP.
- Through Traefik, Omni shows Traefik's internal address (172.30.0.2) as every node's last endpoint. That is expected.
- The WAN UDP 50180 forward on the router must point at Traefik's address for remote nodes.

In the code:
- Traefik: `--entryPoints.omni-wg.address=:50180/udp`, `--entryPoints.omni-wg.udp.timeout=300s`. Omni labels: `traefik.udp.routers.omni-wg.entrypoints=omni-wg`, `...routers.omni-wg.service=omni-wg`, `traefik.udp.services.omni-wg.loadbalancer.server.port=50180`. Check via the Traefik API `/api/udp/routers`.
- Diagnose on the NAS: `docker run --rm --net=host --cap-add NET_ADMIN --cap-add NET_RAW nicolaka/netshoot timeout 30 tcpdump -lnni any udp port 50180` (and `--net=container:omni`); a `P` flag means the frame was not for the NAS's MAC. Container Station's docker: `$(getcfg container-station Install_Path -f /etc/config/qpkg.conf)/bin`.
- `omnictl get links`: a new nodepublickey with connected:false means the API was reached but WireGuard was not.
