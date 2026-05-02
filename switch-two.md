### /etc/network/interfaces
```
##############################################
# switch-two — Mellanox Cumulus Linux
# ASN 65000 | VTEP 10.10.254.6/32
# Anycast GW 10.10.200.1/22 | VNI 200
##############################################

auto lo
iface lo inet loopback
    address 10.10.254.6/32
    address fd00:254::6/128

# -----------------------------------------------
# Underlay P2P links to KVM hosts (MTU 9216)
# -----------------------------------------------
auto swp1
iface swp1
    # Connected to kvm-one eth1 (BACKUP uplink)
    mtu 9216
    address 0.0.0.0/0
    ipv6-addrgenmode eui64

auto swp2
iface swp2
    # Connected to kvm-two eth1 (BACKUP uplink)
    mtu 9216
    address 0.0.0.0/0
    ipv6-addrgenmode eui64

# -----------------------------------------------
# Inter-switch link to switch-one (for iBGP EVPN)
# -----------------------------------------------
auto swp3
iface swp3
    # Connected to switch-one swp3
    mtu 9216
    address 0.0.0.0/0
    ipv6-addrgenmode eui64

# -----------------------------------------------
# VXLAN NVE interface (VTEP)
# -----------------------------------------------
auto vxlan200
iface vxlan200
    vxlan-id 200
    vxlan-local-tunnelip 10.10.254.6
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
    # Same anycast MAC as switch-one
    hwaddress 44:38:39:FF:00:C8
    address fd00:200::1/64
    mtu 9216


##############################################
# /etc/frr/frr.conf
##############################################

frr version 8.5
frr defaults datacenter
hostname switch-two
log syslog informational
ip forwarding
ipv6 forwarding
service integrated-vtysh-config
!
bfd
 profile fast-bfd
  transmit-interval 100
  receive-interval 100
  detect-multiplier 3
 !
!
interface lo
 ip address 10.10.254.6/32
 ipv6 address fd00:254::6/128
!
interface swp1
 description kvm-one BACKUP uplink
 no ip address
 ipv6 nd ra-interval 10
 no ipv6 nd suppress-ra
 bfd profile fast-bfd
 mtu 9216
!
interface swp2
 description kvm-two BACKUP uplink
 no ip address
 ipv6 nd ra-interval 10
 no ipv6 nd suppress-ra
 bfd profile fast-bfd
 mtu 9216
!
interface swp3
 description switch-one inter-switch link
 no ip address
 ipv6 nd ra-interval 10
 no ipv6 nd suppress-ra
 bfd profile fast-bfd
 mtu 9216
!
router bgp 65000
 bgp router-id 10.10.254.6
 bgp bestpath as-path multipath-relax
 bgp bestpath compare-routerid
 no bgp ebgp-requires-policy
!
 neighbor kvm-hosts peer-group
 neighbor kvm-hosts remote-as external
 neighbor kvm-hosts bfd profile fast-bfd
 neighbor kvm-hosts capability extended-nexthop
 neighbor kvm-hosts timers 3 9
 neighbor kvm-hosts timers connect 5
 neighbor swp1 interface peer-group kvm-hosts
 neighbor swp2 interface peer-group kvm-hosts
!
 neighbor switch-one-evpn peer-group
 neighbor switch-one-evpn remote-as 65000
 neighbor switch-one-evpn update-source lo
 neighbor switch-one-evpn bfd profile fast-bfd
 neighbor switch-one-evpn timers 3 9
 neighbor 10.10.254.5 peer-group switch-one-evpn
!
 address-family ipv4 unicast
  redistribute connected
  neighbor kvm-hosts activate
  neighbor kvm-hosts route-map ALLOW-ALL in
  neighbor kvm-hosts route-map ALLOW-ALL out
  neighbor 10.10.254.5 activate
  neighbor 10.10.254.5 next-hop-self
  maximum-paths 2
  maximum-paths ibgp 2
 exit-address-family
!
 address-family l2vpn evpn
  neighbor kvm-hosts activate
  neighbor 10.10.254.5 activate
  neighbor 10.10.254.5 route-reflector-client
  advertise-all-vni
  advertise-svi-ip
  vni 200
   rd 10.10.254.6:200
   route-target import 65000:200
   route-target export 65000:200
  exit-vni
 exit-address-family
!
route-map ALLOW-ALL permit 10
!
vxlan vni 200
 vxlan source interface lo
 vxlan udp-port 4789
!

##############################################
# /etc/sysctl.d/99-network-tuning.conf
##############################################

net.ipv4.ip_forward = 1
net.ipv6.conf.all.forwarding = 1
net.ipv4.conf.all.arp_ignore = 2
net.ipv4.conf.all.arp_announce = 2
net.ipv4.conf.all.rp_filter = 0
net.ipv4.conf.default.rp_filter = 0
net.core.rmem_max = 134217728
net.core.wmem_max = 134217728
net.core.rmem_default = 33554432
net.core.wmem_default = 33554432
net.ipv4.tcp_rmem = 4096 87380 134217728
net.ipv4.tcp_wmem = 4096 87380 134217728
net.ipv4.neigh.default.gc_thresh1 = 2048
net.ipv4.neigh.default.gc_thresh2 = 4096
net.ipv4.neigh.default.gc_thresh3 = 8192
net.ipv6.neigh.default.gc_thresh1 = 2048
net.ipv6.neigh.default.gc_thresh2 = 4096
net.ipv6.neigh.default.gc_thresh3 = 8192
net.bridge.bridge-nf-call-iptables = 0
net.bridge.bridge-nf-call-ip6tables = 0
net.bridge.bridge-nf-call-arptables = 0
```
