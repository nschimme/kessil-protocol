# Kessil WiFi Dongle protocol: start here

This page is the short guide to the WiFi Dongle/AP9X control path recovered from an older app and firmware generation. It replaces the former all-in-one analysis; detailed evidence remains in the linked references. The analyzed baseline is Kessil WiFi app 1.15.3 with Dongle firmware 1.5.13 and AP9X WiFi firmware 1.4.20. Static analysis is extensive, but no live Kessil device capture has validated the packet exchanges.

Kessil currently lists the WiFi Dongle (part KSAWD01) on its [accessories page](https://kessil.com/products/accessories_WifiDongle.php), with setup and app guides on its [downloads page](https://kessil.com/support/downloads.php). The product is not listed there as discontinued; AP700 is listed separately among discontinued/legacy products. “Legacy” below describes the analyzed app/protocol generation where stated, not the current product status of the WiFi Dongle. The current listing does not establish that retail units use firmware 1.5.13 or the exact behavior documented here.

## Individual light control

The verified-by-firmware control model is group-based: on TCP port `8888`, assign a lamp KID to an exclusive GID with `dtype:4`, then write that GID’s channels with `dtype:3`. A GID write affects every lamp assigned to it, so the group must contain only the target fixture for independent control. Use cached inventory (`dtype:114`) cautiously; scan/query paths can change saved group assignments. The wire-level K-Link frame reference explains how the GID write reaches the shared bus.

See [the minimal one-light API sequence](wifi-dongle-api.md#minimal-legacy-api-sequence-for-one-light-control), [request/response schemas](wifi-dongle-api.md#request-schemas-recovered-from-the-android-app), and [scan/query side effects](klink-protocol.md#scan-side-effects-dtype0).

## What is covered

- [Network API and message schemas](wifi-dongle-api.md): direct access-point (`10.0.100.1` in bundled 1.5.13) and router modes, IP discovery, endpoints, JSON framing, control commands, app command IDs, and discovery behavior.
- [K-Link frames and daisy-chain behavior](klink-protocol.md): raw tunnel, UART frame layouts, KID/GID addressing, and the reported two-light topology.
- [Apex / IoTa](apex-iota-protocol.md): the separate port-26977 service and its Dongle/AP9X transport differences.
- [Firmware updates](firmware-update.md): a high-level summary of the separate product update interfaces and their limits.
- [Evidence and open questions](protocol-evidence.md): analysis inputs, confidence boundaries, and remaining work.
- [Controller X USB updater and UART](usb-protocol.md) and [AP700 legacy protocol](ap700-protocol.md).

## Version boundary

The supplied app and images describe the legacy WiFi Dongle/AP9X generation. Kessil's [WiFi app manual](https://kessil.com/downloads/accessories/kessilwifi_UserManual/wifi_user_manual.html) says the legacy app is incompatible with AP9XE and WiFi Dongle XE. Kessil's [IoT setup guide](https://kessil.com/IOTSetup/) directs those products to join their device WiFi (`LBxxxxxxxx`, with the serial reversed in uppercase as the default password), then use `http://192.168.1.1` or `http://<serial>.local` before linking the fixture through the Kessil IoT service and package QR code. Since those local URLs use HTTP, TLS is not part of the documented browser-to-device setup step. The guide does not specify local request fields, cloud control routes, or firmware-update transport; this newer path is not represented by the supplied APK/images.

Kessil's [WiFi Dongle product page](https://kessil.com/products/accessories_WifiDongle.php) lists A360XE Tuna Blue and Tuna Sun alongside A360X and A500X. Its [A360XE product page](https://kessil.com/products/freshwater_A360XE.php) says K-Link can link A360X, A360XE, A500X, and APX units while retaining individual lamp control and independent programs for groups. These are product-level compatibility claims, not evidence that the legacy `dtype` or K-Link schemas recovered here apply to A360XE firmware; “APX” is preserved as Kessil's label without assuming it means AP9X or AP9XE.

AP700 has a separate TCP control path on port 8899 and an unrelated USB HID updater ([AP700 protocol notes](ap700-protocol.md)). It is an EOL reference, not a primary target for the current K-Link analysis.
