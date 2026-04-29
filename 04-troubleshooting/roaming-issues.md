# Roaming Issues — Diagnosis and Fix

## What Is Sticky Client?

A sticky client is a client that refuses to roam to a closer AP even when its current AP signal is poor. The 802.11 standard gives roaming control to the CLIENT — the AP cannot force a disconnection (without 802.11v). Legacy clients are notorious for staying on a distant AP at -80 dBm while a closer AP at -55 dBm broadcasts at the same power.

## Roaming Standards — Quick Recap

| Standard | What It Does | Latency Impact |
|---------|-------------|---------------|
| 802.11k | Neighbour Report — AP sends client list of nearby APs with signal data | Faster target AP selection (no full scan needed) |
| 802.11v | BSS Transition Management — AP suggests client should roam | Solves sticky client (AP can nudge) |
| 802.11r (FT) | Fast BSS Transition — PMK cached at target AP | Re-auth: ~100ms → <20ms |
| OKC | PMK shared between APs | WPA2-Enterprise fast roaming |

Enable all three (k/v/r) for Corp SSIDs. Do not enable 802.11r on open/OWE Guest SSIDs.

## Diagnosing Roaming Issues

### Step 1 — Is Roaming Happening At All?
```
XIQ: Manage → Clients → [MAC] → Client 360
→ Connection History tab
→ Look for "Roam" events in the timeline
```
If no roam events: client is sticky. Check 802.11v is enabled and target AP RSSI is significantly better.

### Step 2 — Is Roaming Too Slow?
```
XIQ: Client 360 → SLA Tab
→ Latency spike during roam?
→ If latency > 100ms during roam: 802.11r may not be negotiated
```

On EXOS/VOSS: roam events logged in XIQ Events. Filter: Category = Wireless, Event = Roam.

### Step 3 — Is the Client Dropping During Roam?
```
XIQ: Client 360 → SLA Tab → Packet Loss
→ > 0.1% packet loss spike during roam = full re-authentication happening
→ Means 802.11r not active for that client
```

### Step 4 — CLI Verification
```
# VOSS — check IS-IS adjacency (roaming crosses fabric)
show isis adj             # must be UP — slow roam if fabric degraded

# EXOS — check AP port up and operational
show port 3 information detail
show port 5 information detail
```

## Common Roaming Problems and Fixes

### Problem 1: Client Stuck on Distant AP (Sticky)
**Symptom:** Client at -78 dBm on AP1 while AP2 is -55 dBm, no roam
**Root cause:** 802.11v not enabled, or client ignoring BSS Transition Request

**Fix:**
1. Enable 802.11v in XIQ SSID profile
2. Enable 802.11k (neighbour report) so client has AP2's info ready
3. Lower AP transmit power (reduces "voice" of AP1 so client threshold triggers)
4. Lower min RSSI threshold if configurable per AP in XIQ

### Problem 2: Slow Roam / VoIP Drops During Roam
**Symptom:** VoIP call breaks momentarily when walking between APs
**Root cause:** 802.11r not negotiated (client or AP not supporting FT)

**Fix:**
1. Verify 802.11r enabled on BOTH source AND destination AP SSID profile
2. Check client: iOS/Android support 802.11r; some Windows clients need registry tweaks
3. Consider OKC as fallback for WPA2-Enterprise if 802.11r not universal

```
XIQ: Network Policy → SSID → Advanced
→ Fast BSS Transition: Enabled
→ OKC: Enabled (for WPA2-Enterprise)
```

### Problem 3: Client Loops / Disconnects on Roam
**Symptom:** Client roams then immediately disconnects, reconnects
**Root cause:** 802.11r key mismatch, or client doesn't support FT properly

**Fix:**
1. Enable WPA2/WPA3 Transition Mode — allows FT and non-FT clients on same SSID
2. For incompatible legacy client: disable 802.11r on that SSID (Guest only — don't disable Corp)

### Problem 4: Roaming Works But XIQ Client 360 Shows Gaps
**Symptom:** Client 360 shows time offline during roam
**Root cause:** AP registration of roam event delayed (not a real disconnect)
**Fix:** Not a real issue — cosmetic in XIQ. Confirm via SLA tab latency graph; if <100ms, roam succeeded.

### Problem 5: 6 GHz Clients Don't Roam to 6 GHz (WiFi 6E/7)
**Symptom:** WiFi 6E/7 client stays on 5 GHz, never moves to 6 GHz
**Root cause:** 6 GHz SSID profile not broadcasting, or PSC channels not advertised

**Fix:**
1. Verify 6 GHz radio enabled on AP (AP5010/5020 only — not AP3000)
2. Confirm PSC (Preferred Scanning Channels) advertised in 6 GHz beacon
3. Enable 802.11k on 6 GHz band — client needs neighbour report to find 6 GHz AP

For WiFi 7 MLO: roaming concept changes. With MLO, client maintains simultaneous links on 2.4+5+6 GHz. A "roam" becomes a link state change (one link degrades, another takes over) — sub-millisecond, transparent to the application.

## Roaming Test Procedure

1. Connect voice call (VoIP app or Microsoft Teams) on Corp SSID
2. Walk from AP1 toward AP2 (opposite end of building)
3. Monitor in XIQ Client 360: look for roam event + SLA tab latency
4. Expected: latency spike <20ms (802.11r) or <50ms (no FT)
5. VoIP call should stay connected with no perceptible glitch

## RSSI Thresholds for Roaming Decision

| RSSI at Current AP | Expected Behaviour |
|--------------------|--------------------|
| ≥ -65 dBm | Stay on current AP |
| -65 to -70 dBm | Client considers roaming (802.11v nudge zone) |
| -70 to -75 dBm | Client should roam if better AP visible |
| < -75 dBm | Client must roam — below usable threshold |
| < -80 dBm | Sticky client territory — 802.11v essential |

## Quick Reference

| Symptom | Check | Fix |
|---------|-------|-----|
| Sticky client | 802.11v enabled? | Enable 802.11v + 802.11k in SSID profile |
| Slow VoIP roam | 802.11r negotiated? | Enable FT on source + target SSID profile |
| Disconnect on roam | Client FT compatibility | WPA2/WPA3 Transition Mode; or disable FT for Guest |
| 6 GHz never used | AP supports 6 GHz? PSC broadcast? | AP5010/5020; enable 6 GHz radio; check PSC channels |
| IS-IS causes slow roam | NNI degraded? | `show isis adj` — must be UP |
