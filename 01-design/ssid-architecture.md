# SSID Architecture

## The Cardinal Rule: Fewer SSIDs = Better Performance
Each SSID broadcasts beacons ~10 times per second on every radio. Every beacon occupies airtime and forces all clients — even fast ones — to wait.

| SSID Count | Beacon Overhead | Impact |
|-----------|-----------------|--------|
| 1–2 | Minimal | Recommended |
| 3–4 | Acceptable | Max for dense environments |
| 5–6 | Noticeable | Avoid if possible |
| 7+ | Significant degradation | Never recommended |

**Rule: Max 3–4 SSIDs per AP.** Horizon example: 2 (Corporate + Guest) = optimal.

## SSID Naming Convention
| Rule | Bad Example | Good Example |
|------|-------------|--------------|
| Don't reveal vendor | "Extreme-Corp" | "HorizonCorp" |
| Don't reveal floor/room | "Floor3-Guest" | "HorizonGuest" |
| Don't reveal technology | "WiFi6-Staff" | "HorizonStaff" |
| Distinguish Corp/Guest | "WiFi" | "HorizonCorp" / "HorizonGuest" |

## Standard 2-SSID Architecture (Horizon Model)
```
Corporate_Wireless  →  VLAN 20  →  WPA2/WPA3-PSK  →  QoS High (qp6)
Guest_Wireless      →  VLAN 30  →  OWE + Transition →  QoS Low  (qp1)
```
Add a third SSID only if there's a clear business need (e.g., IoT, VoIP-only, 802.1X enterprise).

## Band Steering
Push clients to 5 GHz (or 6 GHz for WiFi 6E/7 clients):
- Enable band steering in XIQ: SSID → Advanced → Band Steering
- 5 GHz has 8× more channels, far less interference
- 6 GHz: cleanest spectrum, ideal for high-density voice/video
- Legacy devices that only support 2.4 GHz remain on 2.4 GHz automatically

## Fast Roaming — 802.11k/v/r
| Standard | What It Does | When to Enable |
|----------|-------------|----------------|
| 802.11k | Neighbour Report — AP tells client which APs are nearby | Always — no downside |
| 802.11v | BSS Transition — AP can "suggest" a client roam to a better AP | Always — reduces sticky clients |
| 802.11r | Fast BSS Transition (FT) — reduces re-auth time from ~100ms to <20ms | Corp SSID with voice/video |

**Sticky client fix:** Enable 802.11v BSS Transition + lower min RSSI threshold in XIQ so AP can nudge clients to roam.

## Hidden SSID for OWE Transition Mode
OWE (Opportunistic Wireless Encryption) requires a hidden Open network alongside the OWE SSID. XIQ handles this automatically when you enable Transition Mode toggle.
- Modern devices (Android 10+, iOS 14+, Windows 10+) connect to OWE (encrypted)
- Legacy devices fall back to the hidden Open network (unencrypted but interoperable)

## Horizon SSID Design Summary
| SSID | Band | Security | VLAN | I-SID | Purpose |
|------|------|----------|------|-------|---------|
| Corporate_Wireless | 2.4+5 GHz | WPA2/WPA3-PSK | 20 | 100020 | Staff laptops/phones |
| Guest_Wireless | 2.4+5 GHz | OWE + Transition | 30 | 100030 | Visitors, internet-only |
