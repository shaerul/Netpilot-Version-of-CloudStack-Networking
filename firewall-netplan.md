```
# ============================================================
# FILE: /etc/netplan/01-netcfg.yaml — firewall
# ASN: 65001 | Loopback: 10.10.254.1/32
# Bridge brlan: 10.10.200.1/22 (anycast GW / LAN bridge for VMs)
# WAN: enp1s0 (DHCP) — UPDATE interface name to match your hardware
# Active fabric uplink: enp94s0f0np0 -> switch-one swp48
# Backup fabric uplink: enp94s0f1np1 -> switch-two swp48
# ============================================================
network:
  version: 2
  renderer: networkd

  ethernets:
    # !! UPDATE THIS to your actual WAN interface name !!
    enp1s0:
      dhcp4: true
      dhcp6: false
      # DHCP client sends hostname for DNS registration
      dhcp4-overrides:
        hostname: firewall
        send-hostname: true
        use-routes: true
        route-metric: 100

    # Active BGP fabric uplink to switch-one swp48 (local-pref 200)
    enp94s0f0np0:
      mtu: 9216
      dhcp4: false
      dhcp6: false
      accept-ra: false

    # Backup BGP fabric uplink to switch-two swp48 (local-pref 100)
    enp94s0f1np1:
      mtu: 9216
      dhcp4: false
      dhcp6: false
      accept-ra: false

  # Dummy interface — BGP router-id / loopback source
  dummy-interfaces:
    dummy0:
      addresses:
        - 10.10.254.1/32

  # LAN bridge — VMs attach via tap interfaces (virsh/libvirt)
  # This IS the anycast gateway; same IP as switches' vlan200 SVI
  bridges:
    brlan:
      dhcp4: false
      dhcp6: false
      mtu: 9216
      addresses:
        - 10.10.200.1/22
      # No bridge-ports by default; VMs attach dynamically via tap
      parameters:
        stp: false
        forward-delay: 0

# ============================================================
# NOTE: FRR handles all routing. The firewall originates
# a default route (0.0.0.0/0) into EVPN via 'redistribute static'
# with a static default learned from WAN DHCP.
# After netplan apply, verify: ip route show; ip link show brlan
# ============================================================
```
