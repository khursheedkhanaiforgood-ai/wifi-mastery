# WiFi Mastery — Design, Configuration, Deployment & Troubleshooting

A structured knowledge base for enterprise WiFi covering design principles, Extreme Networks EXOS/VOSS configuration, XIQ cloud management, and end-to-end troubleshooting. Built from real lab deployments (Horizon lab, Meridian Retail) and months of hands-on session material.

Includes WiFi 6, WiFi 6E, and WiFi 7 (802.11be) coverage throughout.

## Repository Structure

```
wifi-mastery/
├── 01-design/          RF fundamentals, SSID architecture, channel planning, security, QoS, VLAN/I-SID mapping
├── 02-configuration/   XIQ network policy, SSID profiles, user profiles, guest isolation, Fabric Attach
├── 03-deployment/      AP placement, PoE requirements, XIQ onboarding, deployment checklist
├── 04-troubleshooting/ Client connectivity, roaming issues, RF interference, debug commands, XIQ diagnostics
├── 05-reference/       802.11 standards, AP3000 notes, glossary
└── .claude/            Claude Code agents, commands, and settings for this knowledge base
```

## Quick Links

### Design
- [RF Fundamentals](01-design/rf-fundamentals.md) — RSSI/SNR targets, bands, path loss, cell sizing
- [SSID Architecture](01-design/ssid-architecture.md) — Max SSIDs, naming, OWE Transition Mode
- [Channel Planning](01-design/channel-planning.md) — 2.4/5/6 GHz, DFS, WiFi 7 MLO
- [Security Design](01-design/security-design.md) — WPA2/WPA3/OWE/802.1X decision matrix
- [QoS Design](01-design/qos-design.md) — WMM ACs, DSCP mapping, EXOS/VOSS CLI
- [VLAN & I-SID Mapping](01-design/vlan-isid-mapping.md) — EXOS vs VOSS, I-SID convention

### Configuration
- [XIQ Network Policy](02-configuration/xiq-network-policy.md) — Step-by-step policy creation, common pitfalls
- [SSID Profiles](02-configuration/ssid-profiles.md) — Band steering, 802.11r, min rates, WiFi 6/7
- [User Profiles & QoS](02-configuration/user-profiles-qos.md) — VLAN assignment, firewall, DSCP trust
- [Guest Isolation](02-configuration/guest-isolation.md) — XIQ firewall + L2 station isolation + VOSS ACL
- [Fabric Attach](02-configuration/fabric-attach.md) — FA negotiation, CLI, verify/debug

### Deployment
- [AP Placement](03-deployment/ap-placement.md) — Coverage targets, height, obstacles, spacing
- [PoE Requirements](03-deployment/poe-requirements.md) — 802.3af/at/bt, AP3000 = 25.5W = PoE+
- [XIQ Onboarding](03-deployment/xiq-onboarding.md) — AP + switch onboarding, FA auto-provisioning
- [Deployment Checklist](03-deployment/deployment-checklist.md) — Full 8-phase pre/post verification

### Troubleshooting
- [Client Connectivity](04-troubleshooting/client-connectivity.md) — 7-step L1→L7 decision tree
- [Roaming Issues](04-troubleshooting/roaming-issues.md) — Sticky client, 802.11r, 6 GHz roaming
- [RF Interference](04-troubleshooting/rf-interference.md) — CCI, DFS, noise floor, BSS Coloring
- [Debug Commands](04-troubleshooting/debug-commands.md) — EXOS + VOSS command reference
- [XIQ Diagnostics](04-troubleshooting/xiq-diagnostics.md) — Network 360, Client 360, Remote Sniffer

### Reference
- [802.11 Standards](05-reference/80211-standards.md) — WiFi 1–7 timeline, WiFi 6 + 7 features
- [AP3000 Notes](05-reference/ap3000-notes.md) — Specs, FA, placement, upgrade path
- [Glossary](05-reference/glossary.md) — 50+ terms: WiFi, VOSS/Fabric, XIQ, EXOS

---

## Key Conventions

| Convention | Rule |
|-----------|------|
| Max SSIDs | ≤ 4 per radio (Corp + Guest + IoT + optional) |
| SSID naming | `Site-Purpose` — e.g., `Horizon-Corp`, `Horizon-Guest` |
| I-SID calculation | VLAN ID + 100,000 (VLAN 20 → I-SID 100020) |
| Guest SSID | OWE + Transition Mode (legacy fallback to hidden Open SSID) |
| Corp SSID | WPA2/WPA3 Transition Mode (broad device compatibility) |
| VOSS ACL | Apply to BOTH switches — Anycast means SW2 routes locally |
| Karl Rule (EXOS) | `enable ipforwarding vlan Default` — without it, no internet |
| DHCP reach-back | Include port 10 in DHCP config — required for SW2 clients |
| PoE for AP3000 | Must be 802.3at (PoE+) — 25.5W draw, 802.3af insufficient |

---

## Horizon Lab Reference

| Device | Model | Role | Ports |
|--------|-------|------|-------|
| SW1 | Extreme 5320 (EXOS) | L3 core + DHCP + routing | Port 1→modem, Port 10→NNI, Port 3+5→UNI |
| SW2 | Extreme 5320 (VOSS) | Fabric peer + Anycast GW | Port 10→NNI, Port 3+5→UNI |
| AP | Extreme AP3000 | WiFi 6, 2×2 MIMO | Connected to SW2 Port 3 via PoE+ |
| Modem | ISP Gateway | Internet exit | 192.168.0.1, connected to SW1 Port 1 |

---

## Claude Code Agents (`.claude/agents/`)

| Agent | Invoke When |
|-------|------------|
| `wifi-designer` | RF design, SSID architecture, channel planning questions |
| `xiq-configurator` | XIQ policy creation, Fabric Attach, push sequence |
| `troubleshooter` | Client connectivity issues, VOSS/EXOS debug |

Slash commands: `/design-review` · `/deploy-checklist`

---

## License

MIT — use freely for study, lab work, and enterprise deployments.
