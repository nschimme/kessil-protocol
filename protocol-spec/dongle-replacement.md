# Replacing the WiFi Dongle and AP9X: feasibility

## Short answer

The design target should be a **shared replacement platform for both the legacy WiFi Dongle and AP9X**, with product-specific compatibility profiles over a shared light-control core. A single host should be able to provide the legacy TCP/8888 JSON interface and, where needed, the corresponding device's discovery and Apex/IoTa surfaces. It can also expose a clean independent API. Individual control still requires a compatible physical K-Link interface and an exclusive GID. No controller implementation is included in this publication.

That is an intended compatibility target, not a claim that one generic wire behavior can serve both products. The evidence is not sufficient for a drop-in replacement of every Dongle or AP9X feature or for a claim of compatibility with the Kessil app or Apex. There has been no end-to-end test on the user's chain; the two firmware targets differ in discovery, Apex transport, internal control hardware, and update procedures. AP9XE and WiFi Dongle XE are a separate generation with unresolved control APIs.

## Replacement scope

| Desired result | Feasibility from current evidence | Main remaining work |
|---|---|---|
| Python client controls lights through an original Dongle | Protocol is sufficiently specified to implement against firmware 1.5.13. No runnable client is included here. | Validate on hardware; map UI channel labels and observed fixture response. |
| Python host controls fixtures directly through a K-Link adapter | Core scan, identity, group assignment, and manual group-write frame builders are recovered. Kessil calls its male-to-male USB Type-C lead a “K-Link Cable” ([cable listing](https://kessil.com/products/accessories_Cables.php)); this identifies the connector, not USB data signaling. | Identify a safe electrical adapter/pinout and bus signaling, implement serial timing/half-duplex behavior, then capture and validate on the actual two-light chain. Kessil's public cable listing does not provide pin assignments or electrical levels. |
| Replacement exposes a custom local API | Straightforward in principle: wrap KID/GID inventory and per-light channel writes behind a small Python HTTP or JSON API. | Hardware adapter and live validation; decide how to persist groups and recover safely from partial writes. |
| Replacement emulates the legacy WiFi Dongle | Partially specified. Normal-mode TCP/8888 JSON, raw K-Link tunnel, UDP `POLO`, direct-AP defaults, and the APK's `dtype:201` shape are documented. | Validate response behavior and saved state; capture UDP discovery and full app workflows. The raw tunnel requires a compatible UART. |
| Replacement emulates the AP9X network controller | Partially specified. TCP/8888 `dtype` dispatch, AP9X UDP discovery (`19276`, `dtype:300/301`), plaintext Apex TCP, and AP9X update staging are documented. | Match AP9X-specific dispatcher/replies and dual-group behavior; validate AP9X discovery sender and capture app/Apex sessions. A physical replacement also needs an equivalent K-Link controller interface. |
| Replacement presents a Kessil endpoint to Neptune Apex | Partial implementation is designable from the analyzed binaries. Dongle UDP/encrypted TCP and AP9X UDP/plaintext TCP, IOTA schema, and AP9X `light.set` to K-Link group-write path are statically recovered. | Capture both Apex sessions, implement the different transports, and validate operation-to-light effects. Several IOTA operations have no recovered fixture effect. |
| Replacement implements all Dongle/AP9X functions | Not established. | Newer APK command IDs do not dispatch in analyzed Dongle 1.5.13 or AP9X 1.4.20; AP9X/Dongle firmware-update hosting and WiFi Dongle XE remain separate/incomplete scopes. |

## Target hardware profiles

Design this as a shared controller service with explicit **Dongle** and **AP9X** compatibility profiles, rather than assuming the two network firmwares are interchangeable:

| Surface | WiFi Dongle 1.5.13 | AP9X WiFi 1.4.20 |
|---|---|---|
| Legacy app command API | TCP `8888`, JSON `dtype`; firmware also exposes a raw `A9` K-Link UART tunnel on this port. | TCP `8888`, JSON `dtype`; the analyzed dispatcher has lamp query/find/info, group-knob (`dtype:3`), and KID-to-group (`dtype:4`) paths. Exact app-schema parity is unverified; a Dongle-style raw tunnel is not established. |
| Discovery | UDP `8888`, replies `POLO`. | UDP `19276`, firmware accepts `dtype:300` and replies `dtype:301` with serial/RSSI; the supplied APK has no recovered send call site. |
| Apex / IoTa | UDP and TCP `26977`; TCP IOTA JSON uses the recovered Salsa20/12 stream session. | UDP and TCP `26977`; TCP IOTA JSON is plaintext in the analyzed listener. |
| Light-control hardware | Linux/MIPS controller writes the K-Link UART directly. | Linux/MIPS WiFi module controls a separate STM32L4 K-Link MCU. AP9X has its own controller identity and firmware-update target. |
| Firmware update | FTP staging plus `dtype:208` and OpenWrt `sysupgrade`. | FTP staging plus `dtype:208` for the WiFi module; a separate KID-addressed YMODEM-style transfer updates the AP9X STM MCU. |

The common service core can normalize inventory and requests, then route them through profile-specific discovery, API serialization, persistence, Apex transport, and IOTA-to-K-Link conversion. The last boundary matters: even the legacy API's `dtype:3` write flag differs in the recovered code. Dongle requests use `enabled:1` for a live write, while AP9X's parser defaults `enabled` to `-1` and only sends the K-Link write when that field remains `-1`; explicitly sending `0` or `1` suppresses that call. Their IOTA paths also differ in selector-to-GID output branches, fallback values, and state handling. For example, AP9X has output branches for selectors 1–3; Dongle code iterates four slots and maps the fourth to GID 0, whose practical meaning is unresolved. AP9X discovery is also not the Dongle's `POLO` probe, and their Apex TCP transports cannot share a single wire codec. One AP9X-specific control point has no Dongle equivalent: if the replacement is expected to take a chain away from an AP9X that is acting as master, it must send the once-per-second K-Link command-`34` presence beacon so the AP9X hands control over; see [command `34` as a handover beacon](klink-protocol.md#command-34-as-a-handover-beacon). Updating original controller firmware is a separate feature from replacing its control/API behavior; the recovered update workflows target those devices' own system images, not the replacement host.

## AP9XE and WiFi Dongle XE are a separate generation

Do not fold AP9XE or WiFi Dongle XE into the legacy AP9X/Dongle profiles based on product names. Kessil's official [IoT setup page](https://kessil.com/IOTSetup/) documents a local device-network setup: join the `LBxxxxxxxx` network, use the reversed serial number (uppercase) as its default password, then open `http://192.168.1.1` or `http://LBxxxxxxxx.local`. The page says to scan/select the home WiFi network, then log into `iot.kessil.com` and link the fixture by scanning its package QR code. This establishes a setup workflow and local browser endpoint, but not the web routes, request schema, authentication/session behavior, cloud API, or device-control API.

Kessil's [legacy WiFi app manual](https://kessil.com/downloads/accessories/kessilwifi_UserManual/wifi_user_manual.html) explicitly says the Kessil Mobile App is not compatible with AP9XE or WiFi Dongle XE. The supplied v1.15.3 APK and analyzed AP9X 1.4.20/Dongle 1.5.13 firmware therefore do not specify these XE devices. No XE firmware image or update procedure is present in this workspace; the legacy FTP/`dtype:208` and old FOTA workflows must not be assumed to apply. Record the XE generation as its own target, with local setup known and control/update protocols unresolved.

## Smallest useful implementation

For direct control without reproducing the Kessil app, the simplest architecture is a Python process on a Linux host with a K-Link electrical adapter. It should:

1. Discover KIDs with raw command `18` and query identity with command `1C`.
2. Read or maintain the current KID-to-GID mapping; never assume a KID is a GID.
3. Put each independently controlled light in its own compatible group using command `1B` (or use the original Dongle's `dtype:4`).
4. Send command `1F` to a GID with its five channel bytes. The write applies to all fixtures on that GID, so unique groups are required for independent control.
5. Log requests and replies, validate frame length/checksum/terminator, and treat a timeout or ambiguous bridge response as an uncertain write requiring readback.

The frame formats are in [K-Link protocol](klink-protocol.md), and network request shapes are in [WiFi Dongle API](wifi-dongle-api.md). Electrical adapters, transport drivers, and runnable control tools are outside this publication.

### Physical interface for a direct controller

The [physical-layer observations](klink-protocol.md#physical-layer-differential-half-duplex-serial-on-the-usb-c-data-pins) show 115200 baud 8N1 serial on the USB-C D+/D- pins, differential and half-duplex. A direct controller therefore needs an RS-485-class transceiver (3.3 V logic) between a microcontroller UART and the port, with the transceiver's direction pin driven around each transmission, plus 5 V from the port if the controller is powered from it. Connect D+ to the transceiver's non-inverting input so that its receive output idles high. The Dongle repeats group writes about once per second; a replacement should do the same until it is shown that fixtures hold a level without repeats.

## Neptune Apex / IoTa endpoint

Treat Apex compatibility as a separate service surface alongside TCP/8888. The analyzed Dongle starts TCP and UDP listeners on port `26977` when Apex mode is enabled (`dtype:210`); its UDP endpoint handles IoTa discovery/time-related messages, and its TCP endpoint accepts encrypted IOTA JSON. A replacement can expose these listeners to the Apex while translating supported light operations to K-Link. The [Apex/IoTa reference](apex-iota-protocol.md#dongle-apex-tcp-encrypted-iota-json) summarizes transport differences, but omits cipher constants and handshake reconstruction. It does not provide a complete Dongle-compatible encrypted endpoint specification; live validation is still required.

The command bridge has a limited, target-specific semantic scope. AP9X `light.set` selectors 1–3 map to fixed GIDs 1, 2, and 26 and ultimately issue K-Link command `1F`; Dongle 1.5.13 uses a separate background writer with slots for GIDs 1, 2, 26, and 0. Both have fallback group-write paths, but their stored values and conversion behavior differ. These selectors are not KIDs, and each write affects all fixtures on its GID. Reproducing the original Dongle's Apex-mode setup also entails its type-to-GID reassignment side effect. The handler evidence and missing operation semantics are summarized in [the Apex operation coverage table](apex-iota-protocol.md#can-k-link-proxy-all-apex-operations).

The Neptune guide's supported topology is a constraint on the product behavior to emulate: same-type A360X fixtures may be chained to mirror one Apex-controlled schedule, while independent control through Apex requires a separate Dongle/Apex attachment per fixture. Do not equate this with the Kessil app's KID-to-exclusive-GID control path. A direct-K-Link replacement might offer more addressing than the official Apex workflow, but that would be an extension and needs its own compatibility tests.

Minimum work before claiming Apex compatibility is: capture the Apex's UDP discovery and TCP request sequence; reproduce and test the Dongle cipher session and message framing; implement only the confirmed IOTA operations; bridge those operations through validated K-Link groups; and verify effects on a fixture. `light.get`, `light.status`, and `light.capability` currently have no recovered live fixture-read path, and `light.flash` actuation is unknown. An Apex-facing prototype is feasible, while a complete behavior-compatible endpoint is not yet specified.

## Drop-in app emulator requirements

An app-compatible emulator would need at minimum:

- A Wi-Fi AP and DHCP configuration compatible with the legacy app's direct-connect flow, or a station-mode deployment on the same LAN. Firmware 1.5.13's direct AP address is `10.0.100.1`; the Kessil app manual also describes router mode.
- UDP port `8888` discovery behavior, including the `POLO` response, and TCP port `8888` JSON request handling. The app's TCP parser is not a robust stream protocol: firmware dispatches each `recv()` as one complete request, so a replacement should be more robust while preserving the observed request/response format.
- Legacy device identity, inventory, group, program, and status responses. Some fields are cached Dongle state, so the replacement must choose which state to synthesize and persist.
- The raw `A9` K-Link tunnel path if clients use it, alongside the JSON command path.
- The correct K-Link electrical interface and bus behavior. Static protocol recovery does not establish safe voltage levels, connector pin assignments, termination, or the exact two-port forwarding behavior.

The APK's `dtype:201` builder is recovered as `data` fields `SSID`, `PWD`, `ENC`, integer `BAND`, and integer `SAVE=1`; see the [WiFi setup request schema](wifi-dongle-api.md#connect-to-the-dongle-direct-ap-or-home-router). Accepted value ranges, the meaning of `BAND` and `SAVE`, and live router-mode behavior remain unverified. AP9X `dtype:300` has a firmware parser/reply but no corresponding request sender was found in the supplied APK. The app's newer multi-lamp command builders exceed both analyzed network firmware dispatchers. These make full app compatibility a separate reverse-engineering and test effort.

## Validation needed before relying on it

Start with read-only inventory and identity queries on a single fixture, then repeat with the two reported daisy-chained Tuna Sun fixtures. Capture both sides of the Dongle/fixture or inter-fixture link with an electrically suitable high-impedance analyzer. Only after confirming KIDs, group assignments, and channel mapping should a low-risk channel write be tested. Confirm that a write to one exclusive GID changes only its fixture and that a shared GID changes every member. A protocol-level success reply alone is not proof of emitted light output.

For a network/API emulator, capture a normal app session in direct AP mode and router mode, including discovery, initial inventory, group reassignment, manual changes, restart/reconnect, and error cases. For the Apex surface, capture UDP discovery and a complete TCP session from connection through response, including the first eight-byte cipher preface and at least one `light.set`. Validate each interface independently before describing the replacement as compatible. Keep firmware update emulation out of the first control prototype: the staging/update sequence can replace executable firmware and is not needed for manual control.
