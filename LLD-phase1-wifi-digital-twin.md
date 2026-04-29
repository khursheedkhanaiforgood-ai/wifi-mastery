# WiFi Digital Twin Platform — Low Level Design: Phase 1

> Version 1.0 | April 2026 | Private — not for public distribution
> Scope: Uniform WiFi 6 OR WiFi 7 deployment (single AP generation per site, no floor plan)

---

## Phase 1 Constraints

| Constraint | Value | Reason |
|-----------|-------|--------|
| AP generation | WiFi 6 OR WiFi 7 (user picks) | Mixed-mode deferred — needs topology |
| Site topology | None — user estimates RF area | No floor plan in Phase 1 |
| AP placement | Uniform distribution assumed | No coordinates until Phase 4 |
| Output | Design + config only | No live EP1 feed in Sprint 1 |
| Comparison | WiFi 6 vs WiFi 7 on same DPM | ComparisonEngine runs both strategies |

---

## L1 — Data Models (Pydantic v2)

### Enumerations

```python
class WiFiGen(str, Enum):
    WIFI6    = "wifi6"
    WIFI6E   = "wifi6e"
    WIFI7    = "wifi7"

class WiFiProtocol(str, Enum):
    N    = "802.11n"
    AC   = "802.11ac"
    AX   = "802.11ax"
    BE   = "802.11be"

class Band(str, Enum):
    GHZ24 = "2.4"
    GHZ5  = "5"
    GHZ6  = "6"

class ChannelWidth(int, Enum):
    W20  = 20
    W40  = 40
    W80  = 80
    W160 = 160
    W320 = 320   # WiFi 7 / 6 GHz only

class SiteType(str, Enum):
    OFFICE       = "office"
    WAREHOUSE    = "warehouse"
    HOSPITAL     = "hospital"
    EDUCATION    = "education"
    RETAIL       = "retail"
    OUTDOOR      = "outdoor"
    VENUE        = "venue"

class WallType(str, Enum):
    OPEN     = "open"       # open plan, no walls
    LIGHT    = "light"      # drywall, glass
    HEAVY    = "heavy"      # brick, concrete block
    CONCRETE = "concrete"   # reinforced concrete

class RFEnvironment(str, Enum):
    OPEN_SPACE   = "open_space"    # noise floor ~−95 dBm, airtime 90%
    OPEN_OFFICE  = "open_office"   # noise floor ~−90 dBm, airtime 80%
    OFFICE       = "office"        # noise floor ~−87 dBm, airtime 70%
    DENSE_OFFICE = "dense_office"  # noise floor ~−85 dBm, airtime 60%
    INDUSTRIAL   = "industrial"    # noise floor ~−80 dBm, airtime 50%

class RFCoverageTier(str, Enum):
    CAPACITY = "capacity"  # −67 dBm — highest SNR, best MCS
    DATA     = "data"      # −75 dBm — good data rates
    BASIC    = "basic"     # −80 dBm — minimum connectivity

class PoEClass(str, Enum):
    AF  = "802.3af"   # 15.4W
    AT  = "802.3at"   # 30W
    BT  = "802.3bt"   # 60W / 90W

class ConcurrentMode(str, Enum):
    BOTH  = "both"   # apply both assoc% and active% cuts
    ASSOC = "assoc"  # apply assoc% only
    ACTIVE = "active"# apply active% only
    NONE  = "none"   # use raw device count

class RRMMode(str, Enum):
    XIQ_CLOUD = "xiq_cloud"   # holistic, preferred
    PER_AP    = "per_ap"      # Broadcom AutoRF local
    STATIC    = "static"      # manual channel/power
```

### Section A — Site Requirements

```python
class SiteRequirements(BaseModel):
    site_type:            SiteType
    coverage_area_m2:     float = Field(gt=0)
    num_floors:           int   = Field(ge=1)
    ceiling_height_m:     float = Field(default=3.0, ge=2.0, le=15.0)
    wall_type:            WallType
    rf_environment:       RFEnvironment
    regulatory_domain:    str   = Field(min_length=2, max_length=3)  # "US","EU","AU"
    # Derived — set by engine, overridable by user
    noise_floor_dbm:      Optional[float] = None   # derived from rf_environment
    available_airtime_pct:Optional[float] = None   # derived from rf_environment
    unii_bands_available: Optional[list[str]] = None  # derived from regulatory_domain

    @model_validator(mode='after')
    def derive_noise_and_airtime(self):
        lookup = {
            RFEnvironment.OPEN_SPACE:   (-95, 90.0),
            RFEnvironment.OPEN_OFFICE:  (-90, 80.0),
            RFEnvironment.OFFICE:       (-87, 70.0),
            RFEnvironment.DENSE_OFFICE: (-85, 60.0),
            RFEnvironment.INDUSTRIAL:   (-80, 50.0),
        }
        if self.noise_floor_dbm is None:
            self.noise_floor_dbm, _ = lookup[self.rf_environment]
        if self.available_airtime_pct is None:
            _, self.available_airtime_pct = lookup[self.rf_environment]
        return self
```

### Section B — Client Devices

```python
class ClientDevice(BaseModel):
    device_type:          str
    protocol:             WiFiProtocol
    tx_chains:            int = Field(ge=1, le=8)
    rx_chains:            int = Field(ge=1, le=8)
    antenna_gain_24ghz:   float = 2.0   # dBi
    antenna_gain_5ghz:    float = 3.0   # dBi
    antenna_gain_6ghz:    float = 3.0   # dBi (WiFi 6E/7 only)
    max_channel_width:    ChannelWidth
    band_support:         list[Band]
    quantity:             int   = Field(ge=1)
    concurrent_assoc_pct: float = Field(default=0.75, ge=0.0, le=1.0)
    concurrent_active_pct:float = Field(default=0.50, ge=0.0, le=1.0)
    concurrent_limit_mode:ConcurrentMode = ConcurrentMode.BOTH

    @model_validator(mode='after')
    def active_le_assoc(self):
        if self.concurrent_active_pct > self.concurrent_assoc_pct:
            raise ValueError("concurrent_active_pct must be ≤ concurrent_assoc_pct")
        return self

    @property
    def associated_count(self) -> int:
        return int(self.quantity * self.concurrent_assoc_pct)

    @property
    def active_count(self) -> int:
        return int(self.quantity * self.concurrent_active_pct)
```

### Section C — Applications

```python
class Application(BaseModel):
    app_type:         str
    throughput_kbps:  float = Field(gt=0)
    transport:        str = "TCP"        # TCP adds ~10% overhead
    latency_sensitive:bool = False       # True = no aggregation = higher overhead
    avg_frame_bytes:  int  = 1400        # VoIP: 150-220 bytes; data: 1400 bytes
    background:       bool = False       # background apps add airtime, not device count

    @property
    def tcp_overhead_factor(self) -> float:
        return 1.10 if self.transport == "TCP" else 1.0

    @property
    def aggregation_factor(self) -> float:
        return 1.5 if self.latency_sensitive else 1.0  # no AMPDU = 50% more overhead
```

### Section D — Network Design

```python
class NetworkDesignParams(BaseModel):
    ap_model:              str                      # AP3000/AP4000/AP5020/AP5022/AP5060
    wifi_generation:       WiFiGen                  # strategy selector
    channel_width_5ghz:    ChannelWidth = ChannelWidth.W80
    channel_width_6ghz:    ChannelWidth = ChannelWidth.W80
    client_band_dist:      dict[Band, float]        # {"2.4": 0.1, "5": 0.6, "6": 0.3}
    assoc_limit_per_radio: int  = 128
    num_ssids:             int  = Field(ge=1, le=15)
    mbr_24ghz:             int  = 12   # Mbps min basic rate
    mbr_5ghz:              int  = 12
    mbr_6ghz:              int  = 12
    rf_coverage_tier:      RFCoverageTier = RFCoverageTier.DATA
    capacity_growth_pct:   float = 0.15
    dfs_enabled:           bool  = False

    @property
    def coverage_rssi_dbm(self) -> float:
        return {RFCoverageTier.CAPACITY: -67.0,
                RFCoverageTier.DATA: -75.0,
                RFCoverageTier.BASIC: -80.0}[self.rf_coverage_tier]
```

### Section G — RRM Parameters

```python
class RRMParams(BaseModel):
    rrm_mode:              RRMMode = RRMMode.XIQ_CLOUD
    cci_threshold_dbm:     float = -85.0
    retry_rate_trigger_pct:float = 20.0
    channel_util_trigger_pct:float = 70.0
    tx_power_min_24ghz:    int = 7    # dBm — WLANPros target
    tx_power_min_5ghz:     int = 13
    tx_power_min_6ghz:     int = 16
    tx_power_max_dbm:      int = 23
    dynamic_channel_width: bool = False  # FIXED: always False — WLANPros ❌
```

### Section H — WiFi 7 Parameters (WiFi7 strategy only)

```python
class WiFi7Params(BaseModel):
    mlo_mode:          str  = "EMLSR"  # EMLSR / EMLMR / STR / disabled
    channel_width_320: bool = False    # 320 MHz on 6 GHz
    puncturing:        bool = True     # preamble puncturing on 6 GHz
    multi_ap_coord:    bool = True     # only on AP5020/5022/5060
    harq_enabled:      bool = True     # chipset-managed, flag for design note
    mcs12_13_expect:   bool = True     # expect 4K-QAM at SNR ≥36 dB
    twt_enabled:       bool = True     # TWT for IoT/battery devices
    rnr_advertise:     bool = True     # Reduced Neighbor Report on 2.4/5 SSIDs
    mfp_required:      bool = True     # 802.11w MFP mandatory on 6 GHz
    wpa3_required:     bool = True     # WPA3 mandatory on 6 GHz
```

### Full DPM

```python
class DesignParameterManifest(BaseModel):
    session_id:   str = Field(default_factory=lambda: str(uuid4()))
    created_at:   datetime = Field(default_factory=datetime.utcnow)
    site:         SiteRequirements
    clients:      list[ClientDevice]
    applications: dict[str, list[Application]]  # device_type → apps list
    network:      NetworkDesignParams
    channel:      ChannelParams
    security:     SecurityParams
    rrm:          RRMParams
    wifi7:        Optional[WiFi7Params] = None  # populated when WiFiGen.WIFI7

    @model_validator(mode='after')
    def wifi7_params_required_for_be(self):
        if self.network.wifi_generation == WiFiGen.WIFI7 and self.wifi7 is None:
            self.wifi7 = WiFi7Params()  # apply defaults
        return self
```

### Result Models

```python
class MCSResult(BaseModel):
    mcs_index:        int
    protocol:         str
    modulation:       str       # "4096-QAM", "1024-QAM", "64-QAM" etc.
    coding_rate:      str       # "3/4", "5/6"
    snr_required_db:  float
    data_rate_mbps:   float     # per spatial stream × SS count
    spatial_streams:  int
    channel_width_mhz:int

class LinkBudgetResult(BaseModel):
    device_type:              str
    band:                     Band
    rssi_design_dbm:          float    # from coverage tier
    noise_floor_dbm:          float
    snr_db:                   float
    mcs:                      MCSResult
    effective_throughput_mbps:float    # after TCP/aggregation overhead
    airtime_pct_per_device:   float
    uplink_mcs:               MCSResult  # may differ (TX power asymmetry)

class CapacityPlan(BaseModel):
    ap_count_coverage:   int
    ap_count_capacity:   int
    ap_count_final:      int   # max(coverage, capacity) × (1 + growth_pct)
    radios_required:     dict[str, int]   # "5ghz" → count
    ssid_overhead_pct:   float
    available_airtime_pct:float
    channel_util_target_pct:float = 20.0

class ChannelPlan(BaseModel):
    channels_24ghz:  list[int]   # always [1, 6, 11]
    channels_5ghz:   list[int]   # U-NII selection
    channels_6ghz:   list[int]   # PSC selection
    dfs_channels:    list[int]   # flagged channels
    cci_risk_score:  float        # 0.0 (none) → 1.0 (severe)
    cci_risk_label:  str          # "LOW" / "MEDIUM" / "HIGH"

class ConfigPackage(BaseModel):
    session_id:          str
    ap_model:            str
    xiq_device_template: dict     # full JSON ready to POST to XIQ API
    xiq_autoRF_profile:  dict     # separate AutoRF profile
    exos_cli_script:     str      # EXOS SwitchEngine CLI
    voss_cli_script:     str      # VOSS Fabric Engine CLI
    qualification_checks:list[str]

class ComparisonReport(BaseModel):
    dpm_session_id:  str
    wifi6_result:    CapacityPlan
    wifi7_result:    CapacityPlan
    delta:           dict   # key differences: AP count, MCS gain, airtime, throughput
    recommendation:  str    # Teaching Agent commentary
```

---

## L2 — Link Budget Engine

### File: `src/engines/link_budget.py`

### MCS Reference Table (per 1 SS, 20 MHz channel)

| MCS | Modulation | Code Rate | Min SNR (dB) | Rate (Mbps) | Protocol |
|-----|-----------|-----------|-------------|------------|---------|
| 0   | BPSK      | 1/2       | 2           | 8.6        | 802.11ax |
| 1   | QPSK      | 1/2       | 5           | 17.2       | 802.11ax |
| 2   | QPSK      | 3/4       | 8           | 25.8       | 802.11ax |
| 3   | 16-QAM    | 1/2       | 11          | 34.4       | 802.11ax |
| 4   | 16-QAM    | 3/4       | 14          | 51.6       | 802.11ax |
| 5   | 64-QAM    | 2/3       | 17          | 68.8       | 802.11ax |
| 6   | 64-QAM    | 3/4       | 19          | 77.4       | 802.11ax |
| 7   | 64-QAM    | 5/6       | 21          | 86.0       | 802.11ax |
| 8   | 256-QAM   | 3/4       | 24          | 103.2      | 802.11ax |
| 9   | 256-QAM   | 5/6       | 26          | 114.7      | 802.11ax |
| 10  | 1024-QAM  | 3/4       | 29          | 129.0      | 802.11ax |
| 11  | 1024-QAM  | 5/6       | 33          | 143.4      | 802.11ax |
| 12  | 4096-QAM  | 3/4       | 36          | 172.1      | 802.11be |
| 13  | 4096-QAM  | 5/6       | 38          | 191.2      | 802.11be |

**Scale factor:** Rate × SS × (width_mhz / 20). Example: MCS 11, 2SS, 80 MHz = 143.4 × 2 × 4 = 1,147 Mbps

### Algorithm

```python
class LinkBudgetStrategy(ABC):
    @abstractmethod
    def snr_to_mcs(self, protocol: WiFiProtocol, channel_width: int,
                   snr_db: float, spatial_streams: int) -> MCSResult: ...
    @abstractmethod
    def max_mcs_index(self) -> int: ...

class WiFi6LinkBudget(LinkBudgetStrategy):
    MCS_TABLE = { ... }  # MCS 0-11 with SNR thresholds above

    def max_mcs_index(self) -> int: return 11

    def snr_to_mcs(self, protocol, channel_width, snr_db, spatial_streams) -> MCSResult:
        eligible = [m for m in self.MCS_TABLE.values()
                    if m.snr_required_db <= snr_db and m.mcs_index <= self.max_mcs_index()]
        best = max(eligible, key=lambda m: m.mcs_index)
        base_rate = best.data_rate_1ss_20mhz
        scaled_rate = base_rate * spatial_streams * (channel_width / 20)
        return MCSResult(data_rate_mbps=scaled_rate, ...)

class WiFi7LinkBudget(WiFi6LinkBudget):
    MCS_EXTENSION = {  # MCS 12 + 13
        12: MCSRow(mcs_index=12, modulation="4096-QAM", coding_rate="3/4",
                   snr_required_db=36, data_rate_1ss_20mhz=172.1),
        13: MCSRow(mcs_index=13, modulation="4096-QAM", coding_rate="5/6",
                   snr_required_db=38, data_rate_1ss_20mhz=191.2),
    }

    def max_mcs_index(self) -> int: return 13

    def snr_to_mcs(self, protocol, channel_width, snr_db, spatial_streams) -> MCSResult:
        if snr_db >= 36 and protocol == WiFiProtocol.BE:
            eligible_ext = [m for m in self.MCS_EXTENSION.values()
                            if m.snr_required_db <= snr_db]
            if eligible_ext:
                best = max(eligible_ext, key=lambda m: m.mcs_index)
                scaled_rate = best.data_rate_1ss_20mhz * spatial_streams * (channel_width / 20)
                return MCSResult(data_rate_mbps=scaled_rate, ...)
        return super().snr_to_mcs(protocol, channel_width, snr_db, spatial_streams)


def run_link_budget(dpm: DesignParameterManifest,
                    strategy: LinkBudgetStrategy) -> list[LinkBudgetResult]:
    results = []
    for device in dpm.clients:
        for band in device.band_support:
            rssi = dpm.network.coverage_rssi_dbm
            noise = dpm.site.noise_floor_dbm
            snr   = rssi - noise
            mcs   = strategy.snr_to_mcs(device.protocol,
                                          dpm.network.channel_width_for_band(band),
                                          snr, device.tx_chains)
            airtime = compute_airtime_per_device(device, dpm.applications, mcs)
            results.append(LinkBudgetResult(
                device_type=device.device_type, band=band,
                rssi_design_dbm=rssi, noise_floor_dbm=noise,
                snr_db=snr, mcs=mcs, airtime_pct_per_device=airtime
            ))
    return results
```

---

## L3 — Airtime Calculator

### File: `src/engines/airtime.py`

### Core Formula (Revolution Wi-Fi)

```
Airtime per device (%) = (App_Throughput_Kbps × overhead_factors) / Device_Throughput_Kbps

Available Airtime per Radio (%) = RF_Environment_Airtime_% − SSID_Beacon_Overhead_%

AP Radios Required = Σ(Active_Devices × Airtime_per_Device) / Available_Airtime_per_Radio
```

### SSID Beacon Overhead Lookup

| SSIDs | 2.4 GHz Overhead | 5 GHz Overhead | 6 GHz Overhead |
|-------|-----------------|----------------|----------------|
| 1     | 2.5%            | 2.5%           | 1.5%           |
| 2     | 5.0%            | 5.0%           | 3.0%           |
| 3     | 7.5%            | 7.5%           | 4.5%           |
| 4     | 10.0%           | 10.0%          | 6.0%           |
| 5     | 12.5%           | 12.5%          | 7.5%           |
| 6     | 15.0%           | 15.0%          | 9.0%           |

6 GHz overhead is lower: beacons broadcast at higher MBR baseline.

### VoIP Trap
G.711 (128 Kbps): small frames (160 bytes) + no AMPDU = airtime formula gives 1.68%, **not** 0.64%. Factor: 2.5× due to frame overhead. The engine computes this correctly via `avg_frame_bytes` + `latency_sensitive` flags.

### OFDMA Modifier
When OFDMA is enabled and > 50% of active devices have frames < 256 bytes:
```python
ofdma_efficiency_gain = 1.30  # 30% — conservative (matches Rev Wi-Fi guidance)
available_airtime_pct *= ofdma_efficiency_gain
```

### Algorithm

```python
def calculate_airtime_demand(
    dpm: DesignParameterManifest,
    link_budgets: list[LinkBudgetResult],
    band: Band
) -> AirtimeDemand:
    ssid_overhead = SSID_OVERHEAD_TABLE[dpm.network.num_ssids][band]
    rf_airtime    = dpm.site.available_airtime_pct
    available     = rf_airtime - ssid_overhead

    if dpm.network.wifi_generation in (WiFiGen.WIFI6, WiFiGen.WIFI6E, WiFiGen.WIFI7):
        available *= ofdma_efficiency_modifier(dpm.clients, dpm.applications)

    total_airtime_demand = 0.0
    for device in dpm.clients:
        if band not in device.band_support:
            continue
        lb = get_link_budget(link_budgets, device.device_type, band)
        for app in dpm.applications.get(device.device_type, []):
            effective_throughput_kbps = (
                app.throughput_kbps
                * app.tcp_overhead_factor
                * app.aggregation_factor
            )
            airtime_pct = effective_throughput_kbps / (lb.effective_throughput_mbps * 1000) * 100
            if app.background:
                # background app adds airtime once, not per device
                total_airtime_demand += airtime_pct
            else:
                total_airtime_demand += airtime_pct * device.active_count

    radios_required = math.ceil(total_airtime_demand / available)
    return AirtimeDemand(
        radios_required_capacity=radios_required,
        available_airtime_pct=available,
        ssid_overhead_pct=ssid_overhead,
        total_demand_pct=total_airtime_demand
    )
```

---

## L4 — AP Selection Engine

### File: `src/engines/ap_selector.py`

### AP Model Library

```python
AP_LIBRARY = {
    "AP3000": APModel(model_id="AP3000", wifi_generation=WiFiGen.WIFI6,
                      bands=[Band.GHZ24, Band.GHZ5], max_ss=2,
                      poe_required=PoEClass.AT, sdr=False, multi_ap_coord=False,
                      max_cw_5ghz=ChannelWidth.W80),
    "AP4000": APModel(model_id="AP4000", wifi_generation=WiFiGen.WIFI6E,
                      bands=[Band.GHZ24, Band.GHZ5, Band.GHZ6], max_ss=2,
                      poe_required=PoEClass.BT, sdr=False, multi_ap_coord=False,
                      max_cw_5ghz=ChannelWidth.W160, max_cw_6ghz=ChannelWidth.W160),
    "AP5020": APModel(model_id="AP5020", wifi_generation=WiFiGen.WIFI7,
                      bands=[Band.GHZ24, Band.GHZ5, Band.GHZ6], max_ss=4,
                      poe_required=PoEClass.AT,  # or BT for full power
                      sdr=True, multi_ap_coord=True,
                      max_cw_5ghz=ChannelWidth.W160, max_cw_6ghz=ChannelWidth.W320),
    "AP5022": APModel(model_id="AP5022", wifi_generation=WiFiGen.WIFI7,
                      bands=[Band.GHZ24, Band.GHZ5, Band.GHZ6], max_ss=4,
                      poe_required=PoEClass.BT, sdr=True, multi_ap_coord=True,
                      security_radio=True, max_cw_6ghz=ChannelWidth.W320),
    "AP5060": APModel(model_id="AP5060", wifi_generation=WiFiGen.WIFI7,
                      bands=[Band.GHZ24, Band.GHZ5, Band.GHZ6],
                      max_ss=4, poe_required=PoEClass.BT, sdr=True, multi_ap_coord=True,
                      outdoor=True, max_cw_6ghz=ChannelWidth.W320),
}
```

### Selection Algorithm

```python
def select_ap(dpm: DesignParameterManifest) -> APSelectionResult:
    requested_model = dpm.network.ap_model
    ap = AP_LIBRARY[requested_model]

    warnings = []

    # PoE check
    if ap.poe_required == PoEClass.BT:
        warnings.append("AP requires 802.3bt (90W PoE). Verify switch port PoE budget.")

    # 6 GHz without BT PoE (AP5020 at at-class = reduced TX on 6GHz radio)
    if Band.GHZ6 in ap.bands and ap.poe_required == PoEClass.AT:
        warnings.append("AP5020 on 802.3at: 6 GHz radio TX power reduced. Recommend 802.3bt.")

    # SDR flag — suggest when 6 GHz capacity demand is high
    if ap.sdr and dpm.network.client_band_dist.get(Band.GHZ6, 0) > 0.5:
        warnings.append("Consider SDR dual-6GHz mode (AP5020/5060): doubles 6 GHz AP capacity.")

    # Multi-AP Coord — only useful with WiFi 7 clients
    if ap.multi_ap_coord and dpm.network.wifi_generation == WiFiGen.WIFI7:
        notes = ["Multi-AP Coordination active. Requires AP5020/5060 cluster ≥2 APs."]

    return APSelectionResult(model=ap, warnings=warnings, notes=notes)
```

---

## L5 — Channel Planning Engine

### File: `src/engines/channel_planner.py`

### Channel Sets by Regulatory Domain

```python
CHANNEL_SETS = {
    "US": {
        "5ghz_unii1":  [36, 40, 44, 48],
        "5ghz_unii2a": [52, 56, 60, 64],    # DFS
        "5ghz_unii2c": [100,104,108,112,116,120,124,128,132,136,140,144],  # DFS
        "5ghz_unii3":  [149, 153, 157, 161, 165],
        "6ghz_psc":    [5,21,37,53,69,85,101,117,133,149,165,181,197],
    },
    "EU": {
        "5ghz_unii1":  [36, 40, 44, 48],
        "5ghz_unii2a": [52, 56, 60, 64],    # DFS
        "5ghz_unii2c": [100,104,108,112,116,120,124,128,132,136,140,144],  # DFS
        # No UNII-3 in EU
        "6ghz_psc":    [5,21,37,53,69,85,101,117],  # EU 6 GHz limited
    },
}

# 5 GHz non-overlapping 80 MHz channels (US, no DFS)
CHANNELS_80MHZ_NODFS_US = [36, 52→skip, 149]   # U-NII-1: 36-48, U-NII-3: 149-165
# With DFS: adds U-NII-2a (52,60) + U-NII-2c (100-144) → up to 14 × 80 MHz channels
```

### CCI Scoring

```python
def cci_score(ap_count: int, available_channels: int) -> tuple[float, str]:
    """
    CCI risk: how many APs share a channel.
    Target: each channel used by < 1/3 of APs in range.
    """
    reuse = ap_count / available_channels if available_channels > 0 else 999
    if reuse < 1.5:
        return reuse / 5.0, "LOW"      # < 0.3 score
    elif reuse < 3.0:
        return reuse / 5.0, "MEDIUM"
    else:
        return min(reuse / 5.0, 1.0), "HIGH"
```

### Channel Plan Output

```python
def plan_channels(dpm: DesignParameterManifest,
                  capacity_plan: CapacityPlan) -> ChannelPlan:
    domain = dpm.site.regulatory_domain
    sets = CHANNEL_SETS.get(domain, CHANNEL_SETS["US"])

    ch_24 = [1, 6, 11]  # always fixed

    # 5 GHz — build available set based on DFS preference
    ch_5 = list(sets["5ghz_unii1"]) + list(sets["5ghz_unii3"])
    if dpm.network.dfs_enabled:
        ch_5 += sets.get("5ghz_unii2a", []) + sets.get("5ghz_unii2c", [])
    # Trim to non-overlapping channels for selected width
    ch_5_nonoverlapping = non_overlapping(ch_5, dpm.network.channel_width_5ghz)

    # 6 GHz — PSC only (out-of-band discovery via RNR)
    ch_6 = sets.get("6ghz_psc", [])
    ch_6_nonoverlapping = psc_channels_for_width(ch_6, dpm.network.channel_width_6ghz)

    cci, label = cci_score(capacity_plan.ap_count_final, len(ch_5_nonoverlapping))

    return ChannelPlan(
        channels_24ghz=ch_24,
        channels_5ghz=ch_5_nonoverlapping,
        channels_6ghz=ch_6_nonoverlapping,
        dfs_channels=sets.get("5ghz_unii2a",[]) + sets.get("5ghz_unii2c",[]) if dpm.network.dfs_enabled else [],
        cci_risk_score=cci,
        cci_risk_label=label
    )
```

---

## L6 — Intake Agent (Socratic → DPM)

### File: `src/agents/intake_agent.py`

### Dialogue Flow State Machine

```
START
  │
  ▼ Section A — Site & Environment (8 questions)
  Site type → Area m² → Floors → Ceiling height → Wall type
  → RF environment → Regulatory domain → Coverage tier
  │
  ▼ Section B — Client Devices (per device type, repeat)
  Device type → Protocol → Chains → Band support
  → Quantity → Concurrent % → Band distribution
  │
  ▼ Section C — Applications (per device type)
  App type → Throughput SLA → Background flag
  │
  ▼ Section D — Network Design
  AP model → Channel widths → SSIDs → MBR → Growth %
  │
  ▼ Section E — Channel Planning
  DFS preference → CCI threshold
  │
  ▼ Section F — Security
  Security per SSID → 802.11r/k/v → VLANs
  │
  ▼ Section G — RRM
  RRM mode → TX power bounds → AutoRF thresholds
  │
  IF wifi_generation == WIFI7:
  ▼ Section H — WiFi 7
  MLO mode → 320 MHz → Puncturing → Multi-AP Coord
  │
  ▼ SHOW DPM to user (with source flags)
  User calibrates any field → engine re-derives dependents
  │
  ▼ CONFIRM → run DesignEngine
```

### Teaching Agent triggers during intake

Each question includes a Teaching Agent annotation:
- Site type → "In 5G this would be your propagation environment (UMa/UMi/InH)"
- Noise floor → "Derived from RF environment — equivalent to thermal noise floor + interference rise"
- DFS → "DFS channels = conditional use spectrum. Radar event = 60s channel evacuate — like a preemptive refarming event"

---

## L7 — Design Agent + Orchestrator

### File: `src/agents/design_agent.py`, `src/agents/orchestrator.py`

### Design Engine Iterative Loop

```python
def run_design(dpm: DesignParameterManifest,
               strategy: LinkBudgetStrategy) -> DesignResult:

    iteration = 0
    MAX_ITER  = 5

    while iteration < MAX_ITER:
        # Step 1: Link Budget → device throughput capability
        link_budgets = run_link_budget(dpm, strategy)

        # Step 2: Airtime → radios required (capacity-driven)
        airtime = {}
        for band in [Band.GHZ5, Band.GHZ6, Band.GHZ24]:
            airtime[band] = calculate_airtime_demand(dpm, link_budgets, band)
        ap_count_capacity = max(a.radios_required_capacity for a in airtime.values())

        # Step 3: Coverage → APs required (coverage-driven)
        ap_count_coverage = coverage_driven_count(dpm)

        # Step 4: Final AP count with growth headroom
        ap_count_final = math.ceil(
            max(ap_count_capacity, ap_count_coverage) * (1 + dpm.network.capacity_growth_pct)
        )

        # Step 5: Validate gap
        gap_pct = abs(ap_count_capacity - ap_count_coverage) / max(1, ap_count_capacity)
        if gap_pct <= 0.20:
            break  # converged

        # Step 6: Adjust parameters to close gap
        dpm = adjust_for_gap(dpm, ap_count_capacity, ap_count_coverage)
        iteration += 1

    # Step 7: AP Selection
    ap_selection = select_ap(dpm)

    # Step 8: Channel Planning
    capacity_plan = CapacityPlan(ap_count_final=ap_count_final, ...)
    channel_plan  = plan_channels(dpm, capacity_plan)

    # Step 9: RAG Validation
    validation = rag_validator.validate(dpm, capacity_plan, channel_plan)

    return DesignResult(
        dpm=dpm, link_budgets=link_budgets, capacity_plan=capacity_plan,
        channel_plan=channel_plan, ap_selection=ap_selection,
        rag_validation=validation, iterations=iteration
    )
```

### Orchestrator State Machine

```
STATES:
  INTAKE      → collecting Socratic answers
  DPM_REVIEW  → user reviewing/calibrating DPM
  DESIGNING   → DesignEngine running
  COMPARING   → ComparisonEngine running (WiFi6 vs WiFi7)
  CONFIGURING → ConfigAgent generating templates
  VALIDATING  → RAG validation pass
  COMPLETE    → DesignSession ready

TRANSITIONS:
  INTAKE      → DPM_REVIEW   (on: intake_complete)
  DPM_REVIEW  → DESIGNING    (on: dpm_confirmed)
  DPM_REVIEW  → INTAKE       (on: user_edit → re-derive affected fields)
  DESIGNING   → COMPARING    (on: design_complete, if compare_mode)
  DESIGNING   → CONFIGURING  (on: design_complete, if single_mode)
  COMPARING   → CONFIGURING  (on: comparison_complete)
  CONFIGURING → VALIDATING   (on: config_generated)
  VALIDATING  → COMPLETE     (on: rag_pass)
  VALIDATING  → CONFIGURING  (on: rag_fail → Teaching Agent explains → adjust params)
```

---

## L8 — Config Agent

### File: `src/agents/config_agent.py`

### XIQ Device Template Structure

```json
{
  "template_name": "WiFiTwin-{ap_model}-{session_id}",
  "description":   "Auto-generated by WiFi Digital Twin v1",
  "ap_model":      "AP5020",
  "radio_profiles": {
    "wifi0_24ghz": {
      "channel":       "auto",
      "channel_width": 20,
      "tx_power_min":  7,
      "tx_power_max":  20,
      "mbr":           12,
      "ofdma":         true,
      "bss_color":     "auto",
      "twt":           true
    },
    "wifi1_5ghz": {
      "channel":       "auto",
      "channel_width": 80,
      "tx_power_min":  13,
      "tx_power_max":  23,
      "mbr":           12,
      "ofdma":         true,
      "mlo_enabled":   true,
      "puncturing":    false
    },
    "wifi2_6ghz": {
      "channel":       "psc",
      "channel_width": 80,
      "tx_power_min":  16,
      "tx_power_max":  23,
      "mbr":           12,
      "ofdma":         true,
      "mlo_enabled":   true,
      "puncturing":    true,
      "security":      "wpa3-sae",
      "mfp":           "required",
      "rnr":           true
    }
  },
  "ssid_profiles":  [ ... ],
  "ft_enabled":     true,
  "neighbor_report":true,
  "bss_transition": true
}
```

### AutoRF Profile (separate, dual output)

```json
{
  "autoRF_profile_name": "WiFiTwin-AutoRF-{session_id}",
  "cci_threshold_dbm":   -85,
  "retry_threshold_pct": 20,
  "util_threshold_pct":  70,
  "scan_interval_sec":   300,
  "dynamic_channel_width": false,
  "client_load_balancing": false
}
```

### EXOS CLI Script Template

```
# WiFi Digital Twin — EXOS Switch Config
# AP: {ap_model} | Session: {session_id} | Generated: {timestamp}

# VLAN setup
create vlan "{ssid_name}" tag {vlan_id}
configure vlan "{ssid_name}" add ports {ap_port} tagged

# PoE — verify budget before enabling
configure inline-power ports {ap_port} operator-limit {poe_watts}
enable inline-power ports {ap_port}

# Management VLAN
configure vlan "mgmt" add ports {ap_port} untagged

# QoS — WMM mapping
configure dot1p type {vlan_id} 6    # VO → CoS 6
configure dot1p type {vlan_id} 5    # VI → CoS 5
```

### VOSS CLI Script Template

```
# WiFi Digital Twin — VOSS Fabric Engine Config
# FA I-SID mapping for SSID VLAN binding

interface GigabitEthernet {port}
  flex-uni enable
  fa enable

vlan create {vlan_id} type port-mstprstp 0
vlan i-sid {vlan_id} {isid_value}
interface GigabitEthernet {port}
  vlan tagging {vlan_id}
```

---

## L9 — API Layer

### File: `src/api/main.py`

### REST Endpoints

```
POST   /api/v1/session                → create DesignSession, return session_id
DELETE /api/v1/session/{id}           → terminate session

POST   /api/v1/intake/answer          → submit one Socratic answer, return next question
POST   /api/v1/dpm/{session_id}       → submit calibrated DPM, trigger validation
GET    /api/v1/dpm/{session_id}       → retrieve current DPM with source flags

POST   /api/v1/design/{session_id}    → run DesignEngine (strategy from DPM.wifi_gen)
GET    /api/v1/design/{session_id}    → retrieve DesignResult
POST   /api/v1/compare/{session_id}   → run ComparisonEngine (WiFi6 + WiFi7 on same DPM)
GET    /api/v1/compare/{session_id}   → retrieve ComparisonReport

GET    /api/v1/config/{session_id}    → retrieve ConfigPackage
POST   /api/v1/validate/{session_id}  → trigger RAG validation pass

GET    /api/v1/health                 → service health check
```

### WebSocket Channels

```
WS /ws/intake/{session_id}
  Client → server: { "answer": "...", "field": "site_type" }
  Server → client: { "next_question": "...", "dpm_field": "...",
                     "teaching_note": "...", "source_flag": "User provided" }

WS /ws/design/{session_id}
  Server → client: { "step": "link_budget|airtime|channel|config",
                     "progress_pct": 0-100, "result": {...} }
```

### Data Flow (E2E)

```
Client (browser/CLI)
    │ POST /session
    ▼
DesignSession created (PostgreSQL) + Redis presence entry
    │ WS /intake
    ▼
IntakeAgent: Socratic loop (LangChain ConversationChain + DPM builder)
    │ POST /dpm (user confirms)
    ▼
DesignEngine.run(dpm, strategy)
    ├── LinkBudgetEngine    → LinkBudgetResult[]
    ├── AirtimeCalculator   → AirtimeDemand per band
    ├── APSelectionEngine   → APSelectionResult
    ├── ChannelPlanner      → ChannelPlan
    └── RAGValidator        → ValidationReport
    │ POST /config
    ▼
ConfigAgent.generate(dpm, design_result) → ConfigPackage
    │ GET /config
    ▼
User downloads: XIQ template JSON + EXOS CLI + VOSS CLI + AutoRF profile
```

---

## Appendix A — Mixed-Mode Input Parameters (Future Sprint 4+)

**Why this appendix exists:** Phase 1 designs for uniform WiFi 6 or WiFi 7. A future sprint (feature/mixed-topology) will allow co-existing AP generations on the same site. That sprint requires the following inputs BEYOND the Phase 1 DPM. These are documented here so the LLD for mixed-mode has a clear starting point.

### Additional Parameters Required for Mixed-Mode

| Parameter | Type | Purpose |
|-----------|------|---------|
| Floor plan file | SVG / DXF / PNG upload | Source for AP placement coordinate system |
| Floor plan scale | float (m/pixel) | Convert pixel coordinates to physical distance |
| AP placement list | list[(x, y, ap_model, zone_id)] | Per-AP position + model assignment |
| Zone definitions | list[Polygon] | Named RF zones with boundary polygons |
| Wall/obstacle layer | GeoJSON or raster mask | Attenuation layer per wall segment |
| Zone-to-zone adjacency | graph[zone_id → zone_id] | Derived from placement proximity |
| Roam boundary pairs | list[(zone_a, zone_b)] | Zone pairs where client roaming is expected |
| Per-zone channel width | dict[zone_id, ChannelWidth] | May differ at 320/160 MHz boundaries |
| Multi-AP Coord clusters | list[list[ap_id]] | Which WiFi 7 APs form coordination clusters |
| Inter-generation roam policy | enum | Drop MLO on roam / notify user / block |
| Mixed-mode comparison target | enum | vs pure-WiFi6 / vs pure-WiFi7 / vs other-mixed |
| Zone AP generation | dict[zone_id, WiFiGen] | WiFi 6 vs WiFi 7 per zone |

### What Phase 1 LLD Provides to Mixed-Mode

Phase 1 builds the interfaces that mixed-mode will inherit:
- `LinkBudgetStrategy` ABC — `MixedAPStrategy` implements it per zone
- `ComparisonEngine` — already outputs WiFi6 vs WiFi7; mode 3 adds mixed
- `DesignParameterManifest` — extends with zone-keyed variants
- `ChannelPlan` — extends with per-zone channel assignments + boundary reconciliation
- `ConfigPackage` — extends to emit one template per AP model in the fleet

Mixed-mode does NOT rewrite Phase 1 engines — it wraps them with zone routing.
