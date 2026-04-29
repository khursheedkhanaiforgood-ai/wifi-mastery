# RF Fundamentals

## Signal Strength Targets (RSSI)
| Use Case | Target RSSI | Minimum RSSI |
|----------|-------------|--------------|
| Voice / VoIP / Video calls | -65 dBm | -67 dBm |
| High-density data (office) | -67 dBm | -70 dBm |
| General data | -70 dBm | -75 dBm |
| IoT / sensors | -75 dBm | -80 dBm |
| Background/backhaul | -80 dBm | -85 dBm |

## SNR Targets
| Use Case | Target SNR | Minimum SNR |
|----------|-----------|-------------|
| Voice / VoIP | ≥ 25 dB | 20 dB |
| Video streaming | ≥ 20 dB | 15 dB |
| Data | ≥ 15 dB | 10 dB |

## Band Comparison
| Band | Frequency | Non-Overlapping 20MHz Channels | Range | Wall Penetration | Congestion |
|------|-----------|-------------------------------|-------|------------------|------------|
| 2.4 GHz | 2400–2484 MHz | 3 (ch 1/6/11) | Long (~35m indoors) | Best | Very High |
| 5 GHz | 5150–5850 MHz | 25 | Medium (~15m indoors) | Good | Low |
| 6 GHz (WiFi 6E/7) | 5925–7125 MHz | 59 | Short (~10m indoors) | Poor | Minimal |

## Free-Space Path Loss
- Every **doubling of distance** = ~6 dB more loss
- Every **10× distance increase** = 20 dB more loss
- Rule of thumb: go from 1m to 10m = 20 dB drop; 10m to 20m = 6 dB more

## Common Obstacles (Attenuation)
| Obstacle | Approximate Loss |
|----------|-----------------|
| Drywall / partition | 3–5 dB |
| Brick wall | 6–18 dB |
| Concrete wall | 15–40 dB |
| Glass window | 2–3 dB |
| Metal shelving | 20–40 dB |
| Human body | 3–5 dB |
| Floor/ceiling | 15–25 dB |
| Elevator shaft | 40+ dB (effective barrier) |

## dBm Quick Reference
| dBm | Approximate mW | Field Meaning |
|-----|---------------|---------------|
| +20 | 100 mW | Max legal outdoor transmit |
| +17 | 50 mW | Typical AP max indoor |
| 0 | 1 mW | Reference point |
| -30 | 0.001 mW | Excellent (right next to AP) |
| -65 | ~0.3 nW | Voice/video minimum |
| -70 | ~0.1 nW | Good data signal |
| -80 | 0.01 nW | Weak — borderline |
| -90 | 0.001 nW | Very weak — barely usable |
| -95 | ~0.3 pW | Typical indoor noise floor |

## Cell Sizing Principles
- **Too high transmit power** creates a "loud AP, deaf client" problem — AP hears itself, clients hear each other and create co-channel interference
- Reduce TX power to match client power (~15–17 dBm typical)
- Aim for cells that overlap ~15–20% for seamless roaming
- Use 5 GHz for capacity, 2.4 GHz for range/legacy coverage

## Fresnel Zone
The first Fresnel zone must be ≥ 60% clear for optimal link quality. Obstacles inside the Fresnel zone degrade signal even if Line of Sight (LoS) exists.

For indoor deployments: mount APs in ceiling, above head height, away from HVAC ducts and metal beams.
