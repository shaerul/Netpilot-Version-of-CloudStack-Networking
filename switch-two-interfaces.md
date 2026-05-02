```
# ============================================================
# FILE: /etc/network/interfaces — switch-two (Cumulus Linux)
# ASN: 65000 | Loopback: 10.10.254.6/32 | VTEP: 10.10.254.6
# Anycast GW: 10.10.200.1/22 (MAC 44:38:39:ff:00:01) | VNI 200
# Design: Pure L3 BGP underlay, BGP EVPN overlay, no MLAG/LACP
# MTU: 9216 on all fabric interfaces
# ============================================================

# ----------------------------------------------------------
# Loopback — router-id, VTEP source, BGP update-source
# ----------------------------------------------------------
auto lo
iface lo inet loopback
    address 10.10.254.6/32

# ----------------------------------------------------------
# swp1 — Uplink to kvm-one enp94s0f1np1 (BACKUP path)
# BGP unnumbered peer, ASN 65010
# ----------------------------------------------------------
auto swp1
iface swp1
    mtu 9216

# ----------------------------------------------------------
# swp2 — Uplink to kvm-two enp94s0f1np1 (BACKUP path)
# BGP unnumbered peer, ASN 65011
# ----------------------------------------------------------
auto swp2
iface swp2
    mtu 9216

# ----------------------------------------------------------
# swp48 — Uplink to firewall enp94s0f1np1 (BACKUP path)
# BGP unnumbered peer, ASN 65001
# ----------------------------------------------------------
auto swp48
iface swp48
    mtu 9216

# ----------------------------------------------------------
# swp49 — Inter-switch P2P underlay link to switch-one swp49
# iBGP session between leaf switches for EVPN RR peering
# ----------------------------------------------------------
auto swp49
iface swp49
    mtu 9216

# ----------------------------------------------------------
# VLAN 200 — Tenant SVI (anycast gateway)
# IDENTICAL IP + MAC to switch-one — this is the anycast design
# ----------------------------------------------------------
auto vlan200
iface vlan200
    address 10.10.200.1/22
    # Anycast MAC — MUST be identical to switch-one
    hwaddress 44:38:39:ff:00:01
    vlan-id 200
    vlan-raw-device bridge
    mtu 9216
    neigh-suppress yes

# ----------------------------------------------------------
# VxLAN 200 — VXLAN tunnel for VNI 200
# local-tunnelip = this switch's loopback (VTEP source)
# ----------------------------------------------------------
auto vxlan200
iface vxlan200
    vxlan-id 200
    vxlan-local-tunnelip 10.10.254.6
    mtu 9216
    bridge-access 200
    bridge-learning off
    mstpctl-bpduguard yes
    mstpctl-portbpdufilter yes

# ----------------------------------------------------------
# Bridge — L2 domain for VXLAN-bridged traffic
# ----------------------------------------------------------
auto bridge
iface bridge
    bridge-vlan-aware yes
    bridge-ports vxlan200
    bridge-vids 200
    bridge-pvid 1
    mtu 9216

# ============================================================
# NOTE: Apply with: sudo ifreload -a
# Verify: net show interface | net show bridge macs
# ============================================================
```
