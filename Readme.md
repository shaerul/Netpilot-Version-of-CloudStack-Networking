### The Problem/Requirement
---
I want to connect two KVM hosts running Ubuntu 24.04, where each host has two Ethernet interfaces connected to two separate Mellanox leaf switches for redundancy. On kvm-one, one interface (enp94s0f0np0 ) connects to switch-one (port swp1) and the other (enp94s0f1np1) connects to switch-two (port swp1). On kvm-two, one interface connects to switch-one (port swp2) and the other connects to switch-two (port swp2). The two hosts will communicate with each other over VXLAN using VNI 200 through these leaf switches. The design ensures full redundancy, meaning if one switch or link fails, the other will take over, and each host uses one active link and one backup link. This setup avoids any Layer 2 technologies like MLAG or LACP and instead uses a pure Layer 3 design with BGP and BGP EVPN for both underlay and overlay networks, along with IPv6 and jumbo frames (MTU 9216) for better performance. Both switches will have an anycast gateway with IP 10.10.200.1/22, while the KVM hosts will use IP addresses 10.10.200.10/22 and 10.10.200.11/22. Loopback interfaces will be used for VTEP endpoints, with IPs 10.10.254.5/32 and 10.10.254.6/32 on the switches, and 10.10.254.10/32 and 10.10.254.11/32 on the hosts. The switches will use BGP ASN 65000, and the KVM hosts will use ASN 65010 and 65011.

Now I will add another device, a firewall host running Ubuntu 24.04 with two Ethernet interfaces, enp94s0f0np0 and enp94s0f1np1. These two interfaces will be connected to switch-one and switch-two on port swp48 for redundancy, where one link will be active and the other will act as a backup. The firewall will use BGP ASN 65001 and have a loopback IP address of 10.10.254.1/32. It will also use the anycast gateway address 10.10.200.1/22 as its bridge IP. The WAN interface will obtain its IP address dynamically using DHCPv4 from the uplink devices. This firewall will act as the default gateway for the entire network and will function mainly as a packet-filtering firewall.

Generate the architectural diagram and generate he configurations that will include `/etc/network/interfaces` for the Mellanox switches, `/etc/netplan/01-netcfg.yaml` for the Ubuntu hosts, and `frr.conf` for all devices, with BFD enabled for fast failure detection and quick convergence, along with kernel tuning using `sysctl.conf` at the end.

---

Everything is generated. Here's a summary of the full design and key implementation notes:

---

## Architecture Summary

### Design Principles Applied
| Principle | Implementation |
|---|---|
| **No L2 redundancy (no MLAG/LACP)** | Pure BGP unnumbered eBGP on every link — each link is an independent L3 path |
| **Active/Backup per host** | BGP `compare-routerid` + BFD sub-100ms detection triggers automatic failover |
| **Anycast Gateway** | Both switches share `10.10.200.1/22` with identical MAC `44:38:39:ff:00:01` — VMs see one consistent gateway regardless of which switch is active |
| **VXLAN/EVPN overlay** | VNI 200 tunneled over loopback-to-loopback paths; VTEP IPs in `10.10.254.0/24` |
| **IPv6 underlay** | BGP unnumbered uses IPv6 link-local addresses (RFC 5549) — no IPv4 point-to-point subnets needed on fabric links |
| **MTU 9216 end-to-end** | Set on all swp/eth interfaces, bridges, and VXLAN tunnels |
| **BFD** | Profile `FAST`: 100ms Tx/Rx, multiplier 3 → **300ms failure detection** on all BGP sessions |

### IP Addressing Reference
| Device | Loopback (VTEP) | Tenant IP | ASN |
|---|---|---|---|
| fw-host | 10.10.254.1/32 | 10.10.200.1/22 (bridge) | 65001 |
| switch-one | 10.10.254.5/32 | 10.10.200.1/22 (SVI) | 65000 |
| switch-two | 10.10.254.6/32 | 10.10.200.1/22 (SVI) | 65000 |
| kvm-one | 10.10.254.10/32 | 10.10.200.10/22 | 65010 |
| kvm-two | 10.10.254.11/32 | 10.10.200.11/22 | 65011 |
| Peerlink | — | 10.10.252.1/30 ↔ .2/30 | — |

### Key Implementation Notes
1. **Inter-switch iBGP peerlink** uses `swp49`+`swp50` bonded — this is the only bond in the design (pure LACP between the two switches for the control plane link, not tenant-facing)
2. **Firewall as default gateway**: fw-host redistributes the `0.0.0.0/0` learned from WAN DHCP into BGP toward the fabric — all KVM hosts get their default route via BGP
3. **Loopback IPs on Ubuntu hosts**: Netplan can't manage extra loopback addresses; use `/etc/systemd/network/10-loopback.network` (template included in configs)
4. **WAN interface on fw-host**: Named `wan0` as placeholder — replace with your actual interface name (`ip link show` to find it)
5. **nftables on fw-host**: Stateful firewall — allows established/related, BFD (UDP 3784/3785), BGP (TCP 179), VXLAN (UDP 4789), and NAT masquerade for tenant-to-WAN traffic
6. **sysctl**: Apply `sysctl --system` on all devices; `nf_conntrack` lines can be omitted on switches and KVM hosts if you don't use NAT there

All 12 configuration blocks are generated and saved. Here's a summary of everything produced:

---

## Architecture Summary

### Design Principles
- **Pure L3 fabric** — no MLAG, no LACP, no STP on uplinks
- **BGP unnumbered** on all host↔switch links using IPv6 link-local (no point-to-point IPs to manage)
- **Active/backup redundancy** via BGP `local-preference`: active links get LP 200, backup links get LP 100 — set as inbound route-maps on each host/firewall
- **BFD profile `fast-link`**: 100ms Tx/Rx, 3× multiplier → **300ms detection** on any link/node failure
- **BGP EVPN** with switch-one acting as **Route Reflector** for all EVPN peers; switch-two is RR client
- **Anycast gateway**: both switches share IP `10.10.200.1/22` and MAC `44:38:39:ff:00:01` on `vlan200`/`vxlan200`

---

### Config Index

| Config Card | File | Device(s) |
|---|---|---|
| `switch-one` | `/etc/network/interfaces` | switch-one |
| `switch-two` | `/etc/network/interfaces` | switch-two |
| `kvm-one` | `/etc/netplan/01-netcfg.yaml` + systemd-networkd | kvm-one |
| `kvm-two` | `/etc/netplan/01-netcfg.yaml` + systemd-networkd | kvm-two |
| `firewall` | `/etc/netplan/01-netcfg.yaml` | firewall |
| `switch-one-frr` | `/etc/frr/frr.conf` | switch-one |
| `switch-two-frr` | `/etc/frr/frr.conf` | switch-two |
| `kvm-one-frr` | `/etc/frr/frr.conf` | kvm-one |
| `kvm-two-frr` | `/etc/frr/frr.conf` | kvm-two |
| `firewall-frr` | `/etc/frr/frr.conf` | firewall |
| `switches-sysctl` | `/etc/sysctl.d/99-fabric.conf` | both switches |
| `hosts-sysctl` | `/etc/sysctl.d/99-fabric.conf` + `99-firewall.conf` | KVM hosts + firewall |

---

### Key Notes & Action Items

1. **WAN interface name** — update `enp1s0` in `firewall`'s netplan to your actual WAN NIC (`ip link show` to find it)

2. **DHCP → FRR default route** — create `/etc/dhcp/dhclient-exit-hooks.d/frr-default`:
   ```bash
   if [ "$reason" = BOUND ] || [ "$reason" = RENEW ]; then
     vtysh -c "conf t" -c "ip route 0.0.0.0/0 $router" -c "end" -c "write"
   fi
   ```

3. **FRR daemons** — ensure `/etc/frr/daemons` has `bgpd=yes` and `bfdd=yes` on all devices

4. **systemd-networkd VXLAN** — the `.netdev`/`.network` files shown as comments in the kvm netplan cards should be created as **actual files** in `/etc/systemd/network/`

5. **nftables** — a production nftables skeleton is included as comments in `firewall-frr`; save it to `/etc/nftables.conf` and enable with `systemctl enable --now nftables`

6. **Apply order**: sysctl → interfaces/netplan → FRR → verify with `vtysh -c 'show bgp summary'` and `vtysh -c 'show bfd peers'`
