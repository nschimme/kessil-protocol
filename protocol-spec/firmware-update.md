# Firmware update interfaces

The analyzed product generations use different update paths. These notes summarize the separation between those interfaces; they are not update instructions.

| Product | High-level interface | Evidence boundary |
|---|---|---|
| Legacy WiFi Dongle / AP9X network module | Network-based staging and a device-specific update request | Inferred from software analysis; no update attempted |
| AP9X controller module | Separate controller-to-module transfer path | Inferred from software analysis; no serial capture or update attempted |
| Controller X | Host-to-controller USB HID updater | Inferred from updater and firmware analysis; no live USB capture or update attempted |
| AP700 | Product-specific legacy updater | Not analyzed for this publication; see the [AP700 reference](ap700-protocol.md) |
| AP9XE / WiFi Dongle XE and light fixtures | Not established | No applicable image or verified update path in this analysis |

An extracted image, a version label, or a checksum does not establish that firmware is authentic, current, compatible with a particular device, or safe to install. The repository does not distribute firmware or provide acquisition and flashing instructions. Use the manufacturer's supported update process for the exact product and consult [the evidence limits](protocol-evidence.md) for what remains unverified.
