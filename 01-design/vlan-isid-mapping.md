# VLAN → I-SID Mapping

## The Fundamental Difference
| Concept | EXOS | VOSS |
|---------|------|------|
| Service identity | VLAN (local scope per switch) | I-SID (global scope — all Fabric switches) |
| Trunk configuration | Manual: tag every VLAN on every uplink port | Automatic: `isis spbm 1` on NNI — done |
| AP port | Static: `configure vlan X add ports 3 tagged` | Dynamic: `auto-sense enable` + `fa enable` |
| New VLAN rollout | Touch both switches + all trunk ports | Add `vlan i-sid` on both — NNI carries it automatically |

## I-SID Convention: VLAN ID + 100,000
This is the industry standard convention. It makes I-SIDs instantly readable:
- See `100020` in a trace → immediately know it's VLAN 20 (Corp Wireless)
- See `100030` → VLAN 30 (Guest Wireless)

## Horizon Service Map
| SSID | VLAN | I-SID | Gateway (Anycast) | QoS | Notes |
|------|------|-------|-------------------|-----|-------|
| Corp Wired | 10 | 100010 | 10.10.0.1 | qp6 | Port 5, untagged |
| Corporate_Wireless | 20 | 100020 | 10.20.0.1 | qp6 | Port 3, FA-provisioned |
| Guest_Wireless | 30 | 100030 | 10.30.0.1 | qp1 | Port 3, FA-provisioned |
| Internet Transit | 100 | — | 192.168.0.28 | — | SW1 Port 1 only, no I-SID |

## EXOS CLI — Manual VLAN-to-Port Mapping
```
# Create VLANs
create vlan "VLAN_10" tag 10
create vlan "VLAN_20" tag 20
create vlan "VLAN_30" tag 30

# AP port (Port 3) — Default untagged for boot, data VLANs tagged
configure vlan Default add ports 3 untagged
configure vlan VLAN_20 add ports 3 tagged
configure vlan VLAN_30 add ports 3 tagged

# Trunk (Port 10) — all data VLANs + Default (after Apr 28 loop fix)
configure vlan VLAN_10 add ports 10 tagged
configure vlan VLAN_20 add ports 10 tagged
configure vlan VLAN_30 add ports 10 tagged
configure vlan Default add ports 10 tagged   # safe ONLY after SW2 Port 1 disconnected

# Wired client (Port 5)
configure vlan VLAN_10 add ports 5 untagged
```

## VOSS CLI — I-SID Service Provisioning
```
# Run on BOTH switches — mapping must be identical
vlan create 10,20,30 type port-mstprstp 0
vlan i-sid 10 100010
vlan i-sid 20 100020
vlan i-sid 30 100030

# NNI (Port 10) — one-time, carries all I-SIDs automatically
interface gigabitEthernet 1/10
  isis enable
  isis spbm 1
  no shutdown
  exit

# AP port (Port 3) — Fabric Attach
interface gigabitEthernet 1/3
  auto-sense enable
  fa enable
  no shutdown
  exit

# Wired (Port 5)
vlan members add 10 1/5

# Verify
show i-sid        # expect I-SID 100010/20/30 — Type: ELAN — State: Active
show fa assignment # expect Port 3 · I-SIDs Active after AP boots
```

## Modem Static Routes (Required — Both EXOS and VOSS)
The QuantumFiber modem must know how to reach the internal subnets via SW1:
```
Route: 10.10.0.0/24 → 192.168.0.28 (SW1)
Route: 10.20.0.0/24 → 192.168.0.28 (SW1)
Route: 10.30.0.0/24 → 192.168.0.28 (SW1)
```
Without these, return traffic from the internet can't find your clients.
