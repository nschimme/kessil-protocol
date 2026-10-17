# Independent Kessil protocol research

Independent protocol research on Kessil lights through K-Link, the legacy WiFi Dongle/AP9X network interfaces, Neptune Apex/IoTa, and the Spectral Controller X USB updater.

**Unofficial project.** This work is not affiliated with, sponsored by, or endorsed by Kessil, DiCon Fiberoptics, or Neptune Systems. Product and company names are used only to identify the devices and interfaces discussed; their respective owners retain their marks.

**Research and safety notice.** The notes are provided for interoperability research, without warranty. Incorrect commands or firmware for the wrong model can disrupt operation or make equipment unusable. Use only with devices and software you are authorized to inspect or control. These notes are not instructions for installing firmware.

This publication contains protocol descriptions only; it does not distribute device-control/discovery utilities, firmware extraction code, firmware payloads, updater executables, or app packages. The [firmware analysis boundary](firmware/README.md) summarizes the source limitations without providing acquisition instructions.

Start with the [protocol index](protocol-spec/README.md). The current replacement design target is one controller service with separate legacy WiFi Dongle and AP9X compatibility profiles; AP9XE and WiFi Dongle XE remain a separate, unresolved generation.

The AP700 Home Assistant script is not included. Its discussion and source are available in the [Reef2Reef reverse-engineering thread](https://www.reef2reef.com/threads/kessil-ap700-reverse-engineered.1154371/). Before publishing, verify the staged files and repository history contain no vendor binaries or third-party code; ignore rules alone do not prevent accidental commits.

The findings are based mainly on static analysis. Hardware captures and firmware-update validation remain open; see [evidence and open questions](protocol-spec/protocol-evidence.md).

Original documentation is licensed under [CC BY 4.0](LICENSE). See [contribution guidance](CONTRIBUTING.md) and [source-rights limitations](firmware/README.md#source-rights-and-research-records) before adding material.
