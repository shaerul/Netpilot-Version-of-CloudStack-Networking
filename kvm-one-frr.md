```
# ============================================================
# FILE: /etc/frr/frr.conf — kvm-one
# ASN: 65010 | Router-ID: 10.10.254.10
# Active uplink: enp94s0f0np0 -> switch-one (local-pref 200)
# Backup uplink: enp94s0f1np1 -> switch-two (local-pref 100)
# VTEP source: 10.10.254.10 (dummy0)
# BFD enabled for <300ms failure detection
# ============================================================

frr version 8.5
frr defaults datacenter
hostname kvm-one
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
  ip address 10.10.254.10/32
  ipv6 address fd00:254::10/128
!
interface dummy0
  ip address 10.10.254.10/32
!
interface enp94s0f0np0
  description switch-one ACTIVE link
  no ipv6 nd suppress-ra
  ipv6 nd ra-interval 5
!
interface enp94s0f1np1
  description switch-two BACKUP link
  no ipv6 nd suppress-ra
  ipv6 nd ra-interval 5
!
# ----------------------------------------------------------
# Route-maps for active/backup path preference
# Active link (enp94s0f0np0): local-pref 200 — preferred
# Backup link (enp94s0f1np1): local-pref 100 — standby
# ----------------------------------------------------------
route-map SET-LOCALPREF-HIGH permit 10
  set local-preference 200
!
route-map SET-LOCALPREF-LOW permit 10
  set local-preference 100
!
router bgp 65010
  bgp router-id 10.10.254.10
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
    # Redistribute loopback (VTEP) and bridge IP
    redistribute connected
    maximum-paths 2
    !
    # Apply local-pref on inbound to prefer active link
    neighbor enp94s0f0np0 activate
    neighbor enp94s0f0np0 route-map SET-LOCALPREF-HIGH in
    neighbor enp94s0f0np0 soft-reconfiguration inbound
    !
    neighbor enp94s0f1np1 activate
    neighbor enp94s0f1np1 route-map SET-LOCALPREF-LOW in
    neighbor enp94s0f1np1 soft-reconfiguration inbound
  exit-address-family
  !
  address-family l2vpn evpn
    # Advertise this host's VTEP as PIP (per-instance IP)
    advertise-pip ip 10.10.254.10
    advertise ipv4 unicast
    !
    neighbor enp94s0f0np0 activate
    neighbor enp94s0f1np1 activate
  exit-address-family
!
# ============================================================
# Verify: vtysh -c 'show bgp summary'
#         vtysh -c 'show evpn vni'
#         vtysh -c 'show bfd peers'
# ============================================================
```
