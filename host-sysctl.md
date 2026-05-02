```
# ============================================================
# FILE: /etc/sysctl.d/99-fabric.conf
# TARGET: kvm-one AND kvm-two (Ubuntu 24.04 KVM hosts)
# Apply with: sudo sysctl -p /etc/sysctl.d/99-fabric.conf
# ============================================================

# ----------------------------------------------------------
# IP Forwarding — KVM hosts must forward for VM traffic
# ----------------------------------------------------------
net.ipv4.ip_forward = 1
net.ipv6.conf.all.forwarding = 1

# ----------------------------------------------------------
# Reverse Path Filter — disable for dual-homed BGP uplinks
# Traffic from switch-two may ingress on enp94s0f1np1 while
# the return path is via enp94s0f0np0 (active link)
# ----------------------------------------------------------
net.ipv4.conf.all.rp_filter = 0
net.ipv4.conf.default.rp_filter = 0
net.ipv4.conf.enp94s0f0np0.rp_filter = 0
net.ipv4.conf.enp94s0f1np1.rp_filter = 0

# ----------------------------------------------------------
# Bridge netfilter
# ----------------------------------------------------------
net.bridge.bridge-nf-call-iptables = 0
net.bridge.bridge-nf-call-ip6tables = 0
net.bridge.bridge-nf-call-arptables = 0

# ----------------------------------------------------------
# ARP tuning — KVM host may have many VM MACs
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
# VXLAN / UDP
# ----------------------------------------------------------
net.core.netdev_max_backlog = 300000
net.core.netdev_budget = 600

# ----------------------------------------------------------
# IPv6 — keep enabled for BGP unnumbered
# ----------------------------------------------------------
net.ipv6.conf.all.disable_ipv6 = 0
net.ipv6.conf.default.disable_ipv6 = 0
net.ipv6.conf.all.accept_ra = 1

# ----------------------------------------------------------
# KVM / libvirt tuning
# ----------------------------------------------------------
# Huge pages for KVM guests (adjust count per workload)
# vm.nr_hugepages = 1024
# vm.hugetlb_shm_group = 0

# Shared memory for KVM
kernel.shmmax = 68719476736
kernel.shmall = 4294967296

# ----------------------------------------------------------
# Congestion control
# ----------------------------------------------------------
net.core.default_qdisc = fq
net.ipv4.tcp_congestion_control = bbr
net.ipv4.tcp_tw_reuse = 1
net.ipv4.ip_local_port_range = 1024 65535

# ============================================================
# FILE: /etc/sysctl.d/99-firewall.conf
# TARGET: firewall host (Ubuntu 24.04)
# Apply with: sudo sysctl -p /etc/sysctl.d/99-firewall.conf
# ============================================================

net.ipv4.ip_forward = 1
net.ipv6.conf.all.forwarding = 1

# Reverse Path Filter — disable; asymmetric fabric routing
net.ipv4.conf.all.rp_filter = 0
net.ipv4.conf.default.rp_filter = 0

# ----------------------------------------------------------
# Connection tracking — scale for firewall workloads
# ----------------------------------------------------------
net.netfilter.nf_conntrack_max = 2000000
net.netfilter.nf_conntrack_tcp_timeout_established = 86400
net.netfilter.nf_conntrack_tcp_timeout_time_wait = 120
net.netfilter.nf_conntrack_tcp_timeout_fin_wait = 30
net.netfilter.nf_conntrack_udp_timeout = 30
net.netfilter.nf_conntrack_udp_timeout_stream = 180
net.netfilter.nf_conntrack_generic_timeout = 60

# ----------------------------------------------------------
# ARP tuning
# ----------------------------------------------------------
net.ipv4.neigh.default.gc_thresh1 = 2048
net.ipv4.neigh.default.gc_thresh2 = 8192
net.ipv4.neigh.default.gc_thresh3 = 16384

# ----------------------------------------------------------
# Socket buffers
# ----------------------------------------------------------
net.core.rmem_max = 134217728
net.core.wmem_max = 134217728
net.core.rmem_default = 16777216
net.core.wmem_default = 16777216
net.core.netdev_max_backlog = 300000
net.ipv4.tcp_rmem = 4096 87380 134217728
net.ipv4.tcp_wmem = 4096 65536 134217728

# ----------------------------------------------------------
# IPv6
# ----------------------------------------------------------
net.ipv6.conf.all.disable_ipv6 = 0
net.ipv6.conf.default.disable_ipv6 = 0
net.ipv6.conf.all.accept_ra = 1

# ----------------------------------------------------------
# Congestion control
# ----------------------------------------------------------
net.core.default_qdisc = fq
net.ipv4.tcp_congestion_control = bbr
net.ipv4.tcp_tw_reuse = 1
net.ipv4.ip_local_port_range = 1024 65535

# ============================================================
```
