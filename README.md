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
- `gainsborough_trilock/gainsborough_trilock/`: custom ESPHome component.
- `secrets_template.yaml`: example secrets file. Copy this to `secrets.yaml`.

The `tools/freestyle-gwasm-status.js` helper and `tools/freestyle-app-config.json`
are no longer needed and can be deleted — they existed only to pull Cognito
refresh tokens and property/BLE-MAC values out of the Freestyle cloud account,
which this project no longer talks to at all.

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

The lock exposes two useful BLE channels:

- **Command channel**: the normal encrypted lock/status protocol.
- **Sync channel**: the DoAi/DiAo transport the official phone app used to
  coordinate with Gainsborough's cloud. This component no longer uses it for
  that purpose (see below).

The ESPHome component uses the command channel for everything: locking,
unlocking, and status reporting. The lock's own status reply (decrypted
locally with your AES key) includes lock state, door-closed status, and
battery percentage — no cloud round-trip involved anywhere in that path.

### Periodic Refresh

Every minute, when BLE is connected and no other command is active, the
component sends a status request over the command channel. The decoded
response updates the lock state, door sensor, and battery sensor in Home
Assistant directly from the lock itself.

### Sync Channel / Diagnostics (local-only, optional)

The lower-level Sync-channel machinery (DoAi/DiAo transfer, phone-like
bootstrap, manual nonce injection) is still present in the component for BLE
protocol debugging, but it is fully local now — it never reaches out to any
Gainsborough server. The two functions that previously uploaded data to the
cloud (`sync_cloud_refresh`, `sync_upload_last_doai_payload`) are no-ops that
just log a warning if called; nothing in this component makes an outbound
HTTP/TLS request of any kind.

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