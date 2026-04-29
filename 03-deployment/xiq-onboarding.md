# XIQ Onboarding — AP and Switch Onboarding

## Overview

XIQ (ExtremeCloud IQ) is the cloud management platform for Extreme Networks devices. Onboarding connects physical hardware (AP3000, 5320 switch) to your XIQ organisation so policies can be pushed and devices monitored.

## Device Onboarding Methods

| Method | Use When | How |
|--------|---------|-----|
| Serial Number | New device in box | Enter S/N in XIQ → device calls home |
| CSV Bulk Import | Multiple devices | Download template → fill S/N + MAC → upload |
| DHCP Option 43/138 | Lab / existing network | DHCP tells device XIQ server URL |
| Redirection Service | Default | Factory device contacts redirect.extremecloudiq.com → redirected to your org |

## AP3000 Onboarding — Step by Step

### 1. Add Device in XIQ
```
Manage → Devices → + Add Devices
→ Enter: Serial Number + MAC Address (from label on AP)
→ Assign to: Location (building → floor)
→ Click Add
```

### 2. Physical Connection
```
AP3000 → PoE+ switch port (5320 Port 3 or 5)
         ↑ 802.3at required — 25.5W draw
AP powers up → gets DHCP on Management VLAN (VLAN 10)
             → calls home to hac.extremecloudiq.com
```

### 3. Verify Connection
```
# On switch:
show lldp neighbors          # AP3000 MAC visible on port 3
show poe port 3              # Class 4, Oper Status: On

# In XIQ:
Manage → Devices → AP status: green = connected
```

### 4. Assign Network Policy
```
XIQ: Manage → Devices → select AP → Actions → Assign Network Policy
→ Select: Horizon-WiFi-Policy
→ Click Update → Complete Update
```

### 5. Verify Policy Push
```
XIQ: Manage → Devices → AP → Config Status: "In Sync"
On AP: SSIDs broadcasting (confirm from mobile device WiFi scan)
```

## 5320 Switch Onboarding

### EXOS IQ Agent Setup
```
# On switch console:
configure iqagent server hac.extremecloudiq.com
enable iqagent

# Verify:
show application iqagent status
# Connection: Connected
# Server: hac.extremecloudiq.com
```

### XIQ: Add Switch
```
Manage → Devices → + Add Devices → enter S/N
→ Device appears in Devices list
→ Assign to location + Network Policy
```

### VOSS IQ Agent Setup
```
# VOSS CLI:
application iqagent enable
application iqagent server hac.extremecloudiq.com

# Verify:
show application iqagent status
```

## Fabric Attach — AP Auto-Provisioning

When FA is enabled on the switch port, the AP doesn't need manual VLAN configuration. The AP requests its I-SIDs via LLDP and the switch provisions them automatically.

**Switch setup (VOSS):**
```
interface gigabitEthernet 1/3
  auto-sense enable
  fa enable
  exit
```

**Switch setup (EXOS):**
```
enable lldp port 3
configure lldp port 3 advertise vendor-specific dot1-tlvs
enable auto-sense port 3
```

**Verify FA working:**
```
show fa assignment        # Port 3 I-SIDs: Active + Dynamic
show fa neighbor          # AP3000 MAC listed
```

If `show fa neighbor` is empty → LLDP blocked or auto-sense not enabled.

## Network Policy Workflow

```
Network Policy
    ├── SSID Profile (radio settings, band steering, 802.11r)
    ├── User Profile → VLAN + Firewall + QoS
    └── Device Template (AP model-specific: radio power, channel, 6 GHz)
```

### Creating a Network Policy
```
Configure → Network Policies → + Add Policy
→ Name: Horizon-WiFi-Policy
→ Add SSID: + (attach Corp-Profile and Guest-Profile)
→ Add User Profiles: Corp-Users, Guest-Users
→ Device Templates: AP3000 template
→ Save
```

### Pushing Policy to APs
```
Configure → Network Policies → select policy → Push Updates
→ Select: Complete Update (full config push — use first time)
→ vs Partial Update (incremental — faster but can miss deletions)
→ Select devices → Push
```

**⚠ Always use Complete Update** after adding/removing SSIDs or changing security settings. Partial Update after QoS or firewall rule edits.

## Location Hierarchy Setup

XIQ uses a location tree for floor plan assignment and monitoring:

```
Manage → Location → + Add
  Organisation
  └── Site (e.g., "Horizon HQ")
      └── Building (e.g., "Main Office")
          └── Floor (e.g., "Ground Floor")
              → Drag APs here
              → Upload floor plan
```

Assigning APs to floors enables:
- Heat map visualisation (RSSI coverage)
- Client roaming trail (Client 360)
- Location-based alerts

## Common Onboarding Issues

| Problem | Cause | Fix |
|---------|-------|-----|
| AP shows Offline in XIQ | No internet / DHCP failed | Check PoE on, VLAN 10 DHCP working, `show iqagent status` |
| Policy push fails | AP not reachable during push | Verify AP online, retry Complete Update |
| S/N not accepted | Already claimed by another org | Contact Extreme TAC to release device |
| FA not provisioning | LLDP blocked or auto-sense disabled | Enable LLDP on port + auto-sense + fa enable |
| IQ agent disconnects | NTP drift / firewall blocking port 443 | Check time sync, allow outbound HTTPS to XIQ |

## XIQ Quick Navigation

| Task | Path |
|------|------|
| Add device | Manage → Devices → + Add |
| Push policy | Configure → Network Policies → Push Updates |
| Monitor AP | Manage → Devices → [AP] → Device 360 |
| Client trace | Manage → Clients → [MAC] → Client 360 |
| Event logs | Manage → Events |
| Floor plan | ML Insights → Network 360 Monitor → Floor Plan |
| Alerts | Configure → Alerts → Alert Policies |
