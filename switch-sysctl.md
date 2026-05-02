```
# ============================================================
# FILE: /etc/sysctl.d/99-fabric.conf
# TARGET: switch-one AND switch-two (Cumulus Linux / Mellanox)
# Apply with: sudo sysctl -p /etc/sysctl.d/99-fabric.conf
# ============================================================

# ----------------------------------------------------------
# IP Forwarding — mandatory for routing
# ----------------------------------------------------------
net.ipv4.ip_forward = 1
net.ipv6.conf.all.forwarding = 1

# ----------------------------------------------------------
# Reverse Path Filter — disable for ECMP and asymmetric routing
# VXLAN and EVPN traffic may arrive on unexpected interfaces
# ----------------------------------------------------------
net.ipv4.conf.all.rp_filter = 0
net.ipv4.conf.default.rp_filter = 0

# ----------------------------------------------------------
# Bridge netfilter — disable; we handle filtering in FRR/nftables
# Prevents iptables from intercepting bridged VXLAN traffic
# ----------------------------------------------------------
net.bridge.bridge-nf-call-iptables = 0
net.bridge.bridge-nf-call-ip6tables = 0
net.bridge.bridge-nf-call-arptables = 0

# ----------------------------------------------------------
# ARP cache tuning — large L3 fabric, many endpoints
# Default gc_thresh values are too small for datacenter scale
# ----------------------------------------------------------
net.ipv4.neigh.default.gc_thresh1 = 2048
net.ipv4.neigh.default.gc_thresh2 = 8192
net.ipv4.neigh.default.gc_thresh3 = 16384
net.ipv6.neigh.default.gc_thresh1 = 2048
net.ipv6.neigh.default.gc_thresh2 = 8192
net.ipv6.neigh.default.gc_thresh3 = 16384

# ----------------------------------------------------------
# Socket buffer sizes — jumbo frame (9216 MTU) support
# ----------------------------------------------------------
net.core.rmem_max = 134217728
net.core.wmem_max = 134217728
net.core.rmem_default = 16777216
net.core.wmem_default = 16777216
net.core.optmem_max = 65536
net.ipv4.tcp_rmem = 4096 87380 134217728
net.ipv4.tcp_wmem = 4096 65536 134217728

# ----------------------------------------------------------
# VXLAN / UDP offload tuning
# ----------------------------------------------------------
# Allow large UDP send buffers (VXLAN encapsulation)
net.core.netdev_max_backlog = 300000
net.core.netdev_budget = 600

# ----------------------------------------------------------
# IPv6 link-local — required for BGP unnumbered
# Keep IPv6 enabled on fabric interfaces
# ----------------------------------------------------------
net.ipv6.conf.all.disable_ipv6 = 0
net.ipv6.conf.default.disable_ipv6 = 0
# Accept router advertisements on uplinks (for BGP unnumbered)
net.ipv6.conf.all.accept_ra = 1

# ----------------------------------------------------------
# Congestion control — BBR for better throughput on fabric
# ----------------------------------------------------------
net.core.default_qdisc = fq
net.ipv4.tcp_congestion_control = bbr

# ----------------------------------------------------------
# Misc
# ----------------------------------------------------------
# Increase local port range for high-connection-count environments
net.ipv4.ip_local_port_range = 1024 65535
# Reduce TIME_WAIT sockets
net.ipv4.tcp_tw_reuse = 1
# ============================================================
```
