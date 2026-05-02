```
##############################################
# /etc/netplan/01-netcfg.yaml
# kvm-one — ASN 65010 | VTEP 10.10.254.10/32
# NIC: enp94s0f0np0 = ACTIVE → switch-one swp1
#      enp94s0f1np1 = BACKUP → switch-two swp1
##############################################

network:
  version: 2
  renderer: networkd

  ethernets:
    enp94s0f0np0:   # ACTIVE uplink → switch-one swp1
      mtu: 9216
      dhcp4: false
      dhcp6: false
      accept-ra: true
      ipv6-privacy: false

    enp94s0f1np1:   # BACKUP uplink → switch-two swp1
      mtu: 9216
      dhcp4: false
      dhcp6: false
      accept-ra: true
      ipv6-privacy: false

  tunnels:
    vxlan200:
      mode: vxlan
      id: 200
      local: 10.10.254.10
      port: 4789
      mtu: 9216
      dhcp4: false
      dhcp6: false

  # NOTE: Loopback addresses are NOT manageable via netplan (single address limit).
  # Use the loopback-addrs.service systemd unit below for persistence.


##############################################
# /etc/systemd/system/loopback-addrs.service
# Install: systemctl enable --now loopback-addrs.service
##############################################

# [Unit]
# Description=Add extra loopback addresses for BGP/VXLAN
# After=network.target
# Before=frr.service
#
# [Service]
# Type=oneshot
# RemainAfterExit=yes
# ExecStart=/sbin/ip addr add 10.10.254.10/32 dev lo label lo:vtep || true
# ExecStart=/sbin/ip addr add 10.10.200.10/22 dev lo label lo:host || true
# ExecStart=/sbin/ip addr add fd00:254::10/128 dev lo || true
#
# [Install]
# WantedBy=multi-user.target


##############################################
# /etc/frr/frr.conf
# kvm-one — BGP EVPN host | active/backup via local-pref
##############################################

frr version 8.5
frr defaults datacenter
hostname kvm-one
log syslog informational
ip forwarding
ipv6 forwarding
service integrated-vtysh-config
!
# -----------------------------------------------
# BFD — 100ms x3 = 300ms failure detection
# -----------------------------------------------
bfd
 profile fast-bfd
  transmit-interval 100
  receive-interval 100
  detect-multiplier 3
 !
!
# -----------------------------------------------
# Interfaces
# -----------------------------------------------
interface lo
 ip address 10.10.254.10/32
 ip address 10.10.200.10/22
 ipv6 address fd00:254::10/128
!
interface enp94s0f0np0
 description ACTIVE uplink to switch-one swp1
 no ip address
 ipv6 nd ra-interval 10
 no ipv6 nd suppress-ra
 bfd profile fast-bfd
 mtu 9216
!
interface enp94s0f1np1
 description BACKUP uplink to switch-two swp1
 no ip address
 ipv6 nd ra-interval 10
 no ipv6 nd suppress-ra
 bfd profile fast-bfd
 mtu 9216
!
# -----------------------------------------------
# BGP — eBGP to both leaf switches
# Active path (switch-one): local-pref 200
# Backup path (switch-two): local-pref 100
# BFD drives sub-second failover on link/switch loss
# -----------------------------------------------
router bgp 65010
 bgp router-id 10.10.254.10
 bgp bestpath as-path multipath-relax
 bgp bestpath compare-routerid
 no bgp ebgp-requires-policy
!
 # ---- eBGP to switch-one (ACTIVE) via BGP unnumbered ----
 neighbor switch-one peer-group
 neighbor switch-one remote-as 65000
 neighbor switch-one bfd profile fast-bfd
 neighbor switch-one capability extended-nexthop
 neighbor switch-one timers 3 9
 neighbor switch-one timers connect 5
 neighbor enp94s0f0np0 interface peer-group switch-one
!
 # ---- eBGP to switch-two (BACKUP) via BGP unnumbered ----
 neighbor switch-two peer-group
 neighbor switch-two remote-as 65000
 neighbor switch-two bfd profile fast-bfd
 neighbor switch-two capability extended-nexthop
 neighbor switch-two timers 3 9
 neighbor switch-two timers connect 5
 neighbor enp94s0f1np1 interface peer-group switch-two
!
 # ---- IPv4 underlay ----
 address-family ipv4 unicast
  redistribute connected
  neighbor switch-one activate
  neighbor switch-one route-map RM-ACTIVE-IN in
  neighbor switch-one route-map ALLOW-ALL out
  neighbor switch-two activate
  neighbor switch-two route-map RM-BACKUP-IN in
  neighbor switch-two route-map ALLOW-ALL out
  maximum-paths 1
 exit-address-family
!
 # ---- L2VPN EVPN overlay ----
 address-family l2vpn evpn
  neighbor switch-one activate
  neighbor switch-two activate
  advertise-all-vni
  vni 200
   rd 10.10.254.10:200
   route-target import 65000:200
   route-target export 65000:200
  exit-vni
 exit-address-family
!
# -----------------------------------------------
# Route-maps: active/backup path preference
# -----------------------------------------------
route-map RM-ACTIVE-IN permit 10
 set local-preference 200
!
route-map RM-BACKUP-IN permit 10
 set local-preference 100
!
route-map ALLOW-ALL permit 10
!

##############################################
# /etc/sysctl.d/99-network-tuning.conf
##############################################

# ---- Forwarding ----
net.ipv4.ip_forward = 1
net.ipv6.conf.all.forwarding = 1

# ---- ARP tuning (VXLAN + anycast GW) ----
net.ipv4.conf.all.arp_ignore = 2
net.ipv4.conf.all.arp_announce = 2
net.ipv4.conf.all.rp_filter = 0
net.ipv4.conf.default.rp_filter = 0

# ---- Jumbo frame socket buffers (MTU 9216) ----
net.core.rmem_max = 134217728
net.core.wmem_max = 134217728
net.core.rmem_default = 33554432
net.core.wmem_default = 33554432
net.ipv4.tcp_rmem = 4096 87380 134217728
net.ipv4.tcp_wmem = 4096 87380 134217728

# ---- Neighbour / VXLAN MAC table scale ----
net.ipv4.neigh.default.gc_thresh1 = 2048
net.ipv4.neigh.default.gc_thresh2 = 4096
net.ipv4.neigh.default.gc_thresh3 = 8192
net.ipv6.neigh.default.gc_thresh1 = 2048
net.ipv6.neigh.default.gc_thresh2 = 4096
net.ipv6.neigh.default.gc_thresh3 = 8192

# ---- KVM / TUN/TAP / virtio performance ----
net.core.netdev_max_backlog = 250000
net.ipv4.tcp_mtu_probing = 1
net.core.somaxconn = 65535

# ---- Disable bridge netfilter (avoid double NAT on VM traffic) ----
net.bridge.bridge-nf-call-iptables = 0
net.bridge.bridge-nf-call-ip6tables = 0
net.bridge.bridge-nf-call-arptables = 0
```
