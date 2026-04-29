# XIQ Diagnostics

## Network 360 Monitor
**Path:** ML Insights → Network 360 Monitor

The bird's-eye view of your entire deployment.

| Visual Element | What It Means |
|----------------|--------------|
| Green switch-to-switch line | IS-IS/Fabric adjacency healthy |
| Red/Orange line | Fabric or physical link down |
| Yellow warning icon on device | Configuration issue or degraded health |
| AP branching from Port 3 | AP connected + FA negotiated |
| Health Score > 90 | Traffic prioritisation working, Fabric paths optimal |

**Device 360 (right-click any device):**
- Fabric Attach tab → shows I-SIDs granted to each port (visual `show fa assignment`)
- Interface view → hover a port for real-time bandwidth, PoE draw
- Events tab → device-specific log

## Client 360
**Path:** Manage → Clients → search by MAC address → open Client 360

The "time-travel" trace for a single client from radio to internet.

**Topology Strip (top of screen):**
```
[iPhone] → [AP3000 Port 3] → [NNI Port 10 — MAC-in-MAC tunnel] → [SW1 Port 1] → [Modem]
```

| Hop | What to Check | Green = |
|-----|--------------|---------|
| Wireless (iPhone → AP) | RSSI, SNR | RSSI ≥ -70 dBm, SNR ≥ 20 dB |
| Fabric Entry (AP → Switch) | FA assignment state | I-SIDs Active |
| Fabric Transit (NNI) | IS-IS adjacency | State: UP |
| Internet Exit (SW1 → modem) | Default route | 0.0.0.0/0 present |

**SLA Tab:**
- Latency should stay below 50ms even during failover — confirms 802.1p CoS tags honoured
- VoIP/SIP/RTP traffic type visible if QoS is working correctly
- Packet loss column: > 0.1% warrants investigation

## Event Logs
**Path:** Manage → Events

**Useful Filters:**
| Filter | When to Use |
|--------|-------------|
| Device = SW1+SW2, Category = System, Severity = Critical | After NNI failover test |
| Device = AP3000, Category = System | AP reboot / policy push issues |
| Category = Fabric Attach | AP I-SID provisioning events |

**Key Event Sequences (Failover):**
1. `IS-IS Adjacency Down` — "Neighbor 0.00.02 on port 1/10 changed state to DOWN"
2. `L3 Gateway Transition` — Anycast takes over (near-instantaneous)
3. `FA Assignment Updated` — AP re-negotiates I-SIDs after recovery
4. `IS-IS Adjacency Up` — fabric recovered (should be within seconds)

## Remote Sniffer (Wireshark)
**Path:** Device 360 → Diagnostics → Remote Sniffer → select port

- Select **Port 10 (NNI)** to capture MAC-in-MAC (802.1ah) Fabric headers
- Open capture in Wireshark → filter: `spbm` or `ieee8021ah`
- Inside the Backbone header you'll see: B-SA (backbone source), B-DA (backbone dest), I-SID
- This proves the IS-IS control plane and SPB data plane are aligned

## Floor Plan + Heat Map
**Path:** ML Insights → Network 360 Monitor → [site] → Floor Plan

1. Upload floor plan image (JPG/PNG/PDF)
2. Scale using a known distance (e.g., 3m doorway)
3. Drag AP3000s to their physical positions
4. Set AP height and orientation
5. Toggle RSSI heat map — filter by SSID (VLAN 20 Corp vs VLAN 30 Guest)
6. Watch client roaming trail in real-time as user moves between APs

## XIQ Alerts Setup
**Path:** Configure → Alerts → Alert Policies → + New Policy

Recommended alerts for a VOSS Fabric deployment:
- **Fabric Link Down** — immediate notification
- **IS-IS Neighbor State Change** — fabric adjacency drop
- **Anycast Gateway Conflict** — misconfigured I-SID or nick-name collision
- **AP Offline** — XIQ loses contact with AP

**Webhook to Slack/Discord:**
Configure → Alerts → Webhook Settings → + → paste Webhook URL → Payload Format: JSON → map to alert policy
Result: JSON message in your Slack channel the moment an IS-IS adjacency drops.
