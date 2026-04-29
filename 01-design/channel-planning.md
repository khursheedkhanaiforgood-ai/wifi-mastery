# Channel Planning

## 2.4 GHz — Use ONLY Channels 1, 6, 11
These are the only three non-overlapping channels in 2.4 GHz (20 MHz width).
**Never use 40 MHz channel width in 2.4 GHz** — it overlaps all three non-overlapping channels.

```
Ch 1  [1──────6]
Ch 6       [6──────11]
Ch 11            [11──────16]
```

Adjacent APs must use different channels from this set. Never use Ch 2/3/4/5/7/8/9/10 as primary.

## 5 GHz UNII Bands
| Band | Channels | DFS Required | Notes |
|------|----------|-------------|-------|
| UNII-1 | 36, 40, 44, 48 | No | Preferred — indoor, no radar risk |
| UNII-2A | 52, 56, 60, 64 | Yes | Radar check before use |
| UNII-2C | 100–144 | Yes | Radar check — wide band |
| UNII-3 | 149, 153, 157, 161, 165 | No | Preferred — widely used outdoors + indoors |

**DFS (Dynamic Frequency Selection):** AP must listen for radar 60 seconds before transmitting. If radar detected, AP vacates channel and moves. Can cause ~1 min interruption. Use non-DFS channels (UNII-1 and UNII-3) for voice/video APs.

## 6 GHz — WiFi 6E and WiFi 7
59 non-overlapping 20 MHz channels. Clean spectrum — no legacy devices, no interference.

**Preferred Scanning Channels (PSC) for 80 MHz:**
5, 21, 37, 53, 69, 85, 101, 117, 133, 149, 165, 181, 197, 213, 229

**Channel widths available:**
- 20 MHz — maximum range, minimum throughput
- 40 MHz — balanced
- 80 MHz — recommended for capacity deployments
- 160 MHz — maximum throughput, shorter range
- 320 MHz — WiFi 7 only (802.11be)

## WiFi 7 Multi-Link Operation (MLO)
WiFi 7 (802.11be) introduces Multi-Link Operation — a device can simultaneously use 2.4 + 5 + 6 GHz at the same time. Traffic is aggregated and load-balanced across all bands.
- Requires AP and client both support WiFi 7
- Effectively eliminates per-band throughput bottlenecks
- AP selection: look for tri-radio APs with MLO capability

## Channel Width by Deployment Type
| Deployment | 2.4 GHz | 5 GHz | 6 GHz |
|-----------|---------|-------|-------|
| High density (office/conference) | 20 MHz | 40 MHz | 80 MHz |
| Medium density (standard office) | 20 MHz | 80 MHz | 80/160 MHz |
| Low density (home/small office) | 20 MHz | 80–160 MHz | 160 MHz |
| Outdoor long-range | 20 MHz | 20–40 MHz | N/A (range limited) |

## AP Spacing Guidelines
| Density | 5 GHz Spacing | 2.4 GHz Spacing |
|---------|--------------|-----------------|
| High density (≥1 AP per room) | 15–20 ft (5–7m) | Disable 2.4 or reduce power |
| Standard office | 20–30 ft (7–10m) | 40–50 ft |
| Open warehouse | 40–60 ft (15–20m) | 60–80 ft |
| Outdoor | 50–100 ft+ | 75–150 ft+ |

## Channel Reuse Pattern — 5 GHz (4 AP grid)
```
AP1: Ch 36   AP2: Ch 44   AP3: Ch 149  AP4: Ch 157
AP5: Ch 149  AP6: Ch 157  AP7: Ch 36   AP8: Ch 44
```
Separate same-channel APs by ≥ 2 AP positions.

## Common Channel Planning Mistakes
1. Using 40 MHz in 2.4 GHz — avoid
2. All APs on same 5 GHz channel — massive co-channel interference
3. Maximum TX power on all APs — creates loud AP / deaf client problem
4. Ignoring DFS behaviour on voice APs — 60-second outage during radar detection
5. Not planning 6 GHz separately from 5 GHz channel assignments
