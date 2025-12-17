# hostapd-privacy-demo

This prototype is a proof of concept demonstrating that network identification during discovery can be decoupled from plaintext SSIDs in return for pseudo-random SSIDs.

The prototype provisions an obfuscated discovery token to a station during association by embedding it within a Vendor Information Element in the Association Response frame. The token is
intended to be stored by the client and later used for token-based discovery.

Important: This is a research prototype. It is not a production-ready protocol and does not replace WPA/WPA2/WPA3.

---

## What this prototype does (current functionality)

### Token derivation (AP-side)
For each station, the AP derives a 16-byte token:

- `token = SHA256(BSSID || STA_MAC)[0..15]`

Properties:
- Per-station and per-BSSID
- Stable across reconnects (same STA MAC and BSSID)
- Independent of SSID visibility
- Truncated to 16 bytes.

### Token provisioning (on-air)
The token is sent in plaintext inside a Vendor IE appended to the Association Response.

IE format:
- Element ID: 221 (Vendor Specific)
- OUI: `00:11:22` (demo/private)
- Subtype: `0x01`
- Payload: 16 bytes (token)

### Security note

This token is not used for authentication or encryption. WPA/WPA2/WPA3 behaviour is unchanged. 

---

## What is NOT implemented yet (planned / future work)

- AP-side parsing/recognition of the token in Probe Requests
- STA-side support (wpa_supplicant) to:
  - store the token alongside the SSID
  - emit the token during active scanning (Probe Requests)

---

## Hardware / driver requirements (critical)

This project requires a **SoftMAC** (mac80211) Wi-Fi device for the AP.

Why:
- FullMAC devices generate management frames in firmware.
- This PoC modifies the **Association Response** in software, which requires SoftMAC/mac80211.

Tested working setup:
- Platform: Raspberry Pi 4 Model B
- AP interface: USB Wi‑Fi adapter (MediaTek MT7612U)
- Driver: `mt76x2u` (mac80211 / SoftMAC)

---

## Repository layout

- `hostapd-dev/hostap/hostapd/`:
  - `src/ap/wpa_auth.c`: token derivation (per-STA state-machine init)
  - `src/ap/wpa_auth_i.h`: token storage fields in `struct wpa_state_machine`
  - `src/ap/ieee802_11.c`: IE construction + Association Response injection

---

## Build

From the repo root:

```bash
git clone https://github.com/mtiluk/hostapd-privacy-demo
cd hostapd-dev/hostap/hostapd
cp defconfig .config
make clean
make -j4