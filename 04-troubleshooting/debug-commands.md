# Debug Commands — EXOS and VOSS

## First 5 Commands at Any Site
```
show application iqagent status    # Is XIQ cloud connection alive?
show vlan detail                   # Are VLANs + SVIs configured?
show iproute                       # (EXOS) Default route present?
show ip route                      # (VOSS) Default route + Fabric routes?
show isis adj                      # (VOSS) Is Fabric adjacency UP?
```

## EXOS Command Reference
| Category | Command | What to Look For |
|----------|---------|-----------------|
| VLAN | `show vlan detail` | All VLANs present, IPs assigned, ipforwarding = Enabled |
| VLAN | `show vlan VLAN_20` | Correct ports listed (tagged/untagged) |
| IP | `show ipconfig vlan VLAN_20` | ip forwarding: Enabled |
| IP | `show iproute` | Default route 0.0.0.0/0 via modem IP |
| DHCP | `show dhcp lease vlan VLAN_20` | Active leases listed |
| DHCP | `show dhcp configuration vlan VLAN_20` | Ports include port 3, 5, 10 |
| AP/LLDP | `show lldp neighbors` | AP3000 MAC + port number visible |
| PoE | `show poe port 3` | Class 4, Oper Status: On |
| PoE | `show poe detail` | Total budget vs consumption |
| Port | `show port 3 information detail` | Link up, speed, duplex |
| Port | `show port utilization` | Traffic on AP port |
| QoS | `show policy` | ACLs and QoS policies applied |
| System | `show switch` | Firmware version, uptime |

## EXOS — Karl Rule Check
```
show ipconfig vlan Default
# Must show: IP forwarding: Enabled
# If disabled:
enable ipforwarding vlan Default
```

## EXOS — DHCP Reach-Back Check
```
show dhcp configuration vlan VLAN_10
# Must show: Enabled on ports 3,5,10
# If port 10 missing — SW2 clients get no IP
enable dhcp ports 3,5,10 vlan VLAN_10
```

## VOSS Command Reference
| Category | Command | What to Look For |
|----------|---------|-----------------|
| Fabric | `show isis adj` | State: UP — if empty/DOWN, fabric broken |
| Fabric | `show isis interface` | NNI port OperState: Up |
| Fabric | `show spbm` | Ethertype 0x8100 — must match on both switches |
| Services | `show i-sid` | I-SIDs 100010/20/30 — Type: ELAN, State: Active |
| Services | `show isis spbm i-sid` | "Local Only" = NNI down; "Both" = fabric active |
| FA | `show fa assignment` | Port 3 I-SIDs Active + Dynamic |
| FA | `show fa neighbor` | AP3000 MAC listed; if empty → LLDP blocked |
| Routing | `show ip route` | 0.0.0.0/0 → 0.00.01 nick-name (SW2) confirms IP Shortcuts |
| Routing | `show ip interface vlan 20` | ip address 10.20.0.1/24 on both switches (Anycast) |
| DHCP | `show ip dhcp-server summary` | Server enabled |
| DHCP | `show ip dhcp-server binding` | Active leases |
| XIQ | `show application iqagent status` | Connection: Connected |
| PoE | `show poe port 3` | Class 4, On |
| LLDP | `show lldp neighbor` | AP3000 visible on Port 3 |

## VOSS — IS-IS Adjacency Troubleshooting
```
# Check both sides
show isis adj                          # State must be UP

# If DOWN: check these
show spbm                              # ethertype 0x8100 on BOTH switches
show isis                              # manual-area must be 00.0001 on BOTH
show interface gigabitEthernet 1/10    # port must be Up

# Check nick-names are unique
# SW1: 0.00.01  SW2: 0.00.02 — if same, adjacency will flap
```

## VOSS — Failover Diagnostic Sequence
```
# While NNI cable is unplugged:
show isis adj              # adjacency gone → confirmed isolated
show ip route              # 0.0.0.0/0 no longer via 0.00.01
show isis spbm i-sid       # I-SIDs show "Local Only"

# After plugging back in:
show isis adj              # State: UP within seconds
show ip route              # 0.0.0.0/0 → 0.00.01 restored
```

## XIQ CLI Equivalent — Using show application
```
show application iqagent status
# Key fields:
# Connection: Connected     ← XIQ cloud reachable
# Server: hac.extremecloudiq.com
# Last Config Change: [timestamp]
```
