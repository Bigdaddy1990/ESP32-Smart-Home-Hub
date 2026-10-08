# ESPHome 2026.9 OTA migration

The installed firmware version is not yet verified. Keep the existing
`espsmarthomehub,yaml` as the password bridge until the first update succeeds.

1. Build and upload `espsmarthomehub,yaml` with ESPHome 2026.9.1. Keep
   `ota_password` and `api_encryption_key` unchanged. Confirm the device boots
   and reconnects to Home Assistant before proceeding.
2. Build and upload `espsmarthomehub-encrypted-ota.yaml`. Its package includes
   the same hub configuration and replaces OTA authentication. Encrypted
   OTA uses the existing API encryption key. The unencrypted Web OTA endpoint
   is removed from this final configuration.
3. Verify another encrypted OTA update, API reconnection, and recovery through
   Safe Mode / USB. Keep the password-bridge configuration for rollback
   until these checks succeed.

Do the first update before ESPHome 2027.3 removes the older unencrypted path.
Do not use the final configuration for an unverified device still running
pre-2026.9 firmware: removing the password before the first update fails.

After upgrading Home Assistant to 2026.10 stable, restart HA while the ESP is
already connected. Each `platform: homeassistant` sensor must receive its initial
state when the source entity appears, without requiring a new state change.
This runtime fix requires no sensor YAML change.

Sources:
- https://esphome.io/blog/2026/09/16/esphome-2026-9/
- https://esphome.io/components/ota/esphome/
- https://www.home-assistant.io/blog/2026/10/07/release-202610/

Configuration validation and compilation do not prove the live firmware version,
successful uploads, reconnection, or recovery. Record those device checks after
execution. ESPHome 2026.10 remains a beta and is not the production build target.
