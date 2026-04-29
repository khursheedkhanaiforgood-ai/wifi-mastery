# Client Connectivity Troubleshooting

## Decision Tree — Layer 1 Up
Work through these in order. Don't skip layers.

### Step 1 — Is the AP Powered and Online?
```
show poe port 3           # Is PoE on? Class 4 = PoE+ = good
show application iqagent status   # Connection: Connected = XIQ online
```
XIQ: Manage → Devices → AP status green?

### Step 2 — Is the SSID Broadcasting?
Client device: can it see the SSID in WiFi scan?
- If SSID invisible: check XIQ policy push completed; check SSID broadcast enabled
- If SSID visible: continue to Step 3

### Step 3 — Can the Client Associate?
Client says "Authentication failed" or loops connecting:
- **PSK issue:** Check Key Value in XIQ SSID profile — common mistake: field left empty
- **WPA3 mismatch:** Legacy device can't do WPA3-SAE → enable WPA2/WPA3 Transition Mode
- **OWE issue:** Legacy device can't do OWE → enable OWE Transition Mode toggle in Guest SSID
- **PMF mismatch:** Client doesn't support PMF Required → change to PMF Optional for WPA2

XIQ: Client 360 → look at Authentication Status

### Step 4 — Is the Client Getting a DHCP Address?
Client has 169.254.x.x (APIPA) = no DHCP lease.

```
# EXOS: check DHCP leases on that VLAN
show dhcp lease vlan VLAN_20

# Verify DHCP is enabled on the right ports
show dhcp configuration vlan VLAN_20
# Must show: ports 3,5,10 included — port 10 needed for SW2 clients
```

Common DHCP issues:
- DHCP not enabled on port 10 → SW2 clients get no IP
- DHCP pool exhausted → check lease count vs pool range
- Wrong VLAN on port → client is on wrong subnet

### Step 5 — Can the Client Reach the Gateway?
Client has IP but can't ping `10.20.0.1` (gateway):
```
# EXOS — Karl Rule
show ipconfig vlan VLAN_20
# Must show: IP forwarding: Enabled
# If disabled: enable ipforwarding vlan VLAN_20

# EXOS — verify SVI exists
show iproute
# Must show a route to 10.20.0.0/24

# VOSS — verify Anycast IP configured
show ip interface vlan 20
# Must show: ip address 10.20.0.1/24
```

### Step 6 — Can the Client Reach the Internet?
Gateway responds but `ping 8.8.8.8` fails:
```
# EXOS
show iproute           # default route present? → 0.0.0.0/0 via 192.168.0.1
enable ipforwarding vlan Default    # Karl Rule — if missing, clients get IPs but no internet

# VOSS
show ip route          # 0.0.0.0/0 → 192.168.0.1 (SW1) or → 0.00.01 nick-name (SW2)
# If missing on SW2: check ip-shortcut under router isis on both switches
```

Modem static routes: verify modem has routes to 10.10/20/30.0.0/24 → 192.168.0.28 (SW1).

### Step 7 — Is an ACL Blocking Traffic?
Guest client can't reach internet:
```
# EXOS: check policy
show policy

# VOSS: check ACL
show ip filter acl GUEST_ISOLATION

# Verify ACL applied to VLAN
show vlan filter ip
```
Guest isolation rules should DENY Guest → Corp, but PERMIT Guest → internet.

## Quick Reference Card
| Symptom | Most Likely Cause |
|---------|------------------|
| Can't see SSID | Policy not pushed / SSID hidden |
| "Wrong password" on Corp | Key Value empty in XIQ |
| Legacy device can't join Guest | OWE Transition Mode not toggled |
| Gets 169.254.x.x | DHCP not enabled or port missing from DHCP config |
| IP but no gateway ping | Karl Rule missing (EXOS) or Anycast IP not set (VOSS) |
| Gateway ok but no internet | Default route missing / modem static routes missing |
| Guest can reach Corp | Guest isolation rules not applied or ACL only on SW1 (VOSS) |
