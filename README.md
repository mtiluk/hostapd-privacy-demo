## Final-Year Project / Research Prototype

This repository contains my final year project proof of concept demonstrating that network identification during discovery can be performed with an obfuscated pseudo-SSID rather than a plaintext SSID.

Coventional IEEE 802.11 clients leak plaintext SSIDs during active discovery through Probe Requests. Those SSIDs can be passively collected and then correlated with geolocation databases to infer user movement, home/work locations, or device history. This project explores whether discovery can instead use a stable but obfuscated alias per device and AP.

***Important:*** This is an academic research prototype and is not production ready. 

## Project Objective

This prototype demonstrates a modified assocation and rediscovery workflow where:
1. A station first connects normally using the plaintext SSID set on the AP.
2. The AP derives a per-station token.
3. The AP provisions that token to the station in the Association Response via a vendor-specific IE.
4. The station stores the token.
5. On future connections, the station performs wildcard SSID probing and includes the token in a vendor-specific IE.
6. The AP can use that token as the discovery handle instead of requiring a directed plaintext SSID probe request.

## Technical Summary

For each associating station, the AP derives a 16-byte discovery token to be used as an obfuscated alias for discovery, not secrecy:

```text
token = SHA256(BSSID || STA_MAC)[0..15]
```

This was chosen becuase it is:
 - Per-station: each device receives a different token.
 - Stable: the same STA and AP produce the same token across reconnects.
 - Pre-associatin available: only MAC addresses are needed.
 - Lightweight: no PMK or additional key management is required.
 - Efficient: truncating SHA-256 to 16 bytes keeps IE overhead small.

On first association, the STA discovers the AP using the normal SSID. The AP creates per-STA state, derives the token from `BSSID} || STA_MAC`, stores it in the STA WPA state machine, and appends it to the Association Response in a vendor-specific Information Element. The STA then receives and stores the token.

On later association or rediscovery, the STA loads the stored token and sends a Probe Request with a wildcard plus a vendor-specific IE carrying the token. The AP parses this IE, matches the token, and proceeds with the normal discovery and association flow.

The token is provisioned in a Vendor Information Element carried in the Association Response using the following format:

| Field | Value | Description |
|------|------|-------------|
| Element ID | 221 | Vendor Specific IE |
| OUI | `00:11:22` | Private/unassigned demo OUI |
| Subtype | `0x01` | Custom token IE type |
| Payload | 16 bytes | Binary discovery token |
| IE length | 20 bytes | 3 (OUI) + 1 (subtype) + 16 (token) |

## Implementation Summary

This prototype was implemented across both the AP (hostapd) and STA (wpa_supplicant) to support token provisioning and token-based rediscovery.

On the AP side, the implementation adds token generation during STA state creation, token storage in per-STA WPA state, token registration in an AP token lookup table, insertion of a vendor-specific IE into the Association Response, and support for parsing and matching token-carrying Probe Requests.

On the STA side, the implementation adds token extraction from the Association Response, persistence of the token in network configuration, wildcard SSID scanning when a token is available, insertion of the vendor-specific token IE into Probe Requests, and reuse of the stored token on reconnect.

This prototype successfully demonstrated:
1. AP-side token derivation using `SHA256(BSSID || STA_MAC)[0..15]`
2. Token provisioning via a vendor-specific IE in the Association Response
3. Token persistence in STA configuration
4. Token-based rediscovery using wildcard SSID plus vendor-specific IE
5. Elimination of directed plaintext SSID probing during reconnect scans

In practice, second-pass reconnect behavior showed wildcard SSID probing, presence of the token IE in the Probe Request, and reuse of the same token value that had previously been provisioned by the AP.

Validation was performed using `hostapd` runtime logs, `wpa_supplicant` runtime logs, persisted configuration inspection, and packet capture analysis with `tcpdump`, `tshark`, and Wireshark.

Example AP-side evidence:
```text
DEMO TOKEN (derived: SHA256(BSSID||STA_MAC)) - hexdump(len=16): 4b a6 51 73 e2 ef 38 16 4a 76 33 53 9a db 37 05
DEMO: token registered for 90:de:80:88:ce:60
DEMO: Adding demo_token IE for 90:de:80:88:ce:60
```

Example STA-side evidence:
```text
DEMO: parsing demo_token from config
DEMO: parsed demo_token - hexdump(len=16): 4b a6 51 73 e2 ef 38 16 4a 76 33 53 9a db 37 05
DEMO: using wildcard SSID for token-based scan
DEMO: scan_ssid present has_demo_token=1 len=16
DEMO: adding token IE to Probe Request
```

## Modified Files

### AP-side

| File | Change |
|------|--------|
| `src/ap/hostapd.h` | Added token lookup table structures |
| `src/ap/hostapd.c` | Added token table allocation and cleanup |
| `src/ap/wpa_auth_i.h` | Added token storage to `struct wpa_state_machine` |
| `src/ap/wpa_auth.c` | Added token derivation, registration, lookup, and helper functions |
| `src/ap/wpa_auth.h` | Added public token helper declarations |
| `src/ap/ieee802_11.c` | Added token IE builder and integrated token into Association Response |
| `src/ap/beacon.c` | Added Probe Request token parsing and token match support |

### STA-side

| File | Change |
|------|--------|
| `wpa_supplicant/config_ssid.h` | Added per-network token storage fields |
| `wpa_supplicant/config.c` | Added token parser and config writer helper |
| `wpa_supplicant/config_file.c` | Added token persistence to config file |
| `wpa_supplicant/events.c` | Added token extraction from Association Response |

## Design Notes and Scope

This implementation requires SoftMAC/mac80211-based hardware because it modifies management frame behavior in software. FullMAC devices are generally unsuitable for this prototype because management frame generation is typically handled in firmware.

The token is intentionally sent in plaintext. Its purpose is obfuscation rather than secrecy, and it acts as a discovery alias rather than a confidential credential. The design does not claim cryptographic confidentiality or strong anti-tracking guarantees.

On the AP, the token is stored in struct wpa_state_machine, which provides one token per station, keeps token lifecycle aligned with STA state, and makes the token easily available during Association Response construction.

The current implementation includes:
- AP token generation
- AP token provisioning
- STA token storage
- STA token persistence
- STA wildcard Probe Request behavior
- STA token IE insertion into Probe Requests
- AP token parsing support
- AP token lookup support

The following are not currently in scope and remain future work:
- Standards-compliant client support without patching wpa_supplicant
- Long-term AP-side persistence independent of reassociation lifecycle
- Token expiry or rotation
- Hidden-SSID policy hardening
- Wider interoperability testing
- Production-grade privacy or anti-tracking guarantees

## Requirements and Test Environment

This project requires a compatible SoftMAC device.

| Component | Requirement |
|-----------|-------------|
| Platform | Linux-based device, tested on Raspberry Pi 4 Model B |
| Wi-Fi adapters | SoftMAC adapters using mac80211 |
| Tested chipset class | MediaTek MT7612U |
| Driver | `mt76x2u` |
| AP daemon | Modified `hostapd` |
| Client | Modified `wpa_supplicant` |
| Capture tools | `tcpdump`, `tshark`, Wireshark |

A recommended lab setup is:

| Interface | Role |
|-----------|------|
| `eth0` | Management / SSH |
| `wlan1` | Access Point (`hostapd`) |
| `wlan2` | Station / Client (`wpa_supplicant`) |
| `mon0` | Optional monitor interface for packet capture |

Notes:

- SoftMAC/mac80211 hardware is required because management frames must be modified in software.
- FullMAC devices are generally unsuitable because management frame generation is handled in firmware.
- A powered USB hub is strongly recommended when using multiple USB Wi-Fi adapters on a Raspberry Pi.

## Build and Runtime

Install dependencies:
```bash
sudo apt-get update
sudo apt-get install -y \
  build-essential \
  git \
  pkg-config \
  libnl-3-dev \
  libnl-genl-3-dev \
  libnl-route-3-dev \
  libssl-dev
```

Build `hostapd`:
```bash
cd hostapd
cp defconfig .config
make clean
make -j4
```

Build `wpa_supplicant`:
```bash
cd wpa_supplicant
cp defconfig .config
make clean
make -j4
```

If DBus build support causes issues in `wpa_supplicant`, disable it in `.config`:
```
# CONFIG_CTRL_IFACE_DBUS_NEW=y
# CONFIG_CTRL_IFACE_DBUS_INTRO=y
```

Example AP configuration:
```text
interface=wlan1
driver=nl80211
ssid=DemoNetwork
hw_mode=g
channel=6
wmm_enabled=1
auth_algs=1
ignore_broadcast_ssid=0
wpa=2
wpa_key_mgmt=WPA-PSK
rsn_pairwise=CCMP
wpa_passphrase=testpassword123
```

Example STA configuration:
```text
update_config=1

network={
    ssid="DemoNetwork"
    psk="testpassword123"
    scan_ssid=1
}
```

## Validation

Create a monitor interface:
```bash
sudo iw dev wlan1 interface add mon0 type monitor
sudo ip link set mon0 up
```

Capture traffic:
```bash
sudo tcpdump -i mon0 -s 512 -w /tmp/rediscovery.pcap
```
