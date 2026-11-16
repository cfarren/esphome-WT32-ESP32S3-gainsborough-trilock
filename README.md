# ESPHome Gainsborough Freestyle Trilock BLE Bridge

This project provides an ESPHome component to bridge Gainsborough Freestyle
Trilock BLE locks to Home Assistant — entirely over local BLE, with
**no dependency on Gainsborough's Freestyle cloud service**.

This project is based on the reverse-engineering work done by mcchas:
[ble-freestyle-client](https://github.com/mcchas/ble-freestyle-client).

## Hardware Support

- Original target: WT32-ETH01 (Ethernet + BLE)
- Current target: ESP32-S3 (WiFi + BLE)

## Files

- `gainsborough_trilock_esp32s3.yaml`: ESPHome configuration for ESP32-S3.
- `gainsborough_trilock/`: custom ESPHome component (`gainsborough_trilock.h`/`.cpp`).
- `secrets_local_test_template.yaml`: example secrets file. Copy this to `secrets.yaml`.
- `freestyle-get-lock-key.js`: one-time helper that logs into your Freestyle
  cloud account and prints `trilock_aes_key` + `trilock_ble_mac` for
  `secrets.yaml`. This is the *only* thing this project still talks to the
  cloud for — getting the key out once, before the firmware ever runs. The
  firmware itself never calls it and has no cloud dependency at runtime.

## Setup

1. Fill in WiFi, ESPHome API/OTA values, and your lock's AES key + BLE MAC in
   `secrets.yaml` (see below).
2. Compile and flash:

   ```bash
   esphome run gainsborough_trilock_esp32s3.yaml
   ```

`secrets.yaml` is intentionally ignored by git. Do not commit your lock's AES
key or BLE MAC address.

Required secrets:

```yaml
trilock_aes_key: "<base64 32-byte BLE lock key>"
trilock_ble_mac: "aa:bb:cc:dd:ee:ff"
api_encryption_key: "<ESPHome API key>"
ota_password: "<ESPHome OTA password>"

wifi_ssid: "<ssid>"
wifi_password: "<password>"
```

That's the complete list — there are no Freestyle/Cognito cloud secrets
(`freestyle_refresh_token`, `freestyle_property_id`, `freestyle_cloud_ble_mac`,
`freestyle_client_id`, `freestyle_client_secret`) to obtain or store anymore.

The AES key is also editable at runtime from Home Assistant (the
"`<friendly_name>` - AES Key Override" text entity) without recompiling —
see "Runtime-Editable Settings" below.

## How It Works

The lock exposes two BLE channels:

- **Command channel**: the encrypted lock/status protocol. This is the only
  channel this project actually uses.
- **Sync channel**: the DoAi/DiAo transport the official phone app used to
  relay telemetry to Gainsborough's cloud. Not used by this project at all —
  see "Battery Percentage Is Not Available" below for why.

The ESPHome component uses the command channel for everything: locking,
unlocking, and status reporting. The lock's own status reply (decrypted
locally with your AES key) only ever contains the lock's current and desired
state — nothing else. No cloud round-trip is involved anywhere in this path.

### Periodic Refresh

Every 30 seconds, when BLE is connected and no other command is active, the
component sends a status request over the command channel and updates the
lock entity and the "Detailed Status" text sensor from the reply.

### Battery Percentage Is Not Available

This was investigated thoroughly and ruled out — not a missing feature, a
genuine limitation of the lock's local BLE protocol:

- The raw BLE advertisement only broadcasts static identity data; it was
  captured repeatedly over several minutes, including across lock/unlock
  cycles, and never changed.
- The lock has no standard Bluetooth Battery Service (confirmed by browsing
  its full GATT table) — only its one proprietary service/characteristic.
- The command channel's status reply has exactly two real fields: desired
  state and reported state. No battery or door field exists in that message.
- The Sync channel does queue encrypted messages that likely carry battery
  telemetry, but decrypting them requires the exact AES-GCM nonce the lock
  used when it originally encrypted that message — a value that only ever
  existed inside a past cloud-authenticated session and is never transmitted
  over BLE. This was confirmed experimentally: supplying an arbitrary
  self-chosen nonce causes the lock to transfer the message successfully,
  but decryption fails every time, since the ciphertext was sealed long
  before the request was ever made.

Short version: Gainsborough's official app doesn't show battery either,
because it has no local way to read it — the cloud is the only place this
value is actually computed/stored. Getting it back would mean reintroducing
real cloud authentication (AWS Cognito), which defeats the point of this
project. Rely on the lock's own physical low-battery indicator (beep/LED)
instead, or add a manual `input_boolean` helper in Home Assistant plus a
periodic reminder automation to check it.

## Diagnostic Sensors

Besides the lock entity and status text, the yaml exposes a handful of
diagnostic sensors: WiFi signal, BLE signal strength to the lock (from
passive advertisement scanning, since the BLE connection itself is only held
open for well under a second per transaction), ESP32 heap/loop-time health,
and a "Last Connected" timestamp. These are all standard ESPHome platforms
and need no cloud or custom protocol knowledge.

## Runtime-Editable Settings

The AES key can be changed from Home Assistant at any time — the
"`<friendly_name>` - AES Key Override" text entity — without a recompile or
reflash. This is useful if you ever need to update the key (e.g. after
re-extracting it from a re-paired lock). The new value is stored in flash and
survives reboots and OTA updates.

## ESP32-S3 WiFi + BLE Coexistence Notes

The ESP32-S3 is a single-core RISC-V chip with a shared 2.4 GHz radio for both
WiFi and BLE. This requires careful configuration.

### Framework

Uses `esp-idf` instead of Arduino for lower RAM usage and better radio
time-sharing.

### BLE Scan Parameters

The ESP32-S3 config uses a reduced scan window so WiFi is not starved while BLE
is active.

### WiFi Power Save

- `power_save_mode: LIGHT` allows the radio to yield time to BLE between WiFi beacon intervals.

### Boot Sequence

WiFi connects before BLE activity starts, reducing radio contention at boot.

### Coexistence SDK Options

- `CONFIG_ESP32_WIFI_SW_COEXIST_ENABLE` enables the coexistence scheduler.
- `CONFIG_SW_COEXIST_PREFERENCE_BALANCE` balances priority between WiFi and BLE.

### AES Encryption

The component uses the standard `mbedtls/gcm.h` API for authenticated AES-GCM
operations used by the direct BLE command protocol.

## Publishing Safety

Before publishing:

1. Confirm `secrets.yaml` is ignored and not tracked:

   ```bash
   git status --ignored --short
   ```

2. Search tracked files for accidental secrets:

   ```bash
   git grep -n -E "aes_key|ble_mac"
   ```

3. If this repository's history predates the cloud removal, it may still
   contain old Freestyle refresh tokens, property IDs, or Cognito client
   secrets from earlier commits. Do not publish that history as-is — start
   from a clean sanitized repository or rewrite history before pushing
   publicly.