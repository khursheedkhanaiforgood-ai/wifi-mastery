# WiFi Digital Twin Platform — High Level Design

> Version 1.1 | April 2026 | Private — not for public distribution

---

## 1. What This Is

A **WiFi Digital Twin** — a virtual representation of an enterprise WiFi network that mirrors the physical deployment in real time, predicts the outcome of design decisions before hardware is touched, and continuously optimizes the live network by feeding EP1 telemetry back into the same simulation engines that designed it.

It is not a documentation tool. It is not a configuration generator. It is a live, intelligent model of your WiFi network that:

- **Predicts** — how many APs, what channels, what power, what throughput — before any AP is mounted
- **Configures** — generates XIQ Device Templates, EXOS CLI scripts, VOSS CLI scripts, ready to push
- **Mirrors** — receives EP1 telemetry from deployed APs and runs the same RRM simulation the Broadcom chipsets are running
- **Tunes** — identifies the gap between predicted and actual, generates config deltas, pushes corrections to XIQ
- **Teaches** — every step explained in 5G/cellular analogies for engineers crossing domains

---

## 2. Architecture Overview

Four conceptual layers, three operational loops.

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  LAYER 4: KNOWLEDGE                                                          │
│  Private RAG corpus (~/.wifi-rag/) + 802.11be-2024 standard + Web           │
│  Tier 1: 802.11be-2024, 802.11-2024, Rev Wi-Fi, WLANPros                    │
│  Tier 2: Aruba VHD, Apple enterprise, DFS papers                            │
│  Tier 3: WLANPi, Wireshark, iPerf3, Linux/Netsh CLI                         │
└────────────────────────────────┬─────────────────────────────────────────────┘
                                 │ validates every recommendation
┌────────────────────────────────▼─────────────────────────────────────────────┐
│  LAYER 3: INTELLIGENCE — AGENT ORCHESTRATION                                 │
│                                                                              │
│  ORCHESTRATOR ──────────────────────────────────────────────────────────┐   │
│       │                                                                  │   │
│  ┌────▼────────┐  ┌────────────────┐  ┌───────────────┐  ┌──────────┐  │   │
│  │ INTAKE      │  │ DESIGN         │  │ CONFIG        │  │ TROUBLE- │  │   │
│  │ AGENT       │  │ AGENT          │  │ AGENT         │  │ SHOOTING │  │   │
│  │             │  │ ├CapacityPlan  │  │ ├XIQ Template │  │ AGENT    │  │   │
│  │ Socratic    │  │ ├LinkBudget    │  │ ├EXOS CLI     │  │ ├30-Row  │  │   │
│  │ →SiteReqs   │  │ ├APSelection   │  │ ├VOSS CLI     │  │ ├Spectrum│  │   │
│  │ JSON        │  │ ├ChannelPlan   │  │ └Qualification│  │ ├Debug   │  │   │
│  └─────────────┘  │ └RAGValidation │  │  Checklist    │  │ └WLANPi  │  │   │
│                   └────────────────┘  └───────────────┘  └──────────┘  │   │
│                                                                          │   │
│  ┌───────────────────────────────────────────────────────────────────┐  │   │
│  │ TEACHING AGENT (cross-cuts all — 5G↔WiFi bridge mode primary)    │  │   │
│  └───────────────────────────────────────────────────────────────────┘  │   │
│  ┌───────────────────────────────────────────────────────────────────┐  │   │
│  │ RAG VALIDATION AGENT (corpus → 802.11be-2024 → web — standard    │  │   │
│  │ wins on conflict)                                                 │  │   │
│  └───────────────────────────────────────────────────────────────────┘  │   │
└────────────────────────────────┬─────────────────────────────────────────┘   │
                                 │                                              │
┌────────────────────────────────▼─────────────────────────────────────────────┐
│  LAYER 2: SIMULATION — DIGITAL TWIN ENGINES                                  │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  RRM SIMULATION AGENT  ← THE DYNAMIC LAYER                         │    │
│  │                                                                     │    │
│  │  DCS Simulator    TPC Simulator    BSS Color     Multi-AP Coord     │    │
│  │  (channel)        (TX power)       (802.11ax)    (WiFi 7 CoMP)      │    │
│  │                                                                     │    │
│  │  Pre-deploy:  runs on designed topology, predicts AutoRF output     │    │
│  │  Post-deploy: runs on EP1 telemetry, mirrors real Broadcom AutoRF   │    │
│  │  Output:      predicted channel map, power map, oscillation risk    │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
│  Link Budget Engine     Airtime Calculator      Channel Planner              │
│  SNR→MCS→data rate      Rev Wi-Fi + OFDMA       2.4/5/6 GHz + DFS           │
│  802.11b → 802.11be     airtime demand model    PSC + CCI scoring            │
│  MCS 0-13 (4K-QAM)      SSID overhead calc      regulatory domain           │
└────────────────────────────────┬─────────────────────────────────────────────┘
                                 │
┌────────────────────────────────▼─────────────────────────────────────────────┐
│  LAYER 1: INTEGRATION                                                        │
│                                                                              │
│  XIQ API              EP1 Telemetry API        EXOS/VOSS SSH (Netmiko)       │
│  Config push          Channel util, RSSI,       Switch config push           │
│  Device Templates     SNR, retry rates,                                      │
│  SSID profiles        roaming events,           TUNING AGENT                 │
│  AutoRF params        AP health, MCS stats      Gap analysis → config delta  │
│                       BSS load, airtime stats   → XIQ push                  │
│                            │                                                 │
│                    EP1 INTEGRATION AGENT                                     │
│                    Normalizes telemetry → feeds RRM Sim + Tuning Agent       │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. The Three Operational Loops

### Loop 1 — Design Loop (pre-deployment, one-time per site)

```
User (Architect / RF Engineer)
    │
    ▼
INTAKE AGENT — Socratic requirements gathering
    │   Site type, area, device mix, apps + SLAs, AP budget, SSID plan
    ▼
DESIGN AGENT
    ├── Capacity Planning Sub-agent
    │   Rev Wi-Fi airtime model: app_throughput / device_throughput_capability
    │   = AP radios required (capacity-driven count)
    │   OFDMA extension: 802.11ax RU scheduling (80% airtime efficiency gain)
    │
    ├── Link Budget Sub-agent
    │   RF Coverage Design RSSI (−67/−75/−80 dBm) − noise floor = SNR
    │   SNR → MCS (802.11b through 802.11be MCS 0-13)
    │   → device throughput capability → airtime per device
    │   = coverage-driven AP count
    │   FINAL AP COUNT = max(capacity-driven, coverage-driven)
    │
    ├── AP Selection Sub-agent
    │   Budget filter → AP3000 / AP4000 / AP5020 / AP5022 / AP5060
    │   Validates: PoE budget (802.3af/at/bt), spatial streams, band support
    │   Broadcom chipset capabilities mapped per model
    │
    ├── Channel Planning Sub-agent
    │   2.4 GHz: channels 1, 6, 11 only (20 MHz, no exceptions)
    │   5 GHz: U-NII 1/2a/2c/3, DFS flagged, channel width per CCI budget
    │   6 GHz: PSC channels primary, RNR on 2.4/5 SSIDs, WPA3 mandatory
    │   CCI score: predicts co-channel interference per channel assignment
    │
    └── RAG Validation Sub-agent
        Every design decision validated: corpus → 802.11be-2024 → web
    │
    ▼
CONFIG AGENT
    ├── XIQ Device Template JSON (dual: static profile + AutoRF profile)
    ├── EXOS CLI script (VLAN, PoE, trunk/access, DHCP)
    ├── VOSS CLI script (FA I-SID, VLAN↔SSID binding)
    └── Qualification Checklist (WLANPros Not-Wireless pre/post)
    │
    ▼
DEPLOY → XIQ API push + Netmiko SSH push
```

**Teaching Agent fires after each step.** For a 5G engineer: Intake = requirements gathering ≈ RNTI allocation planning; Capacity Plan = PRB utilization model; Link Budget = NR link budget; AP count = cell site count.

---

### Loop 2 — Optimization Loop (continuous, post-deployment)

```
Deployed APs + Switches
    │ (every 5-15 min telemetry cycle)
    ▼
Extreme Platform One (EP1)
    Channel utilization per radio
    RSSI per client
    SNR per client
    Retry rate per radio
    Roaming event log (802.11r/k/v)
    Client count per radio
    MCS distribution per radio
    BSS Load IE values
    AP health (uptime, reboot events)
    Broadcom diagnostics (BSS color conflicts, TWT sessions, OFDMA stats)
    │
    ▼
EP1 INTEGRATION AGENT
    Normalizes to DesignSession metric format
    Time-series store (InfluxDB / TimescaleDB)
    │
    ├──────────────────────────────┐
    ▼                              ▼
RRM SIMULATION AGENT          TUNING AGENT
    │                              │
    ├── DCS Simulator              ├── Predicted vs Actual comparison
    │   Real channel util →        │   Channel: designed vs AutoRF actual
    │   Broadcom channel algo →    │   Power: designed vs TPC actual
    │   Predicted next channel     │   Utilization: designed vs live
    │                              │
    ├── TPC Simulator              ├── Broadcom parameter identification
    │   Neighbor RSSI →            │   Which chipset knob to turn?
    │   Power step-down algo →     │   MBR? RX-SOP? Assoc limit?
    │   Predicted TX power         │   Channel width? CCI threshold?
    │                              │
    ├── BSS Color Simulator        └── Config delta → XIQ API push
    │   Conflict detection →           + Teaching Agent commentary
    │   Re-assignment →                ("Your channel oscillation is
    │   Interference reduction         equivalent to MRO instability
    │                                   in 5G SON — here's the fix")
    └── Multi-AP Coord Sim
        (AP5020/5060 only)
        CoMP-equivalent prediction
    │
    ▼
Gap Analysis output:
    ├── GREEN: AutoRF converged to design intent — no action
    ├── AMBER: Minor drift — monitoring recommendation
    └── RED: Significant divergence — config delta generated
```

**This is the core of the digital twin.** The RRM Simulation Agent runs the same algorithm Broadcom AutoRF runs — so the digital twin PREDICTS what the real APs will do before they do it, and detects problems before they impact users.

---

### Loop 3 — Troubleshooting Loop (event-driven)

```
EP1 Alert  OR  User symptom description
    │
    ▼
TROUBLESHOOTING AGENT
    │
    ├── L1→L7 Diagnosis Sub-agent
    │   30-row WLANPros potential causes table
    │   WIRELESS (rows 1-15): End user → RF media → RRM → DFS → Auth → AP config
    │   LOCAL NETWORK (16-28): Wired → switch → DHCP → DNS → RADIUS → Firewall
    │   INTERNET (29-30): Bandwidth throttling, captive portal
    │   Scoring: EP1 symptom data → ranked probable causes
    │
    ├── Spectrum Analysis Sub-agent
    │   13 RF interferer signatures (Ekahau corpus)
    │   Matches EP1 channel utilization patterns to known interferer types
    │
    ├── Debug Commands Sub-agent
    │   EXOS: `show wireless`, `show log`, `show port utilization`
    │   VOSS: FA diagnostics, I-SID binding verification
    │   XIQ: API health checks, device template audit
    │
    └── WLANPi Sub-agent
        Passive survey, active survey, RADIUS testing, packet capture
    │
    ▼
Fix recommendation → Config delta (if applicable) → XIQ push
```

---

## 4. RRM Dynamic Layer — Technical Detail

### Why RRM is the hardest layer to get right

In 5G, SON (ANR, MLB, MRO, CCO) runs with centralized visibility and a predictive model. Every NB's measurements flow to the SON engine; changes are coordinated network-wide.

In WiFi, every AP runs RRM independently. Broadcom AutoRF uses local measurements only (background scanning, channel utilization). There is no central coordinator. This creates:

- **Oscillation risk**: Two APs competing for the same channel, each detecting the other as interference, each moving to a new channel, until they end up on the same channel again
- **Coverage holes**: TPC aggressively reducing power leaves clients at cell edge with poor SNR
- **Channel width instability**: Dynamic channel width (❌ NOT RECOMMENDED per WLANPros) causes roaming algorithms to fail because clients see the AP's channel width changing

### How the Digital Twin handles this

```
Design Phase:
  Static design → seed channel/power → RRM Sim runs forward simulation
  Output: "At 24h, AutoRF will converge to channels X,Y,Z with 
           power levels A,B,C — 2 APs show oscillation risk"
  Action: Adjust CCI threshold or lock oscillating APs to static channel

Post-Deploy (live mirror):
  EP1 telemetry → RRM Sim updates every 15 min
  Compares: designed state vs AutoRF actual state
  Detects: unexpected channel changes, power drift, CCI increases
  Flags: oscillation events, coverage holes forming, high retry rates
```

### Broadcom AutoRF trigger thresholds (XIQ-configurable)
| Parameter | Default | WLANPros Recommendation |
|-----------|---------|------------------------|
| CCI threshold | -85 dBm | >-85 dBm (flag CCI above this) |
| Retry rate trigger | >20% | Site-specific |
| Channel utilization trigger | >70% | <40% for 5 GHz target |
| Scan interval | Background | AP Background Scanning: ON |
| Channel lock (override) | None | Use for known-good channels |

---

## 5. Extreme Platform One (EP1) Integration

EP1 is the observability and AI operations layer of Extreme Networks' cloud platform. For the Digital Twin, EP1 serves as the **measurement plane** — the source of ground truth about what the deployed network is actually doing.

### EP1 data flows into the Digital Twin

| EP1 Metric | Digital Twin Use |
|------------|-----------------|
| Channel utilization per radio | RRM Sim: available airtime input |
| RSSI per client | Link Budget validation: actual vs predicted SNR |
| Retry rate per radio | RRM Sim: DCS trigger condition |
| Roaming event log | Troubleshooting: 802.11r/k/v working as designed? |
| Client MCS distribution | Link Budget validation: actual MCS vs predicted |
| BSS Load IE | Capacity validation: actual load vs designed load |
| AP health / reboot events | Troubleshooting: firmware, PoE, hardware issues |
| Broadcom BSS color conflicts | RRM Sim: color assignment validation |
| OFDMA RU utilization | Capacity: OFDMA effectiveness measurement |
| TWT session stats | Teaching: eDRX equivalent working? |

### EP1 → Digital Twin → XIQ feedback loop

```
EP1 reports: AP on channel 36 has 85% channel utilization
    ↓
EP1 Integration Agent: normalizes, stores time-series
    ↓
RRM Simulation Agent: runs DCS algorithm with actual utilization
    → Predicts: AutoRF will move AP to channel 100 within next 15 min
    ↓
Tuning Agent: designed target was <40% utilization on channel 36
    → Root cause: 6 SSIDs broadcasting (overhead 22% per SSID Calculator)
    → Fix: reduce to 4 SSIDs, move some traffic to 6 GHz
    ↓
Config delta: XIQ template update — disable 2 SSIDs on this radio
    + Teaching: "This is equivalent to reducing your PRB overhead 
      from pilot + PDCCH consuming 30% of the frame — 
      fewer SSIDs = more airtime for data, same principle"
    ↓
XIQ API push → AP reloads config → EP1 confirms utilization drops
```

---

## 6. AP Model → Broadcom Chipset Mapping

| AP Model | WiFi | Chipset | Radios | SS | PoE | SDR |
|----------|------|---------|--------|----|-----|-----|
| AP3000 | WiFi 6 | BCM6750-series | 2 (2.4+5) | 2×2:2 | 802.3at (25.5W) | No |
| AP4000 | WiFi 6E | BCM6756-series | 3 (2.4+5+6) | 3×2×2:2 | 802.3bt (51W) | No |
| AP5020 | WiFi 7 | BCM6726-series | 3 (2.4+5+6) | 3×4×4:4 | 802.3at or 802.3bt | Yes (dual-5 or dual-6) |
| AP5022 | WiFi 7 | BCM6726-series | 4 (2.4+5+6+security) | 4×4:4 | 802.3bt (51W) | Yes |
| AP5060/U/D | WiFi 7 | BCM6726-series | 3 outdoor/venue | 3×4×4:4 | 802.3bt (51W) | Yes |

**SDR (Software-Defined Radio) on AP5020/5022/5060:** The 5 GHz radio can be re-configured as a second 6 GHz radio via XIQ software setting. This doubles 6 GHz capacity in dense deployments. The Digital Twin AP Selection Sub-agent flags this option when 6 GHz capacity demand exceeds single-radio capacity.

---

## 7. Multi-User Collaborative Model

| Role | Capability | View |
|------|-----------|------|
| Architect | Full design + config authority | All layers |
| RF Engineer | Design + channel tuning | Design + RRM simulation |
| Network Engineer | Config + switch CLI | Config + switch layer |
| Learner | Read + query Teaching Agent | Teaching sidebar always active |
| Ops | Troubleshooting + tuning | Loop 2 + Loop 3 |

Real-time WebSocket session — all roles see live changes. Every design decision recorded with rationale (for RRM tuning history).

---

## 8. WiFi Generation Strategy

### Version 1 Scope — Uniform Deployment

User selects AP generation at DPM time: **WiFi 6 (AP3000/AP4000)** or **WiFi 7 (AP5020/AP5022/AP5060)** for the entire site. The tool designs uniformly for that generation. The Comparison Engine runs both strategies on the same DPM inputs and shows the delta.

### Two Branches

| Branch | AP Fleet | Purpose |
|--------|----------|---------|
| `feature/wifi6-baseline` | AP3000 / AP4000 | Current state, MCS 0-11, no WiFi 7 features |
| `feature/wifi7-complete` | AP5020 / AP5022 / AP5060 | All WiFi 7 gaps filled, MCS 0-13, MLO, 320 MHz |

### Strategy Pattern

```python
class WiFi6LinkBudget(LinkBudgetStrategy):
    def max_mcs_index(self) -> int: return 11          # 1024-QAM 5/6

class WiFi7LinkBudget(WiFi6LinkBudget):               # inherits WiFi 6
    def snr_to_mcs(...):
        result = super().snr_to_mcs(...)               # WiFi 6 baseline
        if snr_db >= 36 and self.be_capable:
            result = self._apply_mcs_12_13(snr_db)    # upgrade 4K-QAM
        return result
    def max_mcs_index(self) -> int: return 13          # 4K-QAM 5/6

class ComparisonEngine:
    def run(self, dpm) -> ComparisonReport:
        wifi6 = DesignEngine(strategy=WiFi6Strategy()).run(dpm)
        wifi7 = DesignEngine(strategy=WiFi7Strategy()).run(dpm)
        return ComparisonReport(wifi6=wifi6, wifi7=wifi7, delta=self._diff(wifi6, wifi7))
```

### Feature Matrix

| Feature | WiFi 6 (AP3000/AP4000) | WiFi 7 (AP5020/AP5060) |
|---------|----------------------|----------------------|
| MCS 0-11 (1024-QAM) | ✅ | ✅ |
| MCS 12-13 (4K-QAM) | ❌ | ✅ SNR ≥36 dB |
| Max channel width | 160 MHz | 320 MHz (6 GHz) |
| MLO | ❌ | ✅ EMLSR / STR |
| Multi-AP Coordination | ❌ | ✅ AP5020/5022/5060 |
| Preamble Puncturing | ❌ | ✅ 802.11be |
| HARQ | ❌ | ✅ chipset-managed |
| OFDMA / BSS Color / TWT | ✅ | ✅ |
| SDR (dual-5/dual-6) | ❌ | ✅ AP5020/5022/5060 |

### Mixed-Mode — Deferred to Future Sprint

**Why deferred:** Mixing WiFi 6 and WiFi 7 APs on the same site requires knowing which AP is physically adjacent to which — MLO drops to single-link on roam to a WiFi 6 AP, 320→160 MHz ACI at zone boundaries, Multi-AP Coord cluster limited to WiFi 7 APs only. Without floor plan topology, the adjacency graph cannot be computed and these boundary conditions cannot be resolved.

**What mixed-mode will need (future `feature/mixed-topology`):**
- Floor plan upload (SVG / DXF / image)
- AP placement tool → adjacency graph
- Per-zone AP model assignment
- Zone-boundary channel width reconciliation
- MixedAPStrategy = `zone_map[zone] → strategy`
- Comparison Engine mode 3 (mixed vs pure WiFi 6 vs pure WiFi 7)

---

## 9. Sprint Roadmap

| Sprint | Duration | Deliverable |
|--------|----------|-------------|
| Sprint 1 | 3 days | RAG + Engines (WiFi 6 + WiFi 7 strategies) + Core Agents + Config + FastAPI |
| Sprint 2 | 1 week | RRM Simulation + EP1 Integration + Tuning Agent + XIQ Push + Frontend |
| Sprint 3 | 1 week | Troubleshooting Agent + Multi-user + 802.11be full index + EVE-NG |
| Sprint 4+ | TBD | Mixed-mode (feature/mixed-topology) — floor plan + AP placement + adjacency graph |

---

## 10. What Makes This a True Digital Twin

| Digital Twin Criterion | This Platform |
|----------------------|---------------|
| Virtual model of physical system | ✅ Agent-simulated WiFi network: capacity, link budget, RRM |
| Real-time data from physical system | ✅ EP1 telemetry ingested every 15 min |
| Bidirectional synchronization | ✅ EP1 → Twin (observe) + Twin → XIQ (control) |
| Predictive simulation | ✅ RRM Sim predicts AutoRF convergence before it happens |
| Optimization feedback loop | ✅ Gap analysis → config delta → push → validate |
| Mirroring of physical behavior | ✅ Broadcom AutoRF algorithm replicated in DCS/TPC simulators |

The physical network and the digital twin run the **same RRM algorithm** — one in Broadcom silicon, one in Python. When they diverge, the Tuning Agent finds out why and corrects it.
