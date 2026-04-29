# WiFi Security Design

## Security Mode Decision Matrix
| Scenario | Recommended Mode | Why |
|----------|-----------------|-----|
| Enterprise staff (no RADIUS) | WPA3-SAE or WPA2/WPA3 Transition | Forward secrecy, dictionary-attack resistant |
| Enterprise staff (with RADIUS) | WPA3-Enterprise / 802.1X | Per-user credentials, certificate-based |
| Guest / visitor | OWE + Transition Mode | Encrypted open — no password, broad compat |
| IoT / device-only | WPA2-PSK (separate VLAN) | Most IoT devices don't support WPA3 |
| Legacy-only network | WPA2-PSK | When WPA3 clients not present |

## Security Standard Comparison
| Feature | WPA2-PSK | WPA3-SAE | OWE | 802.1X |
|---------|----------|----------|-----|--------|
| Password | Shared key | SAE handshake | None | Per-user cert/cred |
| Forward secrecy | No | Yes | Yes | Yes |
| Offline dictionary attack | Vulnerable | Resistant | N/A | N/A |
| PMF required | Optional | Mandatory | Mandatory | Optional |
| Legacy device compat | Excellent | Good (transition mode) | Via Transition | Poor (cert overhead) |
| Deployment complexity | Low | Low-Medium | Low | High (RADIUS server) |

## WPA3-SAE (Simultaneous Authentication of Equals)
- Replaces the PSK 4-way handshake with a Dragonfly handshake
- Each session generates a unique PMK — compromising one session doesn't expose others
- Resistant to offline dictionary attacks (even if attacker captures the 4-way handshake)
- XIQ: set Security to WPA2/WPA3 for transition mode (WPA3 where supported, WPA2 fallback)

## OWE — Opportunistic Wireless Encryption (Enhanced Open)
Based on IEEE 802.11-2016. Encrypts open networks without requiring a password.
- Every client session gets a unique encryption key
- Protects against passive eavesdropping — attacker can't read your traffic even on "open" WiFi
- **Does NOT authenticate** — anyone can connect (still open access-wise)
- Ideal for guest networks where you want encryption without the friction of a password

### OWE Transition Mode (Horizon Pattern)
Run a hidden Open SSID alongside the OWE SSID with the same network name:
- Modern devices (Android 10+, iOS 14+, Windows 10 2004+) → connect to OWE (encrypted)
- Legacy devices → fall back to hidden Open (unencrypted, but can connect)
- XIQ: Guest SSID → toggle "Transition Mode for 2.4GHz and 5GHz"
- ⚠ This toggle is NOT on by default — must be explicitly enabled

## PMF (Protected Management Frames / 802.11w)
| Mode | WPA2 | WPA3 | OWE |
|------|------|------|-----|
| PMF Optional (mfpc) | Default | N/A | N/A |
| PMF Required (mfpr) | Configurable | Always | Always |

PMF protects deauthentication and disassociation frames — prevents rogue deauth attacks that disconnect clients. Enable PMF Required for all WPA3 and OWE SSIDs.

## 802.1X Enterprise Authentication
Requires a RADIUS server (on-prem or cloud):
- EAP-TLS: certificate-based — most secure, most complex
- PEAP-MSCHAPv2: username/password with server certificate — most common
- EAP-TTLS: similar to PEAP, more flexible inner methods

For small/medium deployments without a RADIUS server, use WPA3-SAE PSK instead.

## Horizon Security Implementation
```
Corporate_Wireless:
  Security: WPA2/WPA3 Transition (SAE + PSK)
  PMF: Optional (WPA2 compat) → Required recommended for new deployments
  Key Value: [set in XIQ — was EMPTY at Apr 21 deployment, must confirm]
  VLAN: 20

Guest_Wireless:
  Security: OWE
  Transition Mode: ENABLED (toggle in SSID settings)
  PMF: Required (mandatory for OWE)
  VLAN: 30
```

## What Breaks Without OWE Transition Mode
Legacy devices (Windows 7/8, older Android) will fail to connect to the Guest SSID and see "Authentication failed" or just never attempt to connect. They cannot speak OWE. Enable Transition Mode to run the hidden fallback Open network alongside OWE.
