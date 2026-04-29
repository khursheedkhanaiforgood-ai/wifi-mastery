# User Profiles and QoS — XIQ Configuration

## What Is a User Profile?

A user profile in XIQ defines what happens to traffic AFTER a client associates: which VLAN they land on, what firewall rules apply, and what QoS treatment their traffic receives. The user profile is attached to the SSID and applied per-client.

## User Profile = VLAN + Firewall + QoS

```
Client associates to SSID
        ↓
User Profile assigned (based on SSID, or RADIUS attribute for 802.1X)
        ↓
Client tagged to VLAN (10 / 20 / 30)
        ↓
Firewall rules applied (deny Guest→Corp, permit internet)
        ↓
QoS marking applied (DSCP/CoS per traffic type)
```

## Creating a User Profile in XIQ

**Path:** Network Policy → User Profiles → + Add

| Field | Corp Value | Guest Value | Notes |
|-------|-----------|-------------|-------|
| Profile Name | `Corp-Users` | `Guest-Users` | Descriptive name |
| VLAN | 20 | 30 | Must match switch VLAN |
| Default QoS Policy | Voice-Optimised | Best-Effort | |
| Firewall Policy | Corp-FW (permit all) | Guest-FW (deny Corp, permit internet) | |
| RADIUS Attributes | Optional | Not used | For 802.1X dynamic VLAN assignment |

## QoS Policy — Traffic Classification

### Access Categories (WMM / 802.11e)

| Access Category | Traffic Type | 802.1p CoS | DSCP | Airtime Priority |
|----------------|-------------|-----------|------|-----------------|
| AC_VO | Voice (VoIP, SIP, RTP) | 6–7 | 46 (EF) | Highest |
| AC_VI | Video (streaming, conferencing) | 4–5 | 34 (AF41) | High |
| AC_BE | Best Effort (data, web) | 0–3 | 0 (BE) | Normal |
| AC_BK | Background (backup, print) | 1–2 | 10 (AF11) | Lowest |

### XIQ QoS Policy Creation
**Path:** Network Policy → QoS Policies → + Add

```
Policy: Voice-Optimised
  Rule 1: DSCP EF (46) → AC_VO
  Rule 2: DSCP AF41 (34) → AC_VI
  Rule 3: DSCP CS3 (24) → AC_VI  (Microsoft Teams/Zoom signalling)
  Rule 4: UDP 5060 (SIP) → AC_VO
  Rule 5: UDP 16384-32767 (RTP) → AC_VO
  Default: AC_BE
```

## Trust Boundary — Where DSCP Is Trusted

```
Phone (marks DSCP EF) → [AP3000] → [5320 Switch] → [Modem/Internet]
                         ↑
                   Trust starts here
                   AP honours DSCP EF → maps to AC_VO
```

**Guest SSID rule:** Do NOT trust Guest DSCP markings. A guest device could mark all traffic as DSCP EF to get priority. Apply a QoS reset policy on the Guest user profile:
```
Guest QoS: remark all DSCP to CS0 (best effort) on ingress
```

## EXOS QoS CLI Reference

```
# Check QoS policy
show policy

# Apply QoS profile to port 3 (AP port)
configure port 3 qosprofile qp6    # Voice priority

# Verify QoS on VLAN
show qosprofile vlan VLAN_20
```

QoS profiles on EXOS: qp1 (lowest) → qp8 (highest). qp6 = AC_VO equivalent.

## VOSS QoS CLI Reference

```
# Check DSCP mapping
show qos dscp-map

# Apply CoS to I-SID (VOSS maps CoS to I-SID for Fabric forwarding)
# CoS is preserved across MAC-in-MAC tunnel — no re-marking needed

# Verify QoS on interface
show qos interface gigabitEthernet 1/3
```

In VOSS/SPB Fabric: 802.1p CoS is carried inside the MAC-in-MAC header across the NNI. QoS set at the UNI edge (Port 3) is preserved end-to-end — this is a key advantage of SPB over VXLAN.

## Firewall Policy — Guest Isolation Rules

See `guest-isolation.md` for full detail. Summary:

```
Guest-FW rules (applied in order):
  1. DENY: src 10.30.0.0/24 → dst 10.10.0.0/24 (Corp Management)
  2. DENY: src 10.30.0.0/24 → dst 10.20.0.0/24 (Corp Users)
  3. PERMIT: src 10.30.0.0/24 → dst any (internet allowed)
```

**⚠ XIQ prerequisite:** IP Objects (10.30.0.0/24, 10.10.0.0/24, 10.20.0.0/24) must be created BEFORE the firewall policy can reference them.

Path: Network Policy → Common Objects → IP Objects → + Add

## User Profile for 802.1X (Future)

For enterprise deployments with RADIUS:
- RADIUS returns `Tunnel-Private-Group-ID` attribute = VLAN number
- XIQ user profile set to "RADIUS Override" → VLAN assigned dynamically
- Allows one SSID to serve multiple VLANs based on user credentials

## Horizon Lab User Profiles

| Profile | VLAN | Firewall | QoS | Auth |
|---------|------|---------|-----|------|
| Corp-Users | 20 | Permit-All | Voice-Optimised | WPA2/WPA3-PSK |
| Guest-Users | 30 | Guest-FW (deny Corp) | Best-Effort (DSCP reset) | OWE / Open |

## Common Mistakes

| Mistake | Symptom | Fix |
|---------|---------|-----|
| IP Objects not created first | Firewall rule creation fails | Create IP Objects before firewall policy |
| Guest DSCP not reset | Guest marks all traffic AC_VO, starves Corp voice | Add DSCP remark rule in Guest QoS policy |
| Wrong VLAN in user profile | Client on wrong subnet | Match user profile VLAN to switch VLAN config |
| CoS not mapped to AC on AP | Voice traffic treated as best-effort at radio | Verify DSCP EF → AC_VO rule in QoS policy |
