# QoS Design for WiFi

## WMM / 802.11e — Access Categories
All WiFi 6/6E/7 APs support WMM. Traffic is placed into one of four access categories (AC):

| Access Category | 802.11e AC | 802.1p CoS | DSCP | Typical Traffic |
|----------------|-----------|------------|------|----------------|
| Voice | AC_VO | 6–7 | EF (46), CS7 | VoIP, real-time audio |
| Video | AC_VI | 4–5 | AF4x, CS4 | Video conferencing, streaming |
| Best Effort | AC_BE | 0, 3 | CS0, AF3x | HTTP, email, general data |
| Background | AC_BK | 1–2 | CS1, AF1x | Bulk transfers, backups |

Higher AC = shorter contention window = wins medium access more often.

## DSCP Quick Reference
| DSCP Value | Decimal | Traffic Type |
|-----------|---------|-------------|
| EF | 46 | VoIP bearer (voice packets) |
| CS7 | 56 | Network control |
| AF41 | 34 | Video (interactive) |
| AF31 | 26 | Mission-critical data |
| CS0 / BE | 0 | Default / best effort |
| CS1 | 8 | Scavenger / low priority |

## EXOS QoS Profile Mapping
```
# Assign Corp VLANs to high-priority queue
configure qosprofile qp6 name "Corporate-Express" ports 3,10
configure qosprofile qp1 name "Guest-BestEffort" ports 3,10

# Apply per VLAN
configure qos type-of-service vlan VLAN_20 qosprofile qp6
configure qos type-of-service vlan VLAN_30 qosprofile qp1
```

## XIQ User Profile Scheduling Weights (Horizon)
| Profile | SSID | Weight | Effect |
|---------|------|--------|--------|
| Corp_20 | Corporate_Wireless | 100 | Gets ~10× more airtime than guest |
| Guest_30 | Guest_Wireless | 10 | Internet access — capped throughput |

XIQ path: Network Policy → User Profiles → [profile] → Scheduling Weight

## Trust Boundary — Don't Trust Client DSCP on Guest
Clients can self-mark their traffic with any DSCP value. On untrusted networks (Guest), always remark at the switch/AP ingress:
- Accept/trust DSCP marks from Corp devices (managed, domain-joined)
- Remark/ignore DSCP from Guest clients — assign CS0 (best effort) regardless

## Voice/Video SSID Best Practices
- Enable 802.11r (Fast BSS Transition) — reduces re-auth from ~100ms to <20ms
- Minimum RSSI threshold: -67 dBm (clients below this threshold are forced to roam)
- Band steering: prefer 5 GHz or 6 GHz for voice
- DTIM interval: 1 for voice SSIDs (reduces wake-up latency for mobile clients)
- Load balancing: max 20–25 voice clients per radio

## Horizon QoS Summary
```
SW1 CLI (EXOS):
  VLAN 10 (Corp Wired)     → qp6 (high)
  VLAN 20 (Corp Wireless)  → qp6 (high)
  VLAN 30 (Guest Wireless) → qp1 (low)

XIQ:
  Corp_20 User Profile     → Scheduling Weight: 100
  Guest_30 User Profile    → Scheduling Weight: 10
```
