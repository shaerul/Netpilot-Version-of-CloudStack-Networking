All four device configs are now updated with your real NIC names. Here's what changed and what to do next:

---

## What Was Updated

The two KVM host configs are now updated with your real Mellanox interface names:

| Interface | Role | Connects to |
|-----------|------|-------------|
| `enp94s0f0np0` | **ACTIVE** | switch-one `swp1` / `swp2` |
| `enp94s0f1np1` | **BACKUP** | switch-two `swp1` / `swp2` |

> ⚠️ **kvm-two assumption**: I used the same NIC names for kvm-two. If they differ, check with `ip link show` and swap them — the FRR `interface` block and Netplan stanza both need to match.

---

## Deployment Order

Apply in this sequence to avoid BGP flaps:

### 1. Both switches first
```bash
# Apply /etc/network/interfaces → reload
sudo ifreload -a

# Apply sysctl
sudo sysctl -p /etc/sysctl.d/99-network-tuning.conf

# Place frr.conf → restart FRR
sudo systemctl restart frr
```

### 2. Both KVM hosts
```bash
# Create the loopback service (paste the unit, then enable it)
sudo systemctl enable --now loopback-addrs.service

# Apply netplan
sudo netplan apply

# Apply sysctl
sudo sysctl -p /etc/sysctl.d/99-network-tuning.conf

# Place frr.conf → restart FRR
sudo systemctl restart frr
```

### 3. Verify convergence
```bash
# On any device — BGP sessions
vtysh -c 'show bgp summary'
vtysh -c 'show bgp l2vpn evpn summary'

# EVPN MAC/IP routes (should see Type-2 from both hosts)
vtysh -c 'show bgp l2vpn evpn route type macip'

# VTEP table (both 10.10.254.10 and .11 should appear)
vtysh -c 'show evpn vni 200'

# BFD sessions (all should show Up)
vtysh -c 'show bfd peers'

# End-to-end VXLAN test
ping -I lo 10.10.200.11   # from kvm-one → kvm-two
```

### 4. Failover test
```bash
# Simulate switch-one failure
sudo ip link set enp94s0f0np0 down   # on kvm-one
# Traffic should shift to backup within ~300ms
vtysh -c 'show bgp summary'          # switch-one sessions should show Gone
ping -I lo 10.10.200.11              # should still work via switch-two
```
