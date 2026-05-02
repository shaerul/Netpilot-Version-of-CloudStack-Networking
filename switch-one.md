### /etc/network/interfaces
```
##############################################
# switch-one — Mellanox Cumulus Linux
# ASN 65000 | VTEP 10.10.254.5/32
# Anycast GW 10.10.200.1/22 | VNI 200
##############################################

auto lo
iface lo inet loopback
    address 10.10.254.5/32
    # IPv6 loopback (used for iBGP EVPN peering with switch-two)
    address fd00:254::5/128

# -----------------------------------------------
# Underlay P2P links to KVM hosts (MTU 9216)
# -----------------------------------------------
auto swp1
iface swp1
    # Connected to kvm-one eth0 (ACTIVE uplink)
    mtu 9216
    # IPv4 unnumbered — borrows loopback IP
    address 0.0.0.0/0
    # Enable IPv6 link-local for BGP unnumbered
    ipv6-addrgenmode eui64

auto swp2
iface swp2
    # Connected to kvm-two eth0 (ACTIVE uplink)
    mtu 9216
    address 0.0.0.0/0
    ipv6-addrgenmode eui64

# -----------------------------------------------
# Inter-switch link to switch-two (for iBGP EVPN)
# -----------------------------------------------
auto swp3
iface swp3
    # Connected to switch-two swp3
    mtu 9216
    address 0.0.0.0/0
    ipv6-addrgenmode eui64

# -----------------------------------------------
# VXLAN NVE interface (VTEP)
# -----------------------------------------------
auto vxlan200
iface vxlan200
    vxlan-id 200
    vxlan-local-tunnelip 10.10.254.5
    bridge-access 200
    bridge-learning off
    mtu 9216

# -----------------------------------------------
# Bridge for VNI 200 / VLAN 200 (anycast GW)
# -----------------------------------------------
auto bridge
iface bridge
    bridge-vlan-aware yes
    bridge-ports swp1 swp2 vxlan200
    bridge-vids 200
    bridge-pvid 1
    mtu 9216

auto bridge.200
iface bridge.200
    address 10.10.200.1/22
    # Anycast MAC — same on both switches for seamless failover
    hwaddress 44:38:39:FF:00:C8
    # IPv6 anycast GW
    address fd00:200::1/64
    mtu 9216


##############################################
# /etc/frr/frr.conf
##############################################

frr version 8.5
frr defaults datacenter
hostname switch-one
log syslog informational
ip forwarding
ipv6 forwarding
service integrated-vtysh-config
!
# -----------------------------------------------
# BFD — fast failure detection (100ms x3 = 300ms)
# -----------------------------------------------
bfd
 profile fast-bfd
  transmit-interval 100
  receive-interval 100
  detect-multiplier 3
 !
!
# -----------------------------------------------
# Loopback
# -----------------------------------------------
interface lo
 ip address 10.10.254.5/32
 ipv6 address fd00:254::5/128
!
# -----------------------------------------------
# Underlay P2P — BGP unnumbered (swp1/swp2)
# -----------------------------------------------
interface swp1
 description kvm-one ACTIVE uplink
 no ip address
 ipv6 nd ra-interval 10
 no ipv6 nd suppress-ra
 bfd profile fast-bfd
 mtu 9216
!
interface swp2
 description kvm-two ACTIVE uplink
 no ip address
 ipv6 nd ra-interval 10
 no ipv6 nd suppress-ra
 bfd profile fast-bfd
 mtu 9216
!
interface swp3
 description switch-two inter-switch link
 no ip address
 ipv6 nd ra-interval 10
 no ipv6 nd suppress-ra
 bfd profile fast-bfd
 mtu 9216
!
# -----------------------------------------------
# BGP — Underlay + EVPN overlay
# -----------------------------------------------
router bgp 65000
 bgp router-id 10.10.254.5
 bgp bestpath as-path multipath-relax
 bgp bestpath compare-routerid
 no bgp ebgp-requires-policy
!
 # ---- eBGP peers: KVM hosts (BGP unnumbered) ----
 neighbor kvm-hosts peer-group
 neighbor kvm-hosts remote-as external
 neighbor kvm-hosts bfd profile fast-bfd
 neighbor kvm-hosts capability extended-nexthop
 neighbor kvm-hosts timers 3 9
 neighbor kvm-hosts timers connect 5
 neighbor swp1 interface peer-group kvm-hosts
 neighbor swp2 interface peer-group kvm-hosts
!
 # ---- iBGP peer: switch-two (EVPN control plane) ----
 neighbor switch-two-evpn peer-group
 neighbor switch-two-evpn remote-as 65000
 neighbor switch-two-evpn update-source lo
 neighbor switch-two-evpn bfd profile fast-bfd
 neighbor switch-two-evpn timers 3 9
 neighbor 10.10.254.6 peer-group switch-two-evpn
!
 # ---- IPv4 unicast (underlay) ----
 address-family ipv4 unicast
  redistribute connected
  neighbor kvm-hosts activate
  neighbor kvm-hosts route-map ALLOW-ALL in
  neighbor kvm-hosts route-map ALLOW-ALL out
  neighbor 10.10.254.6 activate
  neighbor 10.10.254.6 next-hop-self
  maximum-paths 2
  maximum-paths ibgp 2
 exit-address-family
!
 # ---- L2VPN EVPN overlay ----
 address-family l2vpn evpn
  neighbor kvm-hosts activate
  neighbor 10.10.254.6 activate
  neighbor 10.10.254.6 route-reflector-client
  advertise-all-vni
  advertise-svi-ip
  vni 200
   rd 10.10.254.5:200
   route-target import 65000:200
   route-target export 65000:200
  exit-vni
 exit-address-family
!
route-map ALLOW-ALL permit 10
!
# -----------------------------------------------
# VXLAN VTEP
# -----------------------------------------------
vxlan vni 200
 vxlan source interface lo
 vxlan udp-port 4789
!

##############################################
# /etc/sysctl.d/99-network-tuning.conf
##############################################

# ---- Forwarding ----
net.ipv4.ip_forward = 1
net.ipv6.conf.all.forwarding = 1

# ---- ARP / NDP tuning ----
net.ipv4.conf.all.arp_ignore = 2
net.ipv4.conf.all.arp_announce = 2
net.ipv4.conf.all.rp_filter = 0
net.ipv4.conf.default.rp_filter = 0

# ---- VXLAN / Jumbo frame socket buffers ----
net.core.rmem_max = 134217728
net.core.wmem_max = 134217728
net.core.rmem_default = 33554432
net.core.wmem_default = 33554432
net.ipv4.tcp_rmem = 4096 87380 134217728
net.ipv4.tcp_wmem = 4096 87380 134217728

# ---- Neighbour table (large-scale VXLAN) ----
net.ipv4.neigh.default.gc_thresh1 = 2048
net.ipv4.neigh.default.gc_thresh2 = 4096
net.ipv4.neigh.default.gc_thresh3 = 8192
net.ipv6.neigh.default.gc_thresh1 = 2048
net.ipv6.neigh.default.gc_thresh2 = 4096
net.ipv6.neigh.default.gc_thresh3 = 8192

# ---- Bridge / VXLAN EVPN ----
net.bridge.bridge-nf-call-iptables = 0
net.bridge.bridge-nf-call-ip6tables = 0
net.bridge.bridge-nf-call-arptables = 0
```
