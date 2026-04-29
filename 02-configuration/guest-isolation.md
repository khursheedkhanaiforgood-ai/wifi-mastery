# Guest Isolation

## What Needs to Be Isolated
1. **Guest → Corp Wired** (subnet block)
2. **Guest → Corp Wireless** (subnet block)
3. **Guest → Guest** (L2 station isolation)

## Step 1: Create IP Objects in Common Objects
XIQ → Configure → Common Objects → IP Objects → **+ Add**

**Create these BEFORE adding any firewall rules** (order matters — rule creation fails if IP object doesn't exist yet):

| Object Name | Subnet | Description |
|------------|--------|-------------|
| `Guest_Network` | 10.30.0.0/24 | Guest VLAN source |
| `Corp_Wired` | 10.10.0.0/24 | Corporate wired subnet |
| `Corp_Wireless` | 10.20.0.0/24 | Corporate wireless subnet |

## Step 2: Add Firewall Rules to Guest_30 User Profile
Network Policy → User Profiles → Guest_30 → Firewall Rules → **+ Add Rule**

| Rule # | Action | Source | Destination | Protocol | Notes |
|--------|--------|--------|-------------|----------|-------|
| 1 | **DENY** | Guest_Network | Corp_Wired | Any | Block Guest → Corp wired |
| 2 | **DENY** | Guest_Network | Corp_Wireless | Any | Block Guest → Corp wireless |
| 3 | **DROP (station)** | — | — | — | L2 Guest-to-Guest isolation |

Rule 3 uses XIQ's "Drop traffic between stations" action — this is an AP-level L2 block. ACL rules alone cannot stop same-VLAN client-to-client traffic; this requires the AP to drop at the radio level.

## Step 3: Push Policy
Update Devices → Complete Configuration Update

## Verify Isolation
From a guest client device:
```bash
ping 10.10.0.1    # Corp Wired gateway — should FAIL (timeout)
ping 10.20.0.1    # Corp Wireless gateway — should FAIL (timeout)
ping 8.8.8.8      # Internet — should PASS
ping [other guest client IP]  # should FAIL (station isolation)
```

## VOSS — Critical Posture Change
In EXOS, SW2 was Pure L2 — all guest traffic flowed through SW1 where the single ACL lived.
In VOSS, SW2 owns the Anycast Gateway and routes locally. **Guest traffic from SW2 clients bypasses SW1 entirely.**

**Rule: ACL must be on BOTH SW1 and SW2 in VOSS.**

```
# Apply on BOTH SW1 and SW2
ip filter acl GUEST_ISOLATION
  deny ip 10.30.0.0/24 10.10.0.0/24
  deny ip 10.30.0.0/24 10.20.0.0/24
  permit ip any any
  exit

vlan filter ip GUEST_ISOLATION 30 in
```

VOSS L2 bonus: I-SIDs use MAC-in-MAC (802.1ah) encapsulation. Guest (I-SID 100030) and Corp (I-SID 100020) are logically invisible to each other at L2 by default. L3 ACL is still required for the inter-VLAN routing block.

## Why ACL-Only Doesn't Stop Guest-to-Guest L2 Traffic
An ACL filters at Layer 3 (IP). Two clients in the same VLAN (same subnet) communicate directly at Layer 2 — the packet never goes through the router, so the ACL is never evaluated. This is why you need the XIQ "Drop traffic between stations" rule, which acts at the AP radio level before L2 forwarding.
