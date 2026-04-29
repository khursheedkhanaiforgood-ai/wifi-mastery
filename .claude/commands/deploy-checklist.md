---
name: deploy-checklist
description: Generate a site-specific pre and post deployment checklist. Provide site name, AP count, switch type (EXOS/VOSS), and SSID list.
---

Generate a deployment checklist tailored to the described site. Include all items below, annotated with site-specific values.

## PRE-DEPLOYMENT

**Network Design**
- [ ] Site survey complete — RSSI ≥ -70 dBm at all desired coverage points
- [ ] VLAN design documented with IP addressing confirmed
- [ ] SSID → VLAN → I-SID mapping table finalised (VOSS)
- [ ] Modem/router static routes added for all VLANs → SW1 gateway

**Switch Config (EXOS)**
- [ ] VLANs created and named
- [ ] Port 3 (AP): Default VLAN untagged + Corp/Guest VLANs tagged
- [ ] Port 5 (wired): Corp VLAN untagged
- [ ] Port 10 (trunk): all data VLANs tagged + Default VLAN tagged (after SW2 Port 1 disconnected)
- [ ] Karl Rule: `enable ipforwarding vlan Default` on SW1
- [ ] DHCP pools configured per VLAN (SW1 only)
- [ ] `enable dhcp ports 3,5,10 vlan VLAN_X` — port 10 included for SW2 clients

**Switch Config (VOSS — if applicable)**
- [ ] IS-IS system-id + nick-name unique per switch
- [ ] manual-area identical on both: `00.0001`
- [ ] I-SID mapping identical on both switches
- [ ] Port 10 NNI: `isis enable` + `isis spbm 1`
- [ ] Port 3 FA: `auto-sense enable` + `fa enable`
- [ ] Anycast Gateway IPs on both switches
- [ ] IP Shortcuts: `ip-shortcut` under `router isis`
- [ ] ACL GUEST_ISOLATION on BOTH switches

**XIQ Network Policy**
- [ ] Policy created: [policy name]
- [ ] Corp SSID: WPA2/WPA3-PSK, VLAN [X], PSK Key Value confirmed non-empty
- [ ] Guest SSID: OWE + Transition Mode toggle ON, VLAN [X]
- [ ] IP Objects created in Common Objects (before firewall rules)
- [ ] Guest isolation rules: deny Corp Wired + deny Corp Wireless + station isolation
- [ ] User Profiles: Corp Weight 100, Guest Weight 10
- [ ] Policy validated in test environment

**Physical**
- [ ] PoE budget calculated: [X] APs × 25.5W = [Y]W — switch budget [Z]W
- [ ] Cabling tested: all AP drops pass PoE at 802.3at
- [ ] AP3000 firmware updated to latest in XIQ before deployment

## POST-DEPLOYMENT

**Verify Online**
- [ ] All APs green in XIQ (Manage → Devices)
- [ ] `show application iqagent status` → Connection: Connected

**SSID + Security**
- [ ] Corp SSID broadcasting and PSK correct (test connect)
- [ ] Guest SSID broadcasting with OWE (test on modern device)
- [ ] OWE Transition Mode: legacy device connects to Guest without password

**DHCP + Routing**
- [ ] Corp wired client: DHCP lease on VLAN [10], can ping internet
- [ ] Corp wireless client: DHCP lease on VLAN [20], can ping internet
- [ ] Guest client: DHCP lease on VLAN [30], can ping internet
- [ ] VOSS: `show ip route` on SW2 → 0.0.0.0/0 → 0.00.01 (nick-name)

**Isolation Test**
- [ ] Guest → Corp Wired: `ping 10.10.0.1` → BLOCKED
- [ ] Guest → Corp Wireless: `ping 10.20.0.1` → BLOCKED
- [ ] Guest → Guest: L2 station isolation active
- [ ] Corp → internet: `ping 8.8.8.8` → PASSES

**QoS**
- [ ] XIQ User Profiles weights confirmed
- [ ] Voice test on Corp SSID — no packet loss

**VOSS Fabric (if applicable)**
- [ ] `show isis adj` → State: UP
- [ ] `show i-sid` → all I-SIDs ELAN + Active
- [ ] `show fa assignment` → Port 3 · I-SIDs Active
- [ ] Failover test: pull Port 10 NNI → 0–1 dropped pings → plug back in → auto-heal
