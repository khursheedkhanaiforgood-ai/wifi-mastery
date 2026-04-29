# PoE Requirements

## PoE Standards Comparison
| Standard | Max Power (PSE) | Delivered to Device | Use For |
|----------|----------------|--------------------|---------| 
| 802.3af (PoE) | 15.4W | 12.95W | IP cameras, small APs |
| 802.3at (PoE+) | 30W | 25.5W | AP3000, most enterprise APs |
| 802.3bt Type 3 (PoE++) | 60W | 51W | PTZ cameras, multi-radio APs |
| 802.3bt Type 4 (PoE++) | 100W | 71.3W | High-power APs, thin clients |

## AP3000 Power Requirements
- **Required: 802.3at (PoE+)** — 25.5W delivered
- 802.3af (PoE) is NOT sufficient at full capability
- If only PoE (802.3af) is available, AP3000 may run at reduced radio power or disable one radio
- Always use 802.3at ports when deploying AP3000

## Extreme 5320 PoE Budget
Check available PoE budget before deployment:
```
show poe port all          # per-port status and wattage
show poe detail            # total budget and consumption
```

5320-16P-2MXT-2X: 185W PoE budget (verify with `show poe detail`)
- 7 × AP3000 at 25.5W = 178.5W — very close to limit; leave headroom
- Recommended: plan for 20% headroom → max 6 AP3000 on a 185W switch

## Cable Requirements for PoE+
- Minimum: **Cat5e** (Cat6 preferred for PoE+)
- Max cable length: 100m (328ft) — PoE budget reduces at longer runs
- Use solid-core cable for permanent runs (not stranded patch cable)
- Poorly terminated or damaged cable → PoE negotiation may fall back to 15.4W

## PoE Budget Calculation
```
Total AP count × 25.5W = Required PoE budget
Add 20% headroom: Required ÷ 0.8 = Switch budget needed

Example: 6 AP3000s
6 × 25.5W = 153W required
153 ÷ 0.8 = 191W switch budget needed
→ 5320 at 185W is borderline; use max 6 APs with care
```

## Verify PoE on a Port
```
show poe port 3
# Look for:
# Oper Status: On
# Class: 4 (25.5W — PoE+)
# Actual Power: ~24W typical for AP3000
```

If Class shows 0 or 3 instead of 4 → cable issue or switch only supports 802.3af on that port.
