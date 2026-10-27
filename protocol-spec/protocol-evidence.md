# Evidence, limits, and open protocol questions

These notes distinguish behavior inferred from software analysis, app request serialization, official product documentation, and live observation. Most findings come from static analysis. No live Kessil device capture or firmware update has been performed.

The project documents functional behavior and protocol fields in independently written prose. It does not publish vendor firmware, application packages, extracted binaries, decompiler output, or source-code excerpts. See [the firmware analysis boundary](../firmware/README.md).

## Open questions

1. Validate legacy Dongle and AP9X discovery request/reply behavior, including response segmentation and retries, with authorized devices.
2. Confirm the newer multi-lamp request schemas and whether the analyzed firmware generations support them.
3. Capture one-light and daisy-chained fixture sessions to validate enumeration, identity fields, group assignment, and the effects of commands that may change saved group state.
4. Validate the recovered Apex/IoTa message formats and transport behavior against an authorized, locally controlled device. Some fields and operations remain uncertain.
5. Treat AP9XE and WiFi Dongle XE as a separate generation. Official setup material documents provisioning, but its control API, authentication, firmware, and update path remain uncharacterized.
6. Capture a chain with an AP9X acting as master and a second controller (Spectral Controller X or a compatible replacement) sending the command-`34` presence beacon. The AP9X handover described in [K-Link protocol](klink-protocol.md#command-34-as-a-handover-beacon) has not been observed on a bus.

## Confidence and limitations

The protocol references describe observed or inferred packet structures and device behavior, but static analysis is not proof of runtime behavior. Treat model-specific details as provisional until independently confirmed on hardware. Update details are especially incomplete and must not be treated as instructions for installing firmware.

Official manuals and product pages are linked where they support product-level claims. App- and firmware-derived findings are summarized at the functional level; the source artifacts remain with their respective owners. No claim is made that the findings are exhaustive, current across product revisions, or endorsed by Kessil, DiCon Fiberoptics, or Neptune Systems.
