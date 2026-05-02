```
# ============================================================
# FILE: /etc/frr/frr.conf — switch-two
# ASN: 65000 | Router-ID: 10.10.254.6
# Role: Leaf switch + EVPN Route Reflector client of switch-one
# BFD enabled on all BGP sessions
# ============================================================

frr version 8.5
frr defaults datacenter
hostname switch-two
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
  ip address 10.10.254.6/32
  ipv6 address fd00:254::6/128
!
interface swp1
  description kvm-one BACKUP uplink
  no ipv6 nd suppress-ra
  ipv6 nd ra-interval 5
!
interface swp2
  description kvm-two BACKUP uplink
  no ipv6 nd suppress-ra
  ipv6 nd ra-interval 5
!
interface swp48
  description firewall BACKUP uplink
  no ipv6 nd suppress-ra
  ipv6 nd ra-interval 5
!
interface swp49
  description switch-one P2P underlay
  ip address 169.254.0.26/30
  no ipv6 nd suppress-ra
!
interface vlan200
  description Tenant SVI - anycast GW
  ip address 10.10.200.1/22
  ip address-virtual 44:38:39:ff:00:01 10.10.200.1/22
!
router bgp 65000
  bgp router-id 10.10.254.6
  bgp bestpath as-path multipath-relax
  bgp bestpath compare-routerid
  no bgp ebgp-requires-policy
  no bgp network import-check
  !
  neighbor swp1 interface remote-as 65010
  neighbor swp1 description kvm-one-backup
  neighbor swp1 bfd profile fast-link
  neighbor swp1 advertisement-interval 0
  neighbor swp1 timers 3 9
  neighbor swp1 timers connect 5
  !
  neighbor swp2 interface remote-as 65011
  neighbor swp2 description kvm-two-backup
  neighbor swp2 bfd profile fast-link
  neighbor swp2 advertisement-interval 0
  neighbor swp2 timers 3 9
  neighbor swp2 timers connect 5
  !
  neighbor swp48 interface remote-as 65001
  neighbor swp48 description firewall-backup
  neighbor swp48 bfd profile fast-link
  neighbor swp48 advertisement-interval 0
  neighbor swp48 timers 3 9
  neighbor swp48 timers connect 5
  !
  neighbor 10.10.254.5 remote-as 65000
  neighbor 10.10.254.5 description switch-one-ibgp
  neighbor 10.10.254.5 update-source lo
  neighbor 10.10.254.5 ebgp-multihop 2
  neighbor 10.10.254.5 bfd
  neighbor 10.10.254.5 advertisement-interval 0
  neighbor 10.10.254.5 timers 3 9
  !
  address-family ipv4 unicast
    redistribute connected
    maximum-paths 4
    maximum-paths ibgp 4
    neighbor swp1 activate
    neighbor swp1 prefix-list HOSTS-IN in
    neighbor swp2 activate
    neighbor swp2 prefix-list HOSTS-IN in
    neighbor swp48 activate
    neighbor swp48 prefix-list FIREWALL-IN in
    neighbor 10.10.254.5 activate
    neighbor 10.10.254.5 next-hop-self
  exit-address-family
  !
  address-family l2vpn evpn
    advertise-all-vni
    advertise-default-gw
    neighbor swp1 activate
    neighbor swp1 route-reflector-client
    neighbor swp2 activate
    neighbor swp2 route-reflector-client
    neighbor swp48 activate
    neighbor swp48 route-reflector-client
    neighbor 10.10.254.5 activate
  exit-address-family
!
ip prefix-list HOSTS-IN seq 10 permit 10.10.254.0/24 le 32
ip prefix-list HOSTS-IN seq 20 permit 10.10.200.0/22
ip prefix-list HOSTS-IN seq 30 permit 0.0.0.0/0
ip prefix-list HOSTS-IN seq 100 deny any
!
ip prefix-list FIREWALL-IN seq 10 permit 10.10.254.0/24 le 32
ip prefix-list FIREWALL-IN seq 20 permit 10.10.200.0/22
ip prefix-list FIREWALL-IN seq 30 permit 0.0.0.0/0
ip prefix-list FIREWALL-IN seq 100 deny any
!
# ============================================================
```
