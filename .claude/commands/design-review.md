---
name: design-review
description: Audit a WiFi/SSID design against best practices. Returns PASS/FAIL/WARN per criterion. Use before finalising any design.
---

Review the provided WiFi design against these criteria and return a table with PASS / FAIL / WARN status for each.

## SSID Design
- [ ] Max 4 SSIDs per AP band (beacon overhead)
- [ ] No SSID name reveals vendor, technology, or physical location
- [ ] Band steering configured (clients prefer 5/6GHz over 2.4GHz)
- [ ] 802.11k (neighbour reports) + 802.11v (BSS Transition) enabled
- [ ] 802.11r (Fast BSS Transition) enabled for voice/video SSIDs

## Security
- [ ] Corp: WPA3-SAE or WPA2/WPA3 Transition Mode
- [ ] Guest: OWE (Enhanced Open) + Transition Mode for legacy devices
- [ ] PMF (Protected Management Frames) enabled — mandatory for WPA3
- [ ] PSK Key Value field confirmed non-empty in XIQ
- [ ] 802.1X/RADIUS for enterprise (if applicable)

## Isolation & ACL
- [ ] Guest→Corp blocked (XIQ firewall rule: deny 10.30.0.0/24 → 10.10.0.0/24)
- [ ] Guest→Corp Wireless blocked (deny 10.30.0.0/24 → 10.20.0.0/24)
- [ ] Guest L2 station isolation (Drop traffic between stations)
- [ ] VOSS: ACL on BOTH switches (Anycast Gateway change from EXOS)
- [ ] IP Objects created in Common Objects before firewall rules

## VLAN / I-SID
- [ ] 1:1 SSID-to-VLAN mapping
- [ ] VOSS: I-SID = VLAN ID + 100,000
- [ ] VOSS: I-SID mapping identical on both switches
- [ ] Transit VLAN 100 on SW1 only (VOSS internet exit)

## QoS
- [ ] Corp → high QoS (qp6 / CoS 5–6 / DSCP EF)
- [ ] Guest → low QoS (qp1 / CoS 0)
- [ ] XIQ User Profiles: Corp Weight ≥ 100, Guest Weight ≤ 10
- [ ] WMM enabled on AP

## Deployment Readiness
- [ ] PoE budget calculated (AP3000 = 25.5W, need 802.3at)
- [ ] OWE Transition Mode toggle confirmed ON
- [ ] Complete Configuration Update planned (not partial)
- [ ] Failover test procedure documented

Output format:
| Category | Criterion | Status | Notes |
|----------|-----------|--------|-------|
