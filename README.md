# hostapd-privacy-demo

This is a proof of concept that network identification during discovery can be replaced with a obfuscated pseudo-random SSID rather than a plaintext SSID.

The prototype is implemented using a custom fork of the [hostapd](https://git.w1.fi/cgit/hostap/) repository. Where all additional code is implemented within functions and methods already written.  The prototype provisions an obfuscated discovery token to a station during association by embedding it within a Vendor Information Element in the Association Response frame. The token is intended to be stored by the client and later used for token-based discovery.

Important: This is a research prototype. It is not a production-ready protocol and does not replace WPA/WPA2/WPA3.


### What this prototype does (current functionality)

For each station, the AP derives a 16 byte token where `token = SHA256(BSSID || STA_MAC)[0..15]`.
  - Token is per station.
  - Stable across reconnects (assuming same MAC address)
  - Independent of SSID visibility
  - Truncated to 16 bytes.

The token is sent in plaintext within a Vendor IE appended to the association response.
  - Element ID: 221 (Vendor Specific)
  - OUI: `00:11:22` (demo/private)
  - Subtype: `0x01`
  - Payload: 16 bytes (token)

## What is NOT implemented yet (planned / future work)

  - AP parsing of the token in Probe Requests.
  - Modifying the client (wpa_supplicant) to store the token and emit the token during active scanning.

## Hardware / driver requirements (critical)

This project requires a **SoftMAC** (mac80211) Wi-Fi device for the AP.

Why:
- FullMAC devices generate management frames in firmware.
- This PoC modifies the **Association Response** in software, which requires SoftMAC/mac80211.

Tested working setup:
- Platform: Raspberry Pi 4 Model B
- AP interface: USB Wi‑Fi adapter (MediaTek MT7612U)
- Driver: `mt76x2u` (mac80211 / SoftMAC)

## Build

```bash
git clone https://github.com/mtiluk/hostapd-privacy-demo
cd hostapd-dev/hostap/hostapd
cp defconfig .config
make clean
make -j4
sudo ./hostapd ./hostapd.conf
```

  - Ensure the AP interface in hostapd.conf matches your device.

## Configuration

Minimal example hostapd.conf (edit interface/channel/security as needed):

```
interface=wlan1
driver=nl80211
ssid=DemoNet
hw_mode=g
channel=6

wpa=2
wpa_key_mgmt=WPA-PSK
wpa_passphrase=change-this-passphrase
rsn_pairwise=CCMP
```
