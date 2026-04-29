# AP3000 — Reference Notes

## AP3000 Specifications

| Spec | Value | Notes |
|------|-------|-------|
| WiFi Generation | WiFi 6 (802.11ax) | No 6 GHz — 2.4 + 5 GHz only |
| MIMO | 2×2:2 | 2 spatial streams per radio |
| Max PHY Rate | 574 Mbps (2.4 GHz) + 2.4 Gbps (5 GHz) | Theoretical — real-world ~600 Mbps aggregate |
| Power Requirement | 25.5W | Requires 802.3at (PoE+) — NOT 802.3af (15.4W) |
| Management | XIQ cloud-managed | Also supports local GUI fallback |
| Fabric Attach | Yes (LLDP-based) | Requests I-SIDs from VOSS/EXOS switch |
| Outdoor variant | AP3000X | IP67, vandal-resistant |
| Recommended max clients | ~50 per radio | 2×2 MIMO limits concurrent spatial streams |

## Power Requirement — Critical Detail

The AP3000 draws **25.5W**. This requires **PoE+ (802.3at, 30W budget)**.

| Standard | Max Power | AP3000 Compatible? |
|----------|----------|-------------------|
| 802.3af (PoE) | 15.4W | **NO** — AP will brownout or not power on |
| 802.3at (PoE+) | 30W | **YES** — required |
| 802.3bt Type 3 | 60W | Yes (overkill) |
| 802.3bt Type 4 | 100W | Yes (overkill) |

**Verify on switch:**
```
show poe port 3
# Must show: Class 4 (PoE+), Oper Status: On
# If Class 2 or 3: port may be limited — check switch PoE budget
```

**5320 PoE Budget:**
```
show poe detail
# Check: Total power budget vs consumed
# 5320-24T: 370W budget
# With 8 APs × 25.5W = 204W — within budget
```

## Fabric Attach — How It Works on AP3000

1. AP boots, gets DHCP on management VLAN
2. AP sends LLDP frames with FA TLVs requesting I-SIDs
3. Switch receives request: `show fa neighbor` lists AP MAC
4. Switch grants I-SIDs dynamically on that port
5. AP gets its data VLANs (Corp, Guest) without manual trunk config

```
# Verify FA working:
show fa neighbor          # AP3000 MAC on Port 3
show fa assignment        # Port 3: I-SIDs 100020 + 100030 → Active, Dynamic

# Debug if AP not appearing:
show lldp neighbors       # Is AP visible at all via LLDP?
# If empty: check cable, check port up, check LLDP enabled on port
```

## XIQ Management — AP3000 Specifics

### Radio Settings in Device Template
```
XIQ: Configure → Network Policies → Device Templates → AP3000 Template
→ 2.4 GHz Radio:
   TX Power: Auto (or 17 dBm start point)
   Channel: Auto (RRM) or manual 1/6/11
   Channel Width: 20 MHz (never 40 MHz on 2.4 GHz)
→ 5 GHz Radio:
   TX Power: Auto (or 20 dBm start point)
   Channel: Auto (RRM) or manual from UNII-1 + UNII-3
   Channel Width: 80 MHz (standard) or 40 MHz (dense)
```

### AP3000 Does NOT Support
- 6 GHz radio (no WiFi 6E or WiFi 7)
- Multi-Link Operation (WiFi 7 — needs AP5020)
- 320 MHz channel width (WiFi 7)
- Tri-radio operation

For 6 GHz/WiFi 7: upgrade path is **AP5010** (indoor, tri-radio) or **AP5020** (high-performance).

## Placement Guidelines for AP3000

| Environment | Recommended Coverage Radius | Notes |
|-------------|----------------------------|-------|
| Open office | 15–20m | 2×2 MIMO sufficient |
| Classroom | 10–15m | Dense client scenario |
| Warehouse | 25–30m | Low client density, higher power |
| Corridor | 20–25m (directional) | Mount at end of corridor, angled down |

Height: 2.4–3.0m ceiling ideal. Below 2m = too close to client, poor pattern. Above 4m = coverage holes at 5 GHz.

## Common AP3000 Issues

| Symptom | Cause | Fix |
|---------|-------|-----|
| AP not powering on | PoE port (802.3af only) | Replace with PoE+ switch port or injector |
| AP online in XIQ but no SSIDs | Policy push failed | XIQ: Complete Update → push again |
| FA not provisioning VLANs | LLDP blocked or auto-sense off | `enable lldp port 3` + `auto-sense enable` |
| AP offline after upgrade | Firmware upgrade reboot | Wait 3 min; if still offline, check DHCP on mgmt VLAN |
| Slow WiFi near AP | AP overloaded (>50 clients) | Add second AP; reduce cell size with lower TX power |
| 5 GHz clients dropping | DFS channel radar hit | Move to UNII-1/UNII-3 channels |

## Upgrading from AP3000

When to consider upgrade:
- Site needs 6 GHz spectrum (dense environments, low latency)
- Client base is predominantly WiFi 6E or WiFi 7 capable
- Throughput per AP needs to exceed ~1 Gbps aggregate
- Multi-AP coordination required (WiFi 7 enterprise deployments)

| Model | Generation | Bands | MIMO | PoE |
|-------|----------|-------|------|-----|
| AP3000 | WiFi 6 | 2.4+5 | 2×2 | PoE+ |
| AP5010 | WiFi 6E | 2.4+5+6 | 4×4 | PoE++ (802.3bt) |
| AP5020 | WiFi 7 | 2.4+5+6 | 8×8 | PoE++ (802.3bt) |

## AP3000 — Horizon Lab Configuration Summary

```
Port 3 (SW2 UNI) → AP3000
  PoE+: Class 4, On
  LLDP: AP3000 MAC visible
  FA: I-SIDs 100020 + 100030 provisioned dynamically
  Management VLAN: 10 (untagged, DHCP)
  Corp VLAN: 20 (tagged, FA-assigned)
  Guest VLAN: 30 (tagged, FA-assigned)
```

SSID → VLAN mapping handled entirely via FA + XIQ policy — no manual trunk configuration on AP port.
