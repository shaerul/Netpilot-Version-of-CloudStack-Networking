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
