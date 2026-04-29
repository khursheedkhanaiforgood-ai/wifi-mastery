# AP Placement Guidelines

## Coverage Targets
| Use Case | Minimum RSSI at Edge | Preferred RSSI |
|----------|---------------------|----------------|
| Voice / video calls | -67 dBm | -65 dBm |
| Data / laptops | -72 dBm | -68 dBm |
| IoT / sensors | -78 dBm | -73 dBm |

## Mounting Height
- **Optimal: 8–10 ft (2.4–3m) AFF** (Above Finished Floor)
- Too low (<6 ft): AP is in the same plane as clients — poor pattern, blocked by furniture
- Too high (>12 ft): signal hits floor at steep angle, coverage holes between APs

## Ceiling vs Wall Mount
| Mount Type | When to Use | Notes |
|-----------|-------------|-------|
| Ceiling (preferred) | Standard office, open areas | Omnidirectional pattern works well |
| Wall | Corridors, areas with low ceilings | Point AP slightly down the corridor |
| Above ceiling tile | Drop-ceiling offices | Avoid — degrades signal significantly |

## What to Avoid
- **Metal shelving/racks:** 20–40 dB attenuation — place APs to avoid metal in the signal path
- **HVAC ducts:** Metal + airflow disrupts signal
- **Elevator shafts:** Effective RF barrier
- **Microwaves:** 2.4 GHz interference — keep APs away from kitchen areas
- **Thick concrete walls:** 15–40 dB — treat as a separate coverage zone

## High-Density Deployments (Conference Rooms, Classrooms)
- **One AP per room** for conference rooms with 20+ users
- Use 5 GHz / 6 GHz primarily — more channels, less interference
- **Reduce TX power:** 12–15 dBm to tighten cells and reduce co-channel interference
- **Band steer aggressively:** push all capable clients to 5/6 GHz
- Disable 2.4 GHz radio if all clients support 5 GHz

## Spacing Rules of Thumb
| Environment | 5 GHz AP Spacing |
|-------------|-----------------|
| Dense office (cubicles) | 20–25 ft (6–8m) |
| Standard office | 25–35 ft (8–11m) |
| Open warehouse/retail | 50–75 ft (15–23m) |
| Outdoor (open) | 100–200 ft (30–60m) |

## AP3000 Placement Notes
- AP3000 has omni antennas — best deployed on ceiling, pointing down
- PoE+ required (802.3at) — ensure switch port supports PoE+ before mounting
- Cable must pass 802.3at PoE test (not all Cat5e in older buildings supports full 25.5W)
- Mount near center of coverage area, not at the edge

## Site Survey Before Deployment
1. Walk the space with a WiFi analyzer (Ekahau, NetSpot, WiFi Analyzer)
2. Mark coverage dead zones
3. Note obstacles: glass partitions, server racks, kitchen areas
4. Count expected concurrent users per zone
5. Verify PoE drops reach all planned AP locations
