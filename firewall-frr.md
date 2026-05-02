```
# ============================================================
# FILE: /etc/frr/frr.conf — firewall
# ASN: 65001 | Router-ID: 10.10.254.1
# Active uplink: enp94s0f0np0 -> switch-one swp48 (local-pref 200)
# Backup uplink: enp94s0f1np1 -> switch-two swp48 (local-pref 100)
# Role: Default gateway for fabric; originates 0.0.0.0/0 via EVPN
# WAN: enp1s0 DHCP (update name to match hardware)
# ============================================================

frr version 8.5
frr defaults datacenter
hostname firewall
log syslog informational
log timestamp precision 3
service integrated-vtysh-config
!
bfd
  profile fast-link
    transmit-interval 100
    receive-interval 100
    detect-multiplier 3
    echo-mode
  !
!
interface lo
  ip address 10.10.254.1/32
  ipv6 address fd00:254::1/128
!
interface dummy0
  ip address 10.10.254.1/32
!
interface enp94s0f0np0
  description switch-one ACTIVE uplink
  no ipv6 nd suppress-ra
  ipv6 nd ra-interval 5
!
interface enp94s0f1np1
  description switch-two BACKUP uplink
  no ipv6 nd suppress-ra
  ipv6 nd ra-interval 5
!
interface brlan
  description LAN bridge - anycast GW for fabric
  ip address 10.10.200.1/22
!
# ----------------------------------------------------------
# Static default route — populated by DHCP client hook
# dhclient exit hook should run: ip route replace default via <GW> dev enp1s0
# FRR redistributes this into BGP as the default route for the fabric
# ----------------------------------------------------------
ip route 0.0.0.0/0 0.0.0.0
!
# ----------------------------------------------------------
# Route-maps for active/backup path preference
# ----------------------------------------------------------
route-map SET-LOCALPREF-HIGH permit 10
  set local-preference 200
!
route-map SET-LOCALPREF-LOW permit 10
  set local-preference 100
!
# Outbound route-map: tag default route so we can track it
route-map EXPORT-DEFAULT permit 10
  match ip address prefix-list DEFAULT-ONLY
  set community 65001:1000
!
route-map EXPORT-DEFAULT permit 20
!
# ----------------------------------------------------------
# Prefix lists
# ----------------------------------------------------------
ip prefix-list DEFAULT-ONLY seq 10 permit 0.0.0.0/0
ip prefix-list DEFAULT-ONLY seq 100 deny any
!
ip prefix-list FABRIC-PREFIXES seq 10 permit 10.10.254.0/24 le 32
ip prefix-list FABRIC-PREFIXES seq 20 permit 10.10.200.0/22
ip prefix-list FABRIC-PREFIXES seq 100 deny any
!
# ----------------------------------------------------------
# BGP
# ----------------------------------------------------------
router bgp 65001
  bgp router-id 10.10.254.1
  bgp bestpath as-path multipath-relax
  bgp bestpath compare-routerid
  no bgp ebgp-requires-policy
  no bgp network import-check
  !
  # Active path — switch-one via enp94s0f0np0
  neighbor enp94s0f0np0 interface remote-as 65000
  neighbor enp94s0f0np0 description switch-one-active
  neighbor enp94s0f0np0 bfd profile fast-link
  neighbor enp94s0f0np0 advertisement-interval 0
  neighbor enp94s0f0np0 timers 3 9
  neighbor enp94s0f0np0 timers connect 5
  !
  # Backup path — switch-two via enp94s0f1np1
  neighbor enp94s0f1np1 interface remote-as 65000
  neighbor enp94s0f1np1 description switch-two-backup
  neighbor enp94s0f1np1 bfd profile fast-link
  neighbor enp94s0f1np1 advertisement-interval 0
  neighbor enp94s0f1np1 timers 3 9
  neighbor enp94s0f1np1 timers connect 5
  !
  address-family ipv4 unicast
    redistribute connected
    # Redistribute the static default (from WAN DHCP) into BGP
    redistribute static
    maximum-paths 2
    !
    neighbor enp94s0f0np0 activate
    neighbor enp94s0f0np0 route-map SET-LOCALPREF-HIGH in
    neighbor enp94s0f0np0 route-map EXPORT-DEFAULT out
    neighbor enp94s0f0np0 soft-reconfiguration inbound
    !
    neighbor enp94s0f1np1 activate
    neighbor enp94s0f1np1 route-map SET-LOCALPREF-LOW in
    neighbor enp94s0f1np1 route-map EXPORT-DEFAULT out
    neighbor enp94s0f1np1 soft-reconfiguration inbound
  exit-address-family
  !
  address-family l2vpn evpn
    # Advertise this firewall's VTEP as PIP
    advertise-pip ip 10.10.254.1
    advertise ipv4 unicast
    # Default route into EVPN as type-5 prefix
    default-originate ipv4
    neighbor enp94s0f0np0 activate
    neighbor enp94s0f1np1 activate
  exit-address-family
!
# ============================================================
# nftables sketch (save to /etc/nftables.conf):
# table inet filter {
#   chain input   { type filter hook input priority 0; policy drop;
#     ct state established,related accept
#     ct state invalid drop
#     iif lo accept
#     iif brlan accept
#     iif enp94s0f0np0 accept  # BGP/BFD from fabric
#     iif enp94s0f1np1 accept
#     tcp dport 179 accept     # BGP
#     udp dport 3784 accept    # BFD control
#     udp dport 4784 accept    # BFD multihop
#   }
#   chain forward { type filter hook forward priority 0; policy drop;
#     ct state established,related accept
#     iif brlan oif enp1s0 accept
#   }
#   chain postrouting { type nat hook postrouting priority 100;
#     iif brlan oif enp1s0 masquerade
#   }
# }
# ============================================================
```
