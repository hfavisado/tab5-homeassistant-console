# Tab5 Home Assistant Console

## Status: phase 1 — feasibility and architecture

This repository is a feasibility study, **not yet firmware**. There is no
build, flash, provisioning, configuration, or Home Assistant client to run.
The next step is a hardware proof on a real Tab5; do not use this document as
evidence that a particular software combination has been built or tested.

## Hardware and software investigation

The [official Tab5 documentation](https://docs.m5stack.com/en/core/Tab5)
could not be accessed from this environment. The following findings are based
on vendor source code, inspected on 2026-11-07:

| Area | Evidence and implications |
| --- | --- |
| Board | [Espressif's Tab5 BSP overview](https://github.com/espressif/esp-bsp/blob/67f41a169401d1f23e827188dd08fb351cb0febe/bsp/m5stack_tab5/README.md) describes the ESP32-P4, 16 MB flash, 32 MB PSRAM, ESP32-C6 wireless coprocessor, MIPI-DSI touchscreen and microSD slot. Its [display header](https://github.com/espressif/esp-bsp/blob/67f41a169401d1f23e827188dd08fb351cb0febe/bsp/m5stack_tab5/include/bsp/display.h) defines a native 720 × 1280 panel; landscape rendering and touch coordinates must be verified together. |
| Rust | The [esp-rs ESP-IDF template](https://github.com/esp-rs/esp-idf-template/blob/b814c2b968f84949707ed0a6ab72d262e3fbfbc4/README.md) lists `riscv32imafc-esp-espidf` for ESP32-P4. [esp-idf-sys](https://github.com/esp-rs/esp-idf/blob/64f829959cfe2771b1168d2790939fcc050372ab/esp-idf-sys/README.md) exposes ESP-IDF to Rust, but this is not evidence of an existing tested Rust Tab5 driver. The template describes these bindings as a community effort with limited hardware-in-the-loop testing. |
| Display and touch | The [BSP manifest](https://github.com/espressif/esp-bsp/blob/67f41a169401d1f23e827188dd08fb351cb0febe/bsp/m5stack_tab5/idf_component.yml) identifies version 1.3.2, ESP32-P4, ESP-IDF >= 5.4, LVGL port and both controller families. Its [display implementation](https://github.com/espressif/esp-bsp/blob/67f41a169401d1f23e827188dd08fb351cb0febe/bsp/m5stack_tab5/src/bsp_display.c) probes touch to select ILI9881C/GT911, ST7123 or ST7121; the ST712x firmware version selects the latter two. Unknown ST firmware is rejected and an absent controller can trigger an assertion. Use the BSP's initialization and touch workaround rather than guessing a panel from the product name. Unknown revisions need a recovery strategy and physical testing. |
| Wi-Fi | ESP32-P4 has a separate C6 radio; the [ESP-Hosted station example](https://github.com/espressif/esp-hosted-mcu/blob/9effd51fbc133e06aa4ff042c46d4f71483b13b0/examples/wifi/sta/README.md) demonstrates host-side Wi-Fi APIs over a coprocessor link, including P4/C6 and SDIO. Its [host board configuration](https://github.com/espressif/esp-hosted-mcu/blob/9effd51fbc133e06aa4ff042c46d4f71483b13b0/host/Kconfig.ext) and [SDIO settings](https://github.com/espressif/esp-hosted-mcu/blob/9effd51fbc133e06aa4ff042c46d4f71483b13b0/host/mcu/eh_host_mcu_transport/Kconfig.host.sdio) include Tab5 defaults; the [BSP enables Wi-Fi power](https://github.com/espressif/esp-bsp/blob/67f41a169401d1f23e827188dd08fb351cb0febe/bsp/m5stack_tab5/src/bsp_feature_en.c) through an expander. Host and C6 firmware must be compatible and use matching transport settings. |
| Storage | The BSP supports SD card mounting and lists a [display/SD example](https://github.com/espressif/esp-bsp/tree/67f41a169401d1f23e827188dd08fb351cb0febe/examples/display_sdcard). SD and hosted Wi-Fi operation together still need to be verified on the device. |

### Framework decision

Use **Rust with ESP-IDF**, with a narrow C/FFI boundary to the Espressif Tab5
BSP and its LVGL port, as the candidate architecture. ESP-IDF already has
the board-specific DSI, touch and coprocessor integration needed here; a
bare-metal Rust HAL would require independently verifying or implementing
those board-specific pieces. Keep unsafe FFI and LVGL ownership inside a
hardware/UI adapter, and keep application logic in host-testable Rust.

This is a **provisional selection**, not a proven build. Start evaluation
with ESP-IDF 5.4.4 and BSP 1.3.2; select and pin a mutually compatible Rust
toolchain, ESP-IDF, BSP, LVGL, ESP-Hosted, Wi-Fi-remote and C6 firmware
combination only after integration testing. The [esp-idf-sys component
integration documentation](https://github.com/esp-rs/esp-idf/blob/64f829959cfe2771b1168d2790939fcc050372ab/esp-idf-sys/BUILD-OPTIONS.md#extra-esp-idf-components)
describes a possible C component and bindings route, not a ready-made Tab5
wrapper. Record the lockfiles, component revisions, sdkconfig, partition
table and both chip images when a reproducible build is established.

## Proposed architecture

- **Hardware adapter:** BSP-managed display, touch, backlight, storage and
  Wi-Fi power; ESP-Hosted manages the P4–C6 link. LVGL is owned by one UI
  execution context, using the port's locking rules.
- **Configuration:** a bounded, versioned external JSON configuration on
  microSD, independent of credentials. Define pages, sections, layout,
  entity IDs, names, icons, explicit control/service overrides and
  visibility/enabled conditions. Domain defaults select reusable controls.
  Validate IDs, supported controls, bounds and references before replacing
  the active configuration; show errors and retain the last valid
  configuration or a safe navigation-only fallback. Reload without
  reflashing is a goal. JSON is a candidate for a small parser and host
  tooling; no schema or parser has been validated yet.
- **Home Assistant adapter:** runtime URL and token, REST state snapshots and
  service calls, plus a WebSocket state-change subscription. Authenticate
  with a long-lived token. Subscribe before taking a snapshot and reconcile
  events received during snapshot loading so updates are not lost. On
  reconnect, resubscribe and refresh; use bounded exponential backoff with
  jitter. Authentication failure should stop repeated token attempts and
  present a reprovisioning status. HTTPS/WSS must validate hostnames and
  certificates, including a deliberate trust-store and clock strategy;
  never silently fall back to insecure transport.
- **State and controls:** one application state store translates updates
  into UI models. Domain adapters map switches/lights to toggles and
  supported light attributes, covers to supported open/close/stop and
  position actions, climate to supported temperatures and HVAC modes, and
  sensors to read-only values and units. Treat missing attributes,
  `unknown` and `unavailable` as normal; never infer a capability solely
  from the domain. New domain adapters should not require changes to the
  transport or page renderer.
- **UI and interaction:** configuration generates pages and touch-friendly
  controls, with connection/loading/unavailable indicators and navigation
  available offline. UI callbacks enqueue bounded commands rather than
  performing network I/O; state changes return via a bounded event channel.
  Disable commands while disconnected and while the same action is pending.
  A timed-out service call has an uncertain outcome: report it and refresh
  state rather than automatically retrying a potentially non-idempotent
  action.

## Provisioning and security design

`HA_URL` and `HA_TOKEN` are inputs to a future **host provisioning tool**;
they are not device environment variables. The proposed flow is a
physically initiated, time-limited provisioning mode over USB, with the host
tool reading the environment without echoing values or placing them in
command-line arguments, logs, source, or firmware images. Device firmware
would store the URL and token separately from the editable UI configuration
in encrypted NVS, and allow explicit credential replacement/erasure. Wi-Fi
credentials need the same protection. Neither the tool nor the device-side
protocol exists yet; its authentication, recovery and host/USB threat model
must be reviewed before implementation.

[ESP-IDF NVS encryption](https://github.com/espressif/esp-idf/blob/master/docs/en/api-reference/storage/nvs_encryption.rst)
protects stored data at rest when correctly configured; it does not protect
secrets in a running or compromised device, on the provisioning host, or
from an attacker able to install malicious firmware. Evaluate secure boot,
flash encryption, debug access and production eFuse settings together on
the exact P4/IDF combination. These settings can be irreversible and affect
recovery; do not enable them blindly. USB provisioning exposes the token to
anyone with access to the provisioning host or channel during setup.
Unencrypted development devices do not meet the intended at-rest protection.
Keep credentials off removable SD cards and use a dedicated Home Assistant
account/token that can be revoked. Plain HTTP would expose that token on the
network; require HTTPS for deployment and explicitly warn if a future
development-only HTTP mode is offered.

## Implementation and acceptance gates

Each phase must produce a working, buildable increment before the next:

1. **Hardware foundation:** reproduce upstream display and hosted Wi-Fi
   examples on the actual Tab5 first; record board revision, C6 firmware,
   host/coprocessor versions, transport and power settings. Verify the
   detected panel, landscape drawing, touch alignment, backlight, SD and
   DHCP; test simultaneous SD and Wi-Fi. Then link the BSP into a minimal
   Rust application and show a diagnostic screen. Determine how to recover
   from unknown panels instead of accepting an assertion/reboot loop.
2. **Home Assistant:** independently test REST request construction and
   WebSocket authentication, updates, reconnect and invalid tokens with a
   host mock; verify TLS and service calls against a real instance.
3. **Configuration:** implement and document a versioned schema and sample;
   test parsing, validation, defaults, unsupported controls, fallback and
   reload with malformed and missing files.
4. **UI:** implement dynamic navigation, control adapters, capability-aware
   actions and state synchronization; test loading, offline and unavailable
   states and landscape touch on hardware.
5. **Reliability:** test AP/server interruptions, uncertain service calls,
   reconnection, memory use and extended operation. Document measured
   results, reproducible build/flash instructions and provisioning only
   after these paths work.

## Open risks

No Tab5 hardware, toolchain build, network integration or endurance test was
available in this phase. Panel detection, Rust/C linking, PSRAM and LVGL
buffer sizing, C6 flashing and host/firmware compatibility, SD coexistence,
TLS trust and clock provisioning all remain unverified. In particular,
vendor examples with other versions do not prove this proposed combination.
There is no claim of support for every hardware revision. If a board cannot
run the selected BSP or hosted link reliably, first report the detected
revision and reproduce the vendor example before choosing a different
version or a narrowly scoped board adapter.
