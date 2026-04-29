# WiFi Glossary

| Term | Definition |
|------|-----------|
| **802.1X** | IEEE standard for port-based network access control using EAP and RADIUS |
| **802.11k** | Neighbour Report — AP shares list of nearby APs with client to accelerate roaming |
| **802.11r** | Fast BSS Transition — reduces re-authentication time from ~100ms to <20ms |
| **802.11v** | BSS Transition Management — AP can nudge a client to roam to a better AP |
| **802.11w / PMF** | Protected Management Frames — encrypts deauth/disassoc frames against rogue attacks |
| **AC (Access Category)** | WMM traffic priority class: AC_VO, AC_VI, AC_BE, AC_BK |
| **Anycast Gateway** | Both switches share the same gateway IP; IS-IS coordinates ownership — no VRRP needed |
| **AP** | Access Point — radio device clients connect to |
| **APIPA** | 169.254.x.x — self-assigned IP when DHCP fails; always means a DHCP problem |
| **ARP** | Address Resolution Protocol — maps IP to MAC address |
| **Auto-Sense** | EXOS/VOSS feature — switch port automatically detects connected device type and applies appropriate config |
| **Backbone MAC (B-MAC)** | Outer MAC header in MAC-in-MAC (802.1ah) — identifies switches in the Fabric core |
| **Band Steering** | AP pushes dual-band clients to 5/6 GHz instead of 2.4 GHz |
| **BSS** | Basic Service Set — one AP + its associated clients |
| **BSS Coloring** | WiFi 6 feature — each BSS gets a color; frames from other-color BSSs ignored, reducing co-channel interference |
| **BSSID** | MAC address of an AP radio (each SSID on each radio has its own BSSID) |
| **CoS** | Class of Service — 802.1p 3-bit field in Ethernet frame for QoS prioritisation |
| **DFS** | Dynamic Frequency Selection — mandatory radar detection before using UNII-2A/2C 5 GHz channels |
| **DHCP** | Dynamic Host Configuration Protocol — automatically assigns IP addresses to clients |
| **DSCP** | Differentiated Services Code Point — 6-bit IP header field for QoS marking |
| **EAP** | Extensible Authentication Protocol — authentication framework used with 802.1X |
| **ELAN** | Emulated LAN — I-SID service type for Layer 2 broadcast domain across the Fabric |
| **FA (Fabric Attach)** | Protocol based on LLDP allowing non-Fabric devices (like AP3000) to request I-SIDs dynamically |
| **FT** | Fast Transition — 802.11r fast roaming |
| **Fresnel Zone** | Ellipsoid of radio energy around the LOS path — must be ≥60% clear for optimal signal |
| **I-SID** | Individual Service Identifier — 24-bit global service ID in VOSS Fabric (VLAN + 100,000 convention) |
| **ipforwarding** | EXOS command to enable L3 routing on a VLAN SVI — required for inter-VLAN routing |
| **IS-IS** | Intermediate System to Intermediate System — link-state routing protocol used as VOSS Fabric control plane |
| **Karl Rule** | `enable ipforwarding vlan Default` — required on SW1 in EXOS; without it clients get IPs but no internet |
| **MAC-in-MAC** | IEEE 802.1ah — data plane encapsulation that wraps user frames in a Fabric backbone header |
| **MLO** | Multi-Link Operation — WiFi 7 feature allowing simultaneous use of 2.4+5+6 GHz bands |
| **MU-MIMO** | Multi-User MIMO — AP serves multiple clients simultaneously using separate spatial streams |
| **NNI** | Network-to-Network Interface — inter-switch Fabric link (Port 10 in Horizon lab) |
| **OFDMA** | Orthogonal Frequency-Division Multiple Access — WiFi 6 feature dividing channel into Resource Units |
| **OKC** | Opportunistic Key Caching — WPA2-Enterprise fast roaming by caching PMK at APs |
| **OWE** | Opportunistic Wireless Encryption (Enhanced Open) — encrypts open networks without a password |
| **PMF** | Protected Management Frames (802.11w) — mandatory for WPA3 and OWE |
| **PMK** | Pairwise Master Key — derived from PSK or 802.1X; used to derive session keys |
| **PoE** | Power over Ethernet (802.3af) — 15.4W max; insufficient for AP3000 |
| **PoE+** | Power over Ethernet Plus (802.3at) — 30W max; required for AP3000 (25.5W) |
| **PSC** | Preferred Scanning Channel — designated 6 GHz channels clients probe first |
| **PSK** | Pre-Shared Key — WiFi password (WPA2/WPA3 Personal) |
| **QoS** | Quality of Service — mechanisms to prioritise traffic types |
| **RADIUS** | Remote Authentication Dial-In User Service — centralised auth server for 802.1X |
| **RSSI** | Received Signal Strength Indicator — signal strength at client (dBm) |
| **SAE** | Simultaneous Authentication of Equals — WPA3 replacement for PSK 4-way handshake |
| **SNR** | Signal-to-Noise Ratio — difference between signal and noise floor (dB); ≥25 dB for voice |
| **SPBM** | Shortest Path Bridging MAC — IEEE 802.1aq; data plane using MAC-in-MAC across IS-IS topology |
| **SSID** | Service Set Identifier — the WiFi network name clients see |
| **SVI** | Switched Virtual Interface — L3 interface for a VLAN on a managed switch |
| **TWT** | Target Wake Time — WiFi 6 feature scheduling client wake windows; extends IoT battery life |
| **UNI** | User-Network Interface — edge port connecting clients/APs to the Fabric (Ports 3/5) |
| **VLAN** | Virtual LAN — logical network segment within a switch |
| **VOSS** | Virtual Office Software Suite — Extreme Networks Fabric Engine OS (formerly ERS/VSP) |
| **WMM** | WiFi Multimedia — 802.11e QoS with 4 access categories |
| **WPA2** | WiFi Protected Access 2 — AES-CCMP encryption; widely used |
| **WPA3** | WiFi Protected Access 3 — SAE, OWE, mandatory PMF; current generation |
| **XIQ** | ExtremeCloud IQ — Extreme Networks cloud management platform |
