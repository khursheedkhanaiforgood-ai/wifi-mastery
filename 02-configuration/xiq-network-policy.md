# XIQ Network Policy — Step-by-Step

## Create a New Policy
1. Configure → Network Policies → **+ Add Network Policy**
2. Name: `AP_April21_KarlLab` (or site-specific name)
3. Select policy type: **Wireless**

## Add SSIDs
Inside the policy → SSIDs tab → **+ Add SSID**

### Corporate SSID
- SSID Name: `Corporate_Wireless`
- Broadcast SSID: Yes
- Security: **WPA2/WPA3 Personal** (transition mode)
- Key Value: **[MUST SET — leave empty = clients can't connect]**
- User Profile: Corp_20
- VLAN: 20

### Guest SSID
- SSID Name: `Guest_Wireless`
- Broadcast SSID: Yes
- Security: **Enhanced Open (OWE)**
- Transition Mode: **[ENABLE TOGGLE — "Transition Mode for 2.4GHz and 5GHz"]**
- User Profile: Guest_30
- VLAN: 30

## Create User Profiles
Policy → User Profiles tab → **+ Add**

### Corp_20 Profile
- Name: `Corp_20`
- Default Action: Accept
- Scheduling Weight: **100**
- VLAN: 20
- Firewall Rules: (none — Corp has full access)

### Guest_30 Profile
- Name: `Guest_30`
- Default Action: Accept
- Scheduling Weight: **10**
- VLAN: 30
- Firewall Rules: **[add 3 rules — see guest-isolation.md]**

## Device Templates (Switch Ports)
Policy → Switch Template → Port Assignment:
- Port 3: Auto-Sense + **Fabric Attach** (VOSS) or Trunk (EXOS)
- Port 5: Access — VLAN 10
- Port 10: NNI (VOSS) or Trunk (EXOS)

For Fabric Attach ports: map VLAN → I-SID:
- VLAN 20 → I-SID 100020
- VLAN 30 → I-SID 100030

## Push the Policy
1. Manage → Devices → select target APs
2. **Update Devices → Complete Configuration Update** (use Complete for first push)
3. Monitor: Manage → Events → filter for Update events

## Common Pitfalls
| Issue | Root Cause | Fix |
|-------|-----------|-----|
| Corp SSID connects but no internet | Key Value field empty | Set PSK in SSID Security settings |
| Guest legacy devices can't connect | OWE Transition Mode not toggled | SSID → enable Transition Mode toggle |
| Firewall rule "IP object must be saved first" | IP Object not created before rule | Create in Common Objects first |
| APs not applying policy | Used Partial update | Use Complete Configuration Update |
| FA not provisioning VLANs on AP port | Auto-Sense/FA not in device template | Check Switch Template port settings |
