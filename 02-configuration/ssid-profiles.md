# SSID Profiles — XIQ Configuration

## What Is an SSID Profile?

An SSID profile in XIQ defines the radio behaviour for a single WiFi network: name, band selection, rates, beacon interval, client density settings. It is separate from the security profile (WPA2/WPA3) and the user profile (VLAN/firewall rules). All three are bundled into a Network Policy.

## SSID Profile Settings — Key Fields

| Field | Corp Recommended | Guest Recommended | Notes |
|-------|-----------------|------------------|-------|
| SSID Name | `Horizon-Corp` | `Horizon-Guest` | Match naming convention |
| Broadcast SSID | Enabled | Enabled | Disable only for IoT hidden SSIDs |
| Band | 2.4 + 5 GHz | 2.4 + 5 GHz | Use dual-band; 6 GHz for WiFi 6E/7 APs |
| Band Steering | Enabled | Enabled | Push capable clients to 5 GHz |
| 802.11k | Enabled | Enabled | Neighbour report — faster roaming target selection |
| 802.11v | Enabled | Enabled | AP nudges sticky clients |
| 802.11r (FT) | Enabled | Disabled | FT: 20ms re-auth for Corp; not needed for Guest |
| Min Basic Rate | 12 Mbps (2.4) / 24 Mbps (5) | 6 Mbps | Higher min rate = smaller cell = less airtime waste |
| Beacon Interval | 100 ms (default) | 100 ms | Lower = more overhead; don't change without reason |
| DTIM Period | 1 (voice) / 3 (standard) | 3 | Higher = better IoT battery; 1 for latency-sensitive |
| Max Clients per Radio | 50 | 30 | AP3000 2×2 MIMO — don't exceed 50 per radio |

## Band Steering Logic

```
Client connects to 2.4 GHz → AP checks: is client 5 GHz capable?
  Yes → AP sends 802.11v BSS Transition Request to 5 GHz BSSID
  Client moves → now on 5 GHz (less congestion, higher rates)
  No (legacy 2.4-only device) → stays on 2.4 GHz
```

WiFi 7 note: MLO-capable clients ignore band steering — they use ALL bands simultaneously.

## Minimum Rate Setting — Why It Matters

Low minimum rates (1–2 Mbps) allow clients to stay associated at very low signal. One client at MCS0 (1 Mbps) can consume 50× more airtime than a client at MCS7 (54 Mbps) to transmit the same data.

**Rule:** Set min basic rate ≥ 12 Mbps on 2.4 GHz for office deployments. This shrinks the cell boundary to clients with usable signal.

## SSID Profile — Advanced Settings

### Client Load Balancing
XIQ can distribute clients across APs when multiple APs hear the same client at similar RSSI.
- Enable when: ≥ 2 APs in same area (conference rooms, open plan office)
- Threshold: trigger when AP exceeds 80% of max clients

### Airtime Fairness
Prevents slow legacy clients from dominating the medium. Each client gets equal *time*, not equal *throughput*.
- Enable on: all SSIDs where mixed-generation clients exist (WiFi 4/5/6 sharing a BSS)

### Fast Roaming (802.11r) — Enabling in XIQ
Path: Network Policy → SSID → Advanced → Fast BSS Transition
- Must enable on **both** source and target AP
- OKC (Opportunistic Key Caching) is the WPA2-Enterprise equivalent — enable under RADIUS settings

## WiFi 6 / 6E / 7 SSID Considerations

| Feature | Action in XIQ | Impact |
|---------|--------------|--------|
| OFDMA | Auto-enabled on WiFi 6 APs | Serves multiple clients simultaneously in dense areas |
| BSS Coloring | Auto-enabled | Reduces co-channel interference detection overhead |
| TWT | Enable for IoT SSID | Extends IoT battery life dramatically |
| 6 GHz (WiFi 6E) | Enable 6 GHz radio + 6 GHz band on SSID profile | Clean spectrum, PSC channels, no DFS |
| MLO (WiFi 7) | Supported on AP5020 and newer — not AP3000 | Clients aggregate 2.4+5+6 GHz simultaneously |

## Horizon Lab SSID Profiles Summary

| Profile | Band | 802.11r | Band Steering | Min Rate | Notes |
|---------|------|---------|--------------|---------|-------|
| Corp-Profile | 2.4+5 GHz | Enabled | Enabled | 12/24 Mbps | WPA2/WPA3 Transition |
| Guest-Profile | 2.4+5 GHz | Disabled | Enabled | 6/12 Mbps | OWE Transition Mode |

## Common Mistakes

| Mistake | Symptom | Fix |
|---------|---------|-----|
| 802.11r enabled on Guest open network | Association failures on legacy devices | Disable FT on open/OWE SSIDs |
| Min rate left at 1 Mbps | Slow network in dense area | Set 12 Mbps on 2.4 GHz |
| Band steering off | All capable clients pile onto 2.4 GHz | Enable in SSID profile |
| Max clients not set | AP overloaded with 80+ clients | Set max 50 per radio for AP3000 |
