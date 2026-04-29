# RF Interference — Diagnosis and Mitigation

## Types of Interference

| Type | Source | Impact | Frequency |
|------|--------|--------|-----------|
| Co-channel (CCI) | Two APs on same channel, overlapping cells | Airtime contention, hidden node | 2.4 + 5 GHz |
| Adjacent-channel | APs on overlapping channels (e.g., 1+2) | Signal corruption, retransmissions | 2.4 GHz worst |
| Non-WiFi (same band) | Microwave, BT, baby monitors, DECT | Packet loss, elevated noise floor | 2.4 GHz worst |
| Radar (DFS) | Radar system in UNII-2A/2C 5 GHz band | AP must vacate channel for 30 min | 5 GHz DFS |
| Self-interference | Same building with 2 SSIDs on same channel | Controllable — fix channel plan | 2.4 + 5 GHz |

## The Noise Floor

The noise floor is the background RF energy level in the environment. A typical indoor noise floor is -95 dBm.

```
SNR = RSSI − Noise Floor

Example: RSSI -68 dBm, Noise Floor -95 dBm → SNR = 27 dB ← Good
Example: RSSI -68 dBm, Noise Floor -80 dBm → SNR = 12 dB ← Poor (interference present)
```

Elevated noise floor is the signature of non-WiFi interference.

## 2.4 GHz — Channel Plan

Only 3 non-overlapping channels exist: **1, 6, 11**.

```
NEVER use channels 2–5, 7–10, 12–14 — they overlap and cause adjacent-channel interference.

Correct 2-AP plan:  AP1=1,  AP2=6  (or 11)
Correct 3-AP plan:  AP1=1,  AP2=6,  AP3=11  (then repeat)
Wrong:              AP1=1,  AP2=3,  AP3=5   ← all overlap → catastrophic
```

**Never use 40 MHz channels on 2.4 GHz** — takes 7 of 11 channels, forcing severe overlap.

## 5 GHz — Channel Plan

5 GHz has 25 non-overlapping 20 MHz channels. UNII-2A/2C channels require DFS.

```
UNII-1 (safe, no DFS): 36, 40, 44, 48
UNII-2A (DFS required): 52, 56, 60, 64
UNII-2C (DFS required): 100, 104 ... 140
UNII-3 (safe, no DFS): 149, 153, 157, 161, 165
```

**DFS Rule:** AP must monitor for radar 60 seconds on first use (Channel Availability Check). If radar detected, AP must switch channels and not use that channel for 30 minutes. This causes unexpected outages in 5 GHz DFS channels.

**Recommendation:** Use UNII-1 + UNII-3 for high-availability deployments. Allow DFS only if ≥ 8 non-DFS channels already in use.

## 6 GHz — Clean Spectrum (WiFi 6E / WiFi 7)

6 GHz has 59 non-overlapping 20 MHz channels (or 7 × 160 MHz channels). No legacy devices, no DFS, no microwave interference.

```
PSC (Preferred Scanning Channels) — 6 GHz entry points for new clients:
  PSC-5 (channel 5), PSC-21, PSC-37, PSC-53, PSC-69, PSC-85, PSC-101, PSC-117, PSC-133, PSC-149, PSC-165, PSC-181, PSC-197, PSC-213
```

6 GHz is the strongest argument for upgrading from AP3000 (WiFi 6, no 6 GHz) to AP5010/AP5020 (WiFi 6E/7, tri-radio including 6 GHz).

## BSS Coloring (WiFi 6)

BSS Coloring is a WiFi 6 mechanism that tags each BSS with a 6-bit color (1–63). Frames from a different-color BSS are treated as background noise unless they exceed a threshold, allowing spatial reuse of the same channel.

```
Without BSS Color: AP detects another AP's transmission → defers → wasted airtime
With BSS Color:    AP detects different-color frame → checks signal level → if weak, ignores it
Result:            Higher channel reuse, lower effective CCI in dense deployments
```

## Diagnosing Interference

### Step 1 — Check SNR in XIQ
```
XIQ: Manage → Clients → [MAC] → Client 360 → Wireless tab
→ SNR column
→ < 20 dB = poor; < 10 dB = severe interference
```

### Step 2 — Check Noise Floor
```
XIQ: Manage → Devices → [AP] → Device 360 → Radio tab
→ Noise Floor value
→ > -85 dBm: elevated noise (normal is -95 to -100 dBm)
→ Elevated noise on 2.4 GHz only: likely microwave or BT
→ Elevated on all bands: building-wide interference source
```

### Step 3 — Check Retry Rate
High retry rate (>20%) indicates the medium is congested or corrupted.
```
XIQ: Device 360 → Interface tab → select AP radio → Tx/Rx Retry Rate
```

### Step 4 — Check Channel Utilisation
```
XIQ: Device 360 → Interface tab → Channel Utilisation
→ > 70%: channel too busy — consider channel width reduction or AP power reduction
```

### Step 5 — Remote Sniffer (Wireshark)
```
XIQ: Device 360 → Diagnostics → Remote Sniffer → select radio
→ Capture frames on the affected channel
→ Filter: wlan.fc.retry == 1 (retry frames)
→ High retry count = CCI or near-far problem
```

## Mitigation Strategies

### 2.4 GHz Congestion
1. Move capable clients to 5 GHz with band steering
2. Disable 2.4 GHz on APs in very dense areas (if 5 GHz coverage is complete)
3. Set min basic rate ≥ 12 Mbps (reduces sticky legacy client cell radius)
4. Fix channel plan: 1/6/11 only, never 40 MHz

### 5 GHz DFS Channel Drop
1. Switch to UNII-1 or UNII-3 channels for critical APs
2. Enable RRM (Radio Resource Management) in XIQ — auto-channel selection avoids DFS
3. If DFS must be used: monitor for 30-minute channel-switch events in XIQ Events

### Non-WiFi Interference (Microwave, BT)
1. Move 2.4 GHz APs to channels away from identified interference source
2. If in break room: use channel 11 (microwaves tend to impact channels 1–6)
3. WiFi 6 TWT helps IoT/BT coexistence — scheduled wake windows reduce contention
4. Long-term: deploy AP5010/5020 and move traffic to 6 GHz (zero non-WiFi interference)

### Co-Channel Interference in Dense Deployments
1. Reduce AP transmit power (start at 17 dBm, reduce to 14 dBm if CCI visible)
2. Increase channel width only if co-channel APs are at least 2 cells apart
3. Enable BSS Coloring (WiFi 6) — automatic in XIQ for WiFi 6 APs
4. Enable multi-AP coordination (WiFi 7 feature on AP5020)

## RRM — Radio Resource Management

XIQ RRM automatically adjusts AP channel and power to minimize interference.

```
Configure → Network Policies → Device Template → Radio Settings
→ Enable Auto Channel Selection
→ Enable Auto Power Control
→ Schedule: nightly (off-hours) to avoid disruption
```

RRM scans the environment and reassigns channels/power. After RRM runs, verify:
- No two adjacent APs on the same channel
- RSSI heat map shows smooth coverage (no holes > -75 dBm)

## Quick Reference

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| Low SNR (<20 dB) | Non-WiFi interference or CCI | Spectrum scan; channel change; power reduce |
| High retry rate (>20%) | CCI or hidden node | Fix channel plan; enable BSS Coloring |
| DFS channel drop | Radar detected | Move to UNII-1/3; enable RRM |
| 2.4 GHz worse than 5 GHz | 2.4 GHz congestion | Band steering; min rate increase |
| All clients slow in one area | AP or AP channel overloaded | Check channel utilisation; add AP or change channel |
| Noise floor elevated site-wide | External interference source | Walk site with spectrum analyser; identify source |
