# 802.11 Standards Reference

## WiFi Generation Timeline
| Generation | Standard | Year | Bands | Max PHY Rate | Key Feature |
|-----------|---------|------|-------|-------------|-------------|
| — | 802.11 | 1997 | 2.4 GHz | 2 Mbps | Original WiFi |
| WiFi 1 | 802.11b | 1999 | 2.4 GHz | 11 Mbps | DSSS — mass market |
| WiFi 2 | 802.11a | 1999 | 5 GHz | 54 Mbps | OFDM, 5 GHz pioneer |
| WiFi 3 | 802.11g | 2003 | 2.4 GHz | 54 Mbps | OFDM on 2.4 GHz |
| WiFi 4 | 802.11n | 2009 | 2.4+5 GHz | 600 Mbps | MIMO, 40 MHz, spatial streams |
| WiFi 5 | 802.11ac | 2013 | 5 GHz only | 3.5 Gbps | MU-MIMO downlink, 80/160 MHz |
| WiFi 6 | 802.11ax | 2019 | 2.4+5 GHz | 9.6 Gbps | OFDMA, BSS Coloring, TWT, uplink MU-MIMO |
| WiFi 6E | 802.11ax (ext) | 2021 | 2.4+5+6 GHz | 9.6 Gbps | Adds 6 GHz band (WiFi 6 features in clean spectrum) |
| WiFi 7 | 802.11be | 2024 | 2.4+5+6 GHz | 46 Gbps | Multi-Link Operation (MLO), 320 MHz, 4K-QAM, CMU-MIMO |

## WiFi 6 Key Technologies
| Technology | What It Does | Benefit |
|-----------|-------------|---------|
| OFDMA | Divides channel into Resource Units — serves multiple clients simultaneously | Eliminates per-client overhead in dense environments |
| BSS Coloring | Tags frames with a color to identify overlapping BSSs | Reduces co-channel interference, more efficient medium reuse |
| TWT (Target Wake Time) | AP schedules when each client wakes up | Dramatically extends IoT battery life |
| Uplink MU-MIMO | Multiple clients transmit simultaneously to AP | Doubles uplink capacity vs WiFi 5 |
| 1024-QAM | Higher modulation order | +25% throughput vs 802.11ac 256-QAM |

## WiFi 7 Key Technologies (802.11be)
| Technology | What It Does | Benefit |
|-----------|-------------|---------|
| Multi-Link Operation (MLO) | Device uses 2.4+5+6 GHz simultaneously | Aggregated throughput, sub-millisecond latency, seamless failover per-link |
| 320 MHz Channels | Double the width of WiFi 6 in 6 GHz | Up to 2× peak throughput |
| 4K-QAM | 12 bits per symbol (vs 10-bit 1024-QAM) | +20% throughput gain |
| Multi-AP Coordination | APs coordinate transmissions | Reduces interference in dense deployments |
| CMU-MIMO | Coordinated Multi-User MIMO across APs | Enterprise-grade spatial reuse |

## Security Standards Timeline
| Standard | Year | What It Replaced | Key Change |
|---------|------|-----------------|-----------|
| WEP | 1997 | Nothing (original) | RC4 — broken, never use |
| WPA | 2003 | WEP | TKIP — transitional fix |
| WPA2 | 2004 | WPA | AES-CCMP — still widely used |
| 802.11w (PMF) | 2009 | — | Protected Management Frames |
| WPA3 | 2018 | WPA2 | SAE, OWE, PMF mandatory |
| WPA3-Enterprise 192-bit | 2018 | — | GCMP-256, HMAC-SHA-384 |

## Roaming Standards
| Standard | Function | Latency Reduction |
|---------|----------|------------------|
| 802.11k | Neighbour Report — AP tells client which APs are nearby | Faster target AP selection |
| 802.11v | BSS Transition Management — AP suggests client roam | Reduces sticky client problem |
| 802.11r | Fast BSS Transition (FT) — PMK cached at target AP | Re-auth: 100ms → <20ms |
| OKC | Opportunistic Key Caching — PMK shared between APs | WPA2-Enterprise fast roaming |

## WiFi 6E vs WiFi 7 — What Changes for Enterprises
| Feature | WiFi 6E | WiFi 7 |
|---------|---------|-------|
| 6 GHz support | Yes | Yes (expanded) |
| Max channel width | 160 MHz | 320 MHz (6 GHz only) |
| Multi-link | No | Yes — MLO across 2.4+5+6 GHz simultaneously |
| Latency | <10ms | <1ms (MLO + HARQ) |
| AP coordination | No | Yes (multi-AP) |
| Backward compat | WiFi 4/5/6 clients work | All legacy clients work |

## Extreme Networks AP3000 — Generation Support
- AP3000: WiFi 6 (802.11ax) — 2.4+5 GHz, 2×2 MIMO
- AP3000X: Outdoor variant of AP3000
- For WiFi 6E/7: look at Extreme AP5010/AP5020 series (tri-radio)
