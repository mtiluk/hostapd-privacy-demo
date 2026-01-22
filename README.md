## Overview

This repository is a proof of concept that network identification during discovery can be replaced with an obfuscated pseudo-random SSID rather than a plaintext SSID cryptographically derived. Conventional IEEE 802.11 devices leak plaintext SSIDs during network discovery via probe requests. These probes can be collected and with a geolocation database anyone can infer user movement patterns. 

**Important:** This is an academic research prototype and it is not production ready.

## Technical Explanation


For each associating station (device), the AP derives a 16-byte discovery token where `token = SHA256(BSSID || STA_MAC)[0..15]`.
- The token is derived **per-station** so that for each device it is unique.
- Same device + same AP results in the same token across reconnections. 
- Derivable using only MAC addresses (no PMK).
- 128-bit output space offering 2^64 possible combinations. 
- Not reverisble to SSID without knowledge of the derivation.

### Token Provisioning

The generated token is transmitted to the station via a **Vendor Information Element** appended to the **Association Response** frame:

| Field | Value | Description |
|-------|-------|-------------|
| Element ID | 221 | Vendor Specific IE |
| OUI | `00:11:22` | Private/unassigned (demo use) |
| Subtype | `0x01` | Custom type identifier |
| Payload | 16 bytes | Binary token (SHA256 truncation) |
| Total IE Length | 20 bytes | 3 (OUI) + 1 (subtype) + 16 (token) |


## Implementation

The prototype is implemented using a custom fork of the [hostapd](https://git.w1.fi/cgit/hostap/) repository. Where all additional code is implemented within functions already written. 

| File | Changes |
|------|---------|
| `src/ap/wpa_auth_i.h` | Added `demo_token[16]` and `demo_token_len` fields to `struct wpa_state_machine` |
| `src/ap/wpa_auth.c` | Token derivation in `wpa_auth_sta_init()`; added `wpa_auth_get_demo_token_bin()` accessor |
| `src/ap/wpa_auth.h` | Declared token accessor function for cross-module visibility |
| `src/ap/ieee802_11.c` | Implemented `hostapd_eid_demo_token()` IE builder; integrated into `send_assoc_resp()` |

### Design Reasoning

This implementation requires a **mac80211 (SoftMAC)** driver as management frames are constructed in software. Whereas, FullMAC generate frames in firmware and cannot be modified.

The token is transmitted in plaintext because:
 - There is no encryption during the Association Resposne as this occurs during the [4-way handshake.](https://networklessons.com/wireless/wpa-and-wpa2-4-way-handshake)
 - The research goal is simply cryptographic obfuscation. 

Currently, the token is stored within `struct wpa_state_machine` which ensures that one token exists per association instance and that it is automatically removed when disassociation occurs. 

## Current Implementation & Future Work

**Current Implementation:**
- Token generation at association time.
- Token storage in AP state machine.
- Token provisioning via Vendor IE in Association Response.
- Runtime logging for validation

**Future Work:**
- Modify `wpa_supplicant` to parse the IE and store token.
- Modify device to emit STA in Probe Requests.
- AP matches token to known network and responds.
- Handle token expiration.

## Hardware Requirements

> [!CAUTION]
> This project will not function without a compatible SoftMAC device.

| Component | Requirement |
|-----------|-------------|
| Platform | Linux-based AP (tested: Raspberry Pi 4 Model B) |
| Wi-Fi Adapter | SoftMAC device (mac80211 driver) |
| Driver Type | **NOT** FullMAC (firmware-based frame generation) |

## Build Instructions

### Prerequisites

```bash
sudo apt-get update
sudo apt-get install -y build-essential git libnl-3-dev libnl-genl-3-dev \
                        libssl-dev pkg-config libnl-route-3-dev
```

### Compilation

```bash
git clone https://github.com/mtiluk/hostapd-privacy-demo.git
cd hostapd-privacy-demo/hostap/hostapd
cp defconfig .config
make clean
make -j4
```

### Run

```bash
sudo ./hostapd ./hostapd.conf
```

**Configuration Notes:**
 - Edit hostapd.conf to match your wireless interface (interface=wlanX)
 - Use a SoftMAC-capable device (e.g., wlan1 via MT7612U)
 - Ensure the interface is not managed by NetworkManager (nmcli dev set wlan1 managed no)

### Configuration Example

Minimal `hostapd.conf` for testing:

```bash
#Interface
interface=wlan1
driver=nl80211

# Network Identity
ssid=ResearchAP
hw_mode=a
channel=48
ieee80211n=1
wmm_enabled=1
#ignore_broadcast_ssid=1

# WPA2-Personal
auth_algs=1
wpa=2
wpa_key_mgmt=WPA-PSK
rsn_pairwise=CCMP
wpa_passphrase=ChangeThisPassword123

# Verbose logging for probe requests (PNL)
logger_syslog=-1
logger_syslog_level=2
logger_stdout=-1
logger_stdout_level=2

# Control interface (optional, for hostapd_cli)
ctrl_interface=/var/run/hostapd
ctrl_interface_group=0
```

### Validation & Packet Capture

When a station associates, hostapd will log:

```bash
DEMO TOKEN (derived: SHA256(BSSID||STA_MAC)): cd 53 4a fc 29 36 64 8f 2d 10 7e d2 ee 12 dd 33
DEMO: Adding demo_token IE for ac:5c:2c:44:77:2a
```

To see the IE being transferred over the air, use Wireshark (tshark) to capture the Association Response:

Capture on a monitor interface:
```
sudo tshark -i mon0 -s 512 -w /tmp/assocresp.pcap
```

Open Wireshark and filter using:
```
wlan.fc.type_subtype == 0x01
```

Verify in Association Response → Tagged Parameters:

```
- Tag: Vendor Specific (221)
- OUI: 00:11:22
- Vendor Type: 1
- Data: 16 bytes matching hostapd log
```
