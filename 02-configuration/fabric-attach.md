# Fabric Attach (FA) — AP3000 + VOSS Switch

## What Fabric Attach Does
Instead of statically configuring VLAN trunk ports for each AP, Fabric Attach (FA) allows the AP3000 to tell the switch which I-SIDs (services) it needs. The switch then dynamically provisions those VLANs on the port.

**EXOS (manual):**
```
configure vlan VLAN_20 add ports 3 tagged
configure vlan VLAN_30 add ports 3 tagged
# Must repeat every time an SSID/VLAN changes
```

**VOSS with Fabric Attach:**
```
interface gigabitEthernet 1/3
  auto-sense enable   # switch listens for FA LLDP TLVs
  fa enable           # switch will grant I-SID requests
  no shutdown
# AP requests I-SID 100020 + 100030 via LLDP — switch provisions automatically
```

## FA Negotiation Flow
```
AP3000 boots → sends LLDP FA TLV → "I need I-SID 100020 (Corp) and 100030 (Guest)"
                                              ↓
Switch (fa enable + auto-sense enable on Port 3)
  → validates I-SIDs exist in vlan i-sid table
  → provisions VLAN 20 + VLAN 30 on Port 3 dynamically
  → responds via LLDP FA TLV: "Granted"
```

## Switch Configuration (VOSS)
```
# Global FA enable
fa enable

# Per-port (Port 3 = AP port)
interface gigabitEthernet 1/3
  auto-sense enable
  fa enable
  no shutdown
  exit
```

## XIQ Device Template (FA)
Network Policy → Switch Template → Port 3:
- Mode: **Auto-Sense**
- Fabric Attach: **Enabled**
- I-SID Mapping: VLAN 20 → 100020, VLAN 30 → 100030

## Verify FA is Working
```
show fa assignment
# Expected output:
# Port  I-SID   VLAN  State    Type
# 1/3   100020  20    Active   Dynamic
# 1/3   100030  30    Active   Dynamic

show fa neighbor
# Expected: AP3000 MAC address listed with port 1/3
# If empty → LLDP is blocked on this port
```

## Debug FA Issues
| Symptom | Command | What to Look For |
|---------|---------|-----------------|
| AP gets no VLANs | `show fa neighbor` | Empty → LLDP blocked |
| I-SID not granted | `show vlan i-sid` | I-SID must exist before FA can grant it |
| FA not re-negotiating | `no fa enable` + `fa enable` on port | Force re-sync |
| AP can't reach XIQ | `show application iqagent status` | Check management VLAN path |

## EXOS → VOSS Comparison on Port 3
| Aspect | EXOS | VOSS FA |
|--------|------|---------|
| Config effort | Manual per SSID change | One-time auto-sense enable |
| New SSID added | Must re-touch port config | AP re-negotiates I-SID automatically |
| Visibility | `show vlan VLAN_20` | `show fa assignment` |
| Management VLAN | Default VLAN untagged on port | Handled by auto-sense |
