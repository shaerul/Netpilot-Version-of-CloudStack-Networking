```bash
# =============================================================
# kvm-one — Ubuntu 24.04 KVM Hypervisor
# ASN 65010 | Lo: 10.10.254.10/32 | VTEP: 10.10.254.10
# IP: 10.10.200.10/22 | Active: eth0→sw1 | Backup: eth1→sw2
# =============================================================

# ─────────────────────────────────────────────
# FILE: /etc/netplan/01-netcfg.yaml
# ─────────────────────────────────────────────
# NOTE: Replace eth0/eth1 with your actual interface names
# (e.g. enp3s0f0, eno1). Run: ip link show

network:
  version: 2
  renderer: networkd

  ethernets:
    eth0:
      # Active uplink → switch-one swp1
      dhcp4: false
      dhcp6: false
      mtu: 9216
      # No IP — FRR uses IPv6 link-local for BGP unnumbered
      accept-ra: true

    eth1:
      # Backup uplink → switch-two swp1
      dhcp4: false
      dhcp6: false
      mtu: 9216
      accept-ra: true

  # Loopback with VTEP IP
  # Netplan doesn't manage lo directly for extra addresses;
  # use a systemd-networkd .network file instead (see below).
  # Alternatively add via /etc/rc.local or a oneshot service:
  #   ip addr add 10.10.254.10/32 dev lo

  # VXLAN interface for VNI 200
  tunnels:
    vxlan200:
      mode: vxlan
      id: 200
      local: 10.10.254.10
      # Remote VTEP IPs are learned via BGP EVPN — no static config needed
      port: 4789
      mtu: 9216

  bridges:
    br-vxlan200:
      interfaces: [vxlan200]
      mtu: 9216
      parameters:
        stp: false
      addresses:
        # Host IP on tenant network (also acts as default route source)
        - 10.10.200.10/22
      routes:
        - to: default
          via: 10.10.200.1   # Anycast gateway on leaf switches

# ─────────────────────────────────────────────
# FILE: /etc/systemd/network/10-loopback.network
# (Assign VTEP loopback IP via systemd-networkd)
# ─────────────────────────────────────────────
#
# [Match]
# Name=lo
#
# [Network]
# Address=127.0.0.1/8
# Address=::1/128
# Address=10.10.254.10/32
#
# ─────────────────────────────────────────────
# FILE: /etc/frr/frr.conf
# ─────────────────────────────────────────────

frr version 8.4
frr defaults datacenter
hostname kvm-one
log syslog informational
service integrated-vtysh-config
!
! ── Loopback / VTEP ───────────────────────────────────────────
interface lo
 ip address 10.10.254.10/32
!
! ── Uplink interfaces (unnumbered — IPv6 link-local peering) ──
interface eth0
 ipv6 nd ra-interval 5
 no ipv6 nd suppress-ra
 description active-uplink-to-switch-one
!
interface eth1
 ipv6 nd ra-interval 5
 no ipv6 nd suppress-ra
 description backup-uplink-to-switch-two
!
bfd
 profile FAST
  receive-interval 100
  transmit-interval 100
  detect-multiplier 3
!
! ── BGP underlay + EVPN overlay ───────────────────────────────
router bgp 65010
 bgp router-id 10.10.254.10
 bgp bestpath as-path multipath-relax
 bgp bestpath compare-routerid
 no bgp default ipv4-unicast
 !
 neighbor LEAF peer-group
 neighbor LEAF remote-as external
 neighbor LEAF bfd profile FAST
 neighbor LEAF capability extended-nexthop
 neighbor LEAF timers 3 9
 neighbor LEAF timers connect 5
 !
 ! Active uplink — switch-one swp1
 neighbor eth0 interface peer-group LEAF
 neighbor eth0 description switch-one-active
 !
 ! Backup uplink — switch-two swp1
 ! BGP will prefer eth0 (active) and keep eth1 as standby;
 ! BFD on eth0 triggers sub-second failover to eth1
 neighbor eth1 interface peer-group LEAF
 neighbor eth1 description switch-two-backup
 !
 address-family ipv4 unicast
  neighbor LEAF activate
  redistribute connected
  ! Advertise only loopback (VTEP) and tenant prefix
  neighbor LEAF route-map EXPORT out
  neighbor LEAF route-map IMPORT in
  maximum-paths 2
 exit-address-family
 !
 address-family l2vpn evpn
  neighbor LEAF activate
  advertise-all-vni
  advertise-svi-ip
 exit-address-family
!
! ── Route maps ────────────────────────────────────────────────
ip prefix-list LOOPBACKS seq 10 permit 10.10.254.10/32
ip prefix-list TENANT seq 10 permit 10.10.200.0/22 le 32
!
route-map EXPORT permit 10
 match ip address prefix-list LOOPBACKS
!
route-map EXPORT permit 20
 match ip address prefix-list TENANT
!
route-map IMPORT permit 10
!
line vty
!

# ─────────────────────────────────────────────
# FILE: /etc/sysctl.d/99-network-tune.conf
# ─────────────────────────────────────────────
# (See dedicated sysctl section)
```
