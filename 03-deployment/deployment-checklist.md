# Deployment Checklist — WiFi + Fabric

Use this checklist for every site deployment. Work top-to-bottom. Don't skip layers.

---

## Phase 1 — Network Design Sign-Off

- [ ] SSID count ≤ 4 per radio
- [ ] SSID names follow `Site-Purpose` convention (e.g., `Horizon-Corp`, `Horizon-Guest`)
- [ ] Each SSID maps to exactly one VLAN
- [ ] VLAN IDs defined: Management (10), Corp (20), Guest (30)
- [ ] I-SIDs calculated: VLAN + 100,000 = I-SID (100010, 100020, 100030)
- [ ] IP subnets assigned and documented (10.10.0.0/24, 10.20.0.0/24, 10.30.0.0/24)
- [ ] Security mode selected: WPA3/WPA2-Transition for Corp, OWE-Transition for Guest
- [ ] Guest isolation design confirmed: firewall rules + L2 station isolation
- [ ] QoS design: DSCP EF for voice, DSCP reset for Guest

---

## Phase 2 — Physical Installation

- [ ] AP mounted at correct height (8–10 ft / 2.4–3 m ceiling preferred)
- [ ] AP positioned clear of major obstacles (metal walls, lift shafts, dense concrete)
- [ ] Cell overlap ≥ 15–20% between adjacent APs (no coverage holes)
- [ ] Cat6 cable run to each AP (≤ 100m, no joins)
- [ ] Cable labelled at both ends
- [ ] Switch port verified as PoE+ (802.3at) — AP3000 requires 25.5W

---

## Phase 3 — Switch Configuration (EXOS SW1)

- [ ] VLANs created: VLAN 10, 20, 30
- [ ] SVIs configured with correct IPs
- [ ] **Karl Rule applied:** `enable ipforwarding vlan Default`
- [ ] IP forwarding enabled on all SVIs: `enable ipforwarding vlan VLAN_20`
- [ ] DHCP server configured on all VLANs with pools covering expected client count
- [ ] DHCP enabled on ports 3, 5, **10** (port 10 critical for SW2 reach-back)
- [ ] AP port (3/5) confirmed: PoE+ on, LLDP enabled
- [ ] Default route configured: `configure iproute add default <modem-ip>`
- [ ] Modem static routes: 10.10/20/30.0.0/24 → SW1 IP

---

## Phase 4 — Fabric Configuration (VOSS SW2 — Horizon variant)

- [ ] IS-IS configured: system-id unique, manual-area 00.0001, nickname unique (0.00.02)
- [ ] NNI (Port 10) configured: `isis enable` + `isis spbm 1`
- [ ] SPBM ethertype matches SW1: `show spbm` → 0x8100 on both
- [ ] IS-IS adjacency UP: `show isis adj` → State: UP
- [ ] I-SIDs created: 100010, 100020, 100030 (Type: ELAN)
- [ ] I-SIDs active on both switches: `show isis spbm i-sid` → "Both"
- [ ] Anycast gateway IPs configured on both switches: 10.20.0.1/24 on VLAN 20, etc.
- [ ] IP Shortcuts enabled: `ip-shortcut` under `router isis`
- [ ] SW2 default route learned via Fabric: `show ip route` → 0.0.0.0/0 → 0.00.01
- [ ] UNI ports (3, 5) configured: `fa enable` + `auto-sense enable`
- [ ] Guest isolation ACL applied to **both SW1 and SW2** (VOSS Anycast requires this)

---

## Phase 5 — XIQ Configuration

- [ ] Network Policy created: `Horizon-WiFi-Policy`
- [ ] IP Objects created: 10.10.0.0/24, 10.20.0.0/24, 10.30.0.0/24
- [ ] Corp SSID profile: WPA2/WPA3 Transition, band steering, 802.11r enabled
- [ ] **Corp PSK Key Value field populated** (common gotcha — left empty)
- [ ] Guest SSID profile: OWE, **OWE Transition Mode toggle ON** (legacy device support)
- [ ] Guest firewall rules: deny Guest→Corp, permit internet
- [ ] User profiles: Corp-Users (VLAN 20), Guest-Users (VLAN 30)
- [ ] QoS policies: Voice-Optimised for Corp, Best-Effort + DSCP reset for Guest
- [ ] Device Template: AP3000 template selected with correct radio power

---

## Phase 6 — XIQ Onboarding

- [ ] AP3000 S/N + MAC entered in XIQ
- [ ] AP assigned to correct location (building → floor)
- [ ] Network Policy assigned to AP
- [ ] **Complete Update** pushed (not Partial — first push always Complete)
- [ ] AP shows Config Status: "In Sync"
- [ ] Switch IQ agent connected: `show application iqagent status` → Connected

---

## Phase 7 — Post-Deployment Verification

### Wireless Connectivity
- [ ] Corp SSID visible on client device
- [ ] Guest SSID visible on client device
- [ ] Corp SSID: associates with correct password (WPA2 and WPA3 clients)
- [ ] Guest SSID: legacy device (iPhone/Android) associates with OWE Transition fallback

### DHCP
- [ ] Corp client receives IP in 10.20.0.0/24 range
- [ ] Guest client receives IP in 10.30.0.0/24 range
- [ ] No APIPA (169.254.x.x) addresses observed

### Routing
- [ ] Corp client pings gateway 10.20.0.1
- [ ] Corp client pings 8.8.8.8 (internet)
- [ ] Guest client pings gateway 10.30.0.1
- [ ] Guest client pings 8.8.8.8 (internet)

### Guest Isolation
- [ ] Guest → Corp ping FAILS (10.30.x → 10.20.x denied)
- [ ] Guest → Guest ping FAILS (L2 station isolation working)
- [ ] Corp → Corp ping succeeds
- [ ] Corp → internet succeeds

### QoS
- [ ] Voice traffic (SIP/RTP) lands in AC_VO in XIQ Client 360 SLA tab
- [ ] Guest traffic shows DSCP 0 (reset confirmed)

### Fabric Health (VOSS sites)
- [ ] `show isis adj` → UP on both switches
- [ ] `show isis spbm i-sid` → "Both" for all three I-SIDs
- [ ] `show fa assignment` → AP port I-SIDs Active + Dynamic
- [ ] XIQ Network 360: all links green

### Roaming Test
- [ ] Walk client from AP1 coverage to AP2 coverage
- [ ] Roaming event visible in XIQ Client 360
- [ ] Roaming latency ≤ 20ms (802.11r) — confirm in SLA tab

---

## Phase 8 — Documentation

- [ ] Floor plan uploaded to XIQ with AP positions marked
- [ ] RSSI heat map generated and reviewed (no holes below -75 dBm)
- [ ] IP address table saved (switch IPs, gateway IPs, modem IP)
- [ ] VLAN/I-SID mapping table saved
- [ ] PSK stored in password manager (not in email or chat)
- [ ] XIQ organisation credentials stored in password manager

---

## Quick Fault Reference

| Test Fails | Most Likely Cause | First Command |
|-----------|------------------|---------------|
| SSID not visible | Policy not pushed / AP offline | XIQ Manage → Devices → check status |
| Auth fails Corp | PSK Key Value empty in XIQ | Check SSID profile → Key Value field |
| Legacy device can't join Guest | OWE Transition Mode not toggled | SSID profile → OWE Transition Mode ON |
| 169.254.x.x on client | DHCP not on port 10 (SW2 reach-back) | `show dhcp configuration vlan VLAN_20` |
| No gateway ping | Karl Rule (EXOS) or Anycast IP missing | `show ipconfig vlan VLAN_20` → forwarding? |
| No internet | Missing default route | `show iproute` / `show ip route` |
| Guest reaches Corp | ACL missing on SW2 (VOSS) | Apply `vlan filter ip GUEST_ISOLATION 30 in` on SW2 |
| IS-IS DOWN | Ethertype mismatch or nick-name collision | `show spbm` on both — 0x8100 must match |
