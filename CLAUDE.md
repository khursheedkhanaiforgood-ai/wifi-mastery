# WiFi Mastery — Claude Code Instructions

> Related: https://github.com/khursheedkhanaiforgood-ai/5320-onboarding · https://github.com/khursheedkhanaiforgood-ai/voss-fabric-migration

## Project Architecture

| Folder | Contents |
|--------|----------|
| `01-design/` | RF, SSID, channels, security, QoS, VLAN mapping |
| `02-configuration/` | XIQ Network Policy, SSID profiles, guest isolation, Fabric Attach |
| `03-deployment/` | AP placement, PoE, XIQ onboarding, checklists |
| `04-troubleshooting/` | Connectivity, roaming, interference, debug CLI |
| `05-reference/` | 802.11 standards, AP3000 notes, glossary |
| `.claude/agents/` | Specialist agents per domain |
| `.claude/commands/` | Repeatable workflows as slash commands |

## Hardware Context

- **AP:** Extreme Networks AP3000 (802.11ax / WiFi 6, dual-radio, PoE+)
- **Switch:** Extreme 5320-16P-2MXT-2X — EXOS or VOSS persona
- **Cloud:** ExtremeCloud IQ (XIQ) — hac.extremecloudiq.com
- **Fabric:** VOSS/Fabric Engine — IS-IS + SPBM + Fabric Attach

## Domain Agents

| Agent | When to invoke |
|-------|---------------|
| `wifi-designer` | RF planning, SSID architecture, channels, security, QoS, VLAN mapping |
| `xiq-configurator` | XIQ Network Policy walkthrough, SSID/user profile config, guest isolation |
| `troubleshooter` | Client connectivity, roaming, interference, debug command sequences |

## Slash Commands

| Command | Purpose |
|---------|---------|
| `/design-review` | Audit a WiFi design against best practices — returns PASS/FAIL/WARN |
| `/deploy-checklist` | Pre- and post-deployment checklist for a site |

## Key Conventions (apply to all work in this repo)

- Max **3–4 SSIDs per AP** (beacon overhead degrades all clients)
- SSID → VLAN mapping is **1:1** (simplest, cleanest)
- VOSS I-SID convention: **VLAN ID + 100,000** (e.g., VLAN 20 → I-SID 100020)
- Guest always uses **OWE + Transition Mode** (legacy device compat — critical)
- QoS: Corp → `qp6` / CoS 5–6 / DSCP EF; Guest → `qp1` / CoS 0
- Guest isolation: **XIQ firewall rules** (deny subnet) + **L2 station isolation** (drop between stations)
- **VOSS critical:** ACL must be on BOTH switches — Anycast Gateway means SW2 routes locally, bypassing SW1 ACL

## Lab Reference (Horizon Custom Fabrication — Apr 2026)

| SSID | Security | VLAN | I-SID | QoS Weight |
|------|----------|------|-------|------------|
| Corporate_Wireless | WPA2/WPA3-PSK | 20 | 100020 | 100 |
| Guest_Wireless | OWE + Transition | 30 | 100030 | 10 |

- XIQ Policy: **AP_April21_KarlLab**
- SW1: L3 Core — all SVIs, DHCP, default route to modem
- SW2: Pure L2 (EXOS) or Anycast Gateway peer (VOSS)
- ⚠ Corp SSID PSK Key Value field — must be set in XIQ (was empty Apr 21)
- ⚠ OWE Transition Mode toggle — Guest SSID → push policy after enabling

## Quality Gates

- `/design-review` before finalising any SSID/VLAN design
- `/deploy-checklist` before any physical deployment
- Verify `show fa assignment` after AP boots — confirm I-SIDs Active
- Verify `show ip route` on SW2 shows `0.0.0.0/0 → 0.00.01` (VOSS IP Shortcuts)
- Run failover test: pull NNI cable during `ping -t` → expect 0–1 dropped pings
