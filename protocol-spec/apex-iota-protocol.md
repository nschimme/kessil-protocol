# Apex / IoTa network protocol

This is a separate integration path from the legacy Kessil JSON API on TCP port 8888. The findings are static analysis of Dongle 1.5.13 and AP9X WiFi 1.4.20; neither plaintext nor encrypted traffic has been captured live.

## Apex integration service

The separate port `0x6961` (26977) is an Apex integration service. Dongle command `dType:210` enables Apex mode by creating `/etc/.apex`, setting `APEX_FLAG`, and starting `ApexUDPServer` and `ApexTCPServer` threads. Disabling the mode removes that marker, clears Apex data, and closes both server sockets. This distinguishes port 26977 from the app's ordinary JSON command channel on TCP port 8888.

Enabling Apex mode also changes the K-Link group assignment of every lamp in the Dongle's current inventory: the handler calls `SetGid` for each lamp with a known type, writes each assignment, saves the updated lamp records, and recalculates group lamp counts. The recovered mapping is:

| Firmware lamp `type` | GID assigned when enabling Apex mode | `light.set.device` selector using that GID |
|---:|---:|---:|
| 1 | 1 | 1 |
| 2 | 2 | 2 |
| 3 | 26 (`0x1A`) | 3 |

This firmware behavior explains why the IOTA selector-to-GID mapping is fixed and model-class-oriented. It also means enabling Apex can replace prior KID-to-GID assignments for inventoried lamps of these types; read back lamp/group state after changing Apex mode. This side effect is statically recovered from Dongle 1.5.13, not verified on hardware. It is separate from using `dtype:4` to assign one lamp to an exclusive GID through the regular app API.

## Apex Ready and IoTa context

Kessil's current downloads page lists a **Neptune APEX IOTA User Guide** under both AP9X and WiFi Dongle support. That establishes a documented Kessil–Neptune integration path. The guide describes IoTa as Neptune's Internet of Things: Aquarium initiative for Apex control of internet-connected equipment. Kessil's older Tuna Blue brochure also carries an **APEX READY** badge, so that phrase is compatibility-program branding across product generations, not the name of one wire protocol. The later IoTa/Apex path is the relevant context for both firmware generations' port-26977 services. In the recovered binaries, Apex mode is opt-in (`dType:210`), TCP and UDP listeners share port 26977, and the TCP side dispatches nested IOTA JSON. The transport differs by target: Dongle 1.5.13 encrypts the TCP JSON stream with a Salsa20/12-derived session, while AP9X WiFi 1.4.20 parses plaintext JSON. The public support listing and guide establish the product integration, while binary analysis supplies the message-level details recorded here. The public sources do not document the business terms of any Kessil–Neptune agreement, nor establish that all Kessil products, AP700 TCP commands, or K-Link UART traffic use the IOTA format.

The Neptune/Kessil guide (revision 1.0, 2021-08-17) applies to A360X Tuna Sun/Blue and AP9X, not AP700 or every Kessil fixture. It says directly controlled A360X Tuna Sun/Blue fixtures use a WiFi Dongle on the same network as the Apex. Additional same-type A360X fixtures can be K-Link daisy-chained and mirror the Apex-controlled light's schedule; for independent A360X control **through Apex**, each fixture needs its own Dongle and Apex attachment. The guide also says not to mix A360X and AP9X, or Tuna Sun and Tuna Blue, in one K-Link chain. Kessil lights do not require a user-provided password when attaching them to Apex. The daemon contains an `iotaPassword` string, but its purpose in the TCP path has not been established. This Apex-specific limitation should not be generalized to the Kessil app's own grouping workflow documented below. The guide does not specify the K-Link electrical or packet-level forwarding mechanism.

Sources: [Kessil downloads and support listing](https://kessil.com/support/downloads.php) (includes the Neptune APEX IOTA guide under AP9X and WiFi Dongle); [Neptune IoTa–Kessil A360X/AP9X user guide](https://www.neptunesystems.com/downloads/docs/IoTa-Kessil_A360X_and_AP9X_User_Guide.pdf); [Kessil Tuna Blue brochure](https://kessil.com/downloads/A160/Saltwater_brochure.pdf) (legacy APEX READY badge); [Kessil WiFi app manual](https://kessil.com/downloads/accessories/kessilwifi_UserManual/wifi_user_manual.html) (lamp grouping, chain size, and FOTA guidance).

The Apex UDP server binds `0.0.0.0:26977` and calls `recvfrom` with a 4096-byte buffer. `IOTA_IsValidHeader` checks the ASCII magic `iota` at bytes 0–3, requires request version `1` at byte 8, accepts a type below 3 at byte 9, and requires the big-endian 16-bit length at bytes 10–11 to be below 513. The handler processes types 0 and 1; type 2 passes header validation but is ignored. Bytes 4–7 and the precise meaning of the length field are unresolved; the validator only enforces that its big-endian value is below 513 and has no lower-bound or datagram-length consistency check. Header type 0 treats datagram bytes 14–19 as a MAC address in its log format and consumes bytes 20–23 as a big-endian 32-bit value logged as `Time sync`; it stores that raw value in shared Apex state together with the daemon's current timestamp. The MAC label is explicit in the firmware log string, but the MAC's role is not recovered. The time value's units, epoch/offset interpretation, and downstream use are unknown; this handler stores it but does not call `settimeofday`. Bytes 12–13 also remain unknown. Because the receive path does not pass the received datagram length into the validator or branch handler, it does not reject a short type-0 datagram before reading through offset 23. Provide at least 24 bytes to supply all consumed fields; this is a static read-footprint recommendation, not a validated protocol minimum. Type 1 returns a 48-byte response to the sender. The response buffer is zeroed first, copies request bytes 0–7, then sets the response header to version `2`, type `2`, and big-endian length 48. Thus the recovered validator's accepted request version differs from the version emitted in this response. The statically constructed layout is:

| Response offsets | Bytes/contents | Evidence and limit |
|---:|---|---|
| 0–7 | Copied from request | Header prefix, including magic and bytes 4–7 |
| 8–11 | Version `02`, type `02`, length `00 30` | Length is encoded big-endian |
| 12–13 | `00 00` | Zero from the initial buffer clear |
| 14–19 | Six bytes written by `kd_ifc_get_mac` | Interface MAC; selected interface depends on the daemon's Wi-Fi mode |
| 20–21 | `00 22` | Fixed bytes; field meaning unknown |
| 22–23 | `00 02` | Fixed bytes; field meaning unknown |
| 24–25 | `00 02` | Fixed bytes; field meaning unknown |
| 26–41 | AP9X serial string copied by `strcpy` into a 16-byte field | Buffer was initially zeroed; the code does not bound the copy, and no live response has validated serial length or padding |
| 42–47 | `00 01 00 00 69 61` | Fixed tail bytes; field meaning unknown |

The response is sent back to the datagram source on port 26977. This layout comes from AP9X WiFi firmware 1.4.20's `ApexUDPServer`; its byte values and offsets are static findings, not a live capture.

The AP9X WiFi Apex TCP listener uses port `26977` and accepts plaintext JSON. Its parser accumulates data across reads and uses brace counting to detect complete objects. This framing is limited and has not been tested against a live device. The Dongle's corresponding Apex TCP interface uses an encrypted JSON stream; the implementation details are omitted here, and interoperability has not been validated.

## Dongle Apex TCP: encrypted IOTA JSON

When enabled, the Dongle also listens on TCP port `26977`. Unlike AP9X, it encrypts the IOTA JSON stream. The recovered implementation has fragile message framing, and no live connection has been captured. Cipher constants, handshake reconstruction, and binary-level implementation details are intentionally not included in this publication.

After writing a response, the thread clears its input length and returns to `recv` on the same accepted socket; it closes that socket when `recv` returns zero or an error. The JSON response serializer appends CRLF to its plaintext JSON, then encrypts the complete JSON-plus-CRLF byte sequence with the output stream context before writing it. Thus the CRLF is a decrypted framing marker, not visible on the Dongle wire. A client should decrypt the response stream and wait for the plaintext CRLF before sending the next request. Concatenated requests are not a documented batch format: if TCP coalesces them into one `recv`, the simple brace counter can hand multiple root objects to one parser call. This conservative sequencing follows the recovered implementation; it has not been validated by a live capture.

The AP9X TCP parser expects nested JSON objects with a top-level namespace and command key: namespaces are `general`, `light`, and `pump`. The light command table is `get`, `set`, `flash`, `status`, `fallback`, and `capability`; each command's value must be an object parsed by a command-specific field table. For example, the structural form is `{"light":{"get":{...}}}`. The required field names for each light operation are listed below, while nested value encodings and full side effects still need tracing. AP9X receives raw JSON and sends JSON plus CRLF. Dongle 1.5.13 uses the same recovered IOTA command structure behind its Salsa20/12 stream handling; its response serializer also appends plaintext CRLF before encrypting the JSON response. Live interoperability remains unverified.

The command field tables expose these Apex light-object keys. The `IOTA_ParseObject` helper checks the object member count against the listed count and matches each key against the corresponding table; nested value encodings and the full side effects are not all recovered yet. In particular, `light.flash` is a timed lighting-effect command with `device`, `intensity`, `start`, `count`, `duration`, and `colors` fields; despite its name, it is not a firmware-flash request.

| Nested object | Required member count | Field names |
|---|---:|---|
| `light.get` | 2 | `device`, `data` |
| `light.set` | 5 | `device`, `time`, `ramp`, `start`, `end` |
| `light.flash` | 6 | `device`, `intensity`, `start`, `count`, `duration`, `colors` |
| `light.status` | 5 | `intensity`, `temperature`, `fan`, `status`, `colors` |
| `light.fallback` | 3 | `device`, `intensity`, `colors` |
| `light.capability` | 4 | `temperature`, `fan`, `flash`, `colors` |

The nested field parsers reveal more of the input shape:

| Value | Recovered encoding |
|---|---|
| `device` | 16-bit integer selector; the `light.set` handler accepts selectors 1–4. In AP9X WiFi 1.4.20, the background output thread maps selectors 1–3 to K-Link GIDs 1, 2, and 26; selector 4 has no matching group-write branch. It is not a KID. |
| `light.get.data` | String; the handler recognizes `"capability"` and `"status"`. Other strings take its unsupported-request path. |
| `time`, `ramp`, `light.flash.start`, `intensity`, `count` | Integer fields; `count` and `device` use the 16-bit parser, while time/ramp/flash-start/intensity use 32-bit parsers. |
| `light.set.start` / `light.set.end` | Period objects with `intensity` and `colors`; `colors` is an integer array with five entries for selector 1 and four for other supported selectors. |
| `light.flash.duration` | Object with integer `on` and `off` members. |
| `light.flash.colors`, `light.status.colors` | Four-entry integer arrays. |
| `light.fallback.colors` | Integer array; the fallback handler reads four entries for selector 1 and three for the other accepted selectors. |
| `light.status.status` | Nested object with exactly two members: `code` (32-bit integer) and `text` (string). The shared `IOTA_Status_parse` metadata table contains those two key strings in that order, at structure offsets 0 and 4. |
| Capability color entries | Objects with `name`, `rgb` (JSON string of exactly six hexadecimal characters, decoded as three bytes), and `max` (JSON number parsed into 32-bit storage at one decimal place of precision). The parser rejects non-hex characters or any RGB string length other than six; the output formatter renders the same three bytes as six hexadecimal digits. |

The AP9X `light.get` handler is a metadata/template lookup, not a live light query. For `data:"capability"`, it selects a built-in color/capability profile from the `device` selector; for `data:"status"`, it builds the corresponding status template. The handler calls no K-Link read routine, so neither response is evidence of measured fixture output or current lamp state. This differs from the port-8888 `dtype:1` request, which queries an individual lamp by KID, and from the separately named `light.status` IOTA command. For example, the JSON body expected by the parser for a capability lookup is:

```json
{"light":{"get":{"device":1,"data":"capability"}}}
```

This is the JSON body the AP9X parser expects. The corresponding Dongle body is carried in the decrypted JSON stream described above; neither form has been validated by live capture.

The following parser-shaped `light.set` example illustrates the fixed-group output path for selector 2. It supplies the required five top-level and two period fields, and four colors for each period as required by selector 2:

```json
{"light":{"set":{"device":2,"time":0,"ramp":0,"start":{"intensity":0,"colors":[0,0,0,0]},"end":{"intensity":0,"colors":[0,0,0,0]}}}}
```

In the recovered AP9X WiFi 1.4.20 path, the handler stores the selector-2 start period and sets the command-received flag. The lighting thread converts these all-zero start values to five zero channel bytes and calls the group writer for GID 2. With the bundled `power_saving=0` K-Link setting, the resulting frame is:

```text
A9 0B 00 00 1F 00 00 00 00 00 02 00 D4 5C
```

This is static code-derived behavior, not a captured request/reply. The request controls a GID, so every fixture assigned to GID 2 is in scope; it does not select one fixture by KID. The example also does not establish how `time`, `ramp`, or the `end` period affect later output.

### Dongle versus AP9X light-output semantics

The shared IOTA command names and parser shape do not imply identical light behavior. In Dongle 1.5.13, `IOTA_Light_Set_dump` stores values in per-selector state and saves Apex state; it does not write K-Link directly. The separate `program_handler` periodically reads up to four active slots while Apex mode is enabled, selects stored schedule or fallback channel values, then calls `SetKnobGroup` and `WriteData`. Its slot-to-GID mapping is 1→1, 2→2, 3→26, and 4→0. This is a static finding from the Dongle binary; selector 4/GID 0 should be treated as an unresolved legacy branch, not as evidence that GID 0 is a valid or useful independent-light address.

The Dongle's `IOTA_Light_Fallback_dump` also persists fallback values for selectors 1–4. Its recovered conversion stores zero channel values when intensity is zero; when nonzero, it stores a profile with `UI1=255`, `UI2=127` (selector 3 uses `120`), and `UI3`–`UI5=100`. The background writer uses fallback values when the slot's set-state field is clear and schedule-derived values otherwise. The state-field interpretation, precise schedule conversion, fourth slot behavior, timing/ramp behavior, and actual emitted fixture response have not been validated on hardware. These Dongle values differ from the AP9X fallback path documented above, so an emulator must preserve target-specific conversion and state behavior rather than reusing AP9X formulas.

| Behavior | AP9X WiFi 1.4.20 | WiFi Dongle 1.5.13 |
|---|---|---|
| `light.set` path | Stores state; background thread outputs selectors 1–3 to GIDs 1, 2, 26. | Stores state; `program_handler` outputs up to four active slots to GIDs 1, 2, 26, 0. |
| `light.fallback` state/output | Conditional output for selectors 1–3; values become `(127,127,0,0,0)` for nonzero intensity on selectors 1–2, `(127,120,0,0,0)` on selector 3, or all zero. | Persists a different profile: nonzero intensity stores `(255,127,100,100,100)` for selectors 1–2, `(255,120,100,100,100)` for selector 3; zero stores all zero. Writer chooses fallback when its set-state field is clear. |
| Transport into IOTA TCP | Plaintext JSON. | Salsa20/12-derived encrypted JSON stream. |

All output details in this comparison are code-derived. No live Apex session or physical output has been captured, and matching API syntax is not proof of drop-in compatibility.

For `data:"capability"`, the device selector chooses one of three compiled profiles. The RGB values below are the three bytes passed into each capability entry; each entry is initialized with internal `max` storage of `1000` (the capability parser uses one decimal place, corresponding to `100.0`). These are advertised profile values, not a calibrated mapping from K-Link `UI1`–`UI5` to emitted spectra.

| `device` selector | Capability entries (`name`: `rgb`) |
|---:|---|
| 1 | `BLUE`: `0000FF`; `WHITE`: `C6CCFF`; `VI`: `9166A6`; `RED`: `E61E2C`; `GRN`: `1F9A3A` |
| 2 | `6000K`: `FFFAC8`; `9000K`: `C6CCFF`; `RED`: `E61E2C`; `AMBER`: `F5761A` |
| 3 | `PURPLE-BLUE`: `7B00FF`; `RED`: `FF0000`; `WHITE`: `828282`; `GRN`: `1F9A3A` |

The separate `light.status` handler is also not a live K-Link read. Its recovered handler builds a JSON `status` object from the five parsed request members: three one-decimal numeric fields (`intensity`, `temperature`, `fan`), the nested `status` (`code`, `text`) object, and the four-entry `colors` array. The function contains no K-Link read or write call; it serializes the supplied status fields into the handler result. This proves the returned shape and local behavior, but not the intended peer-to-peer role or whether another path updates these fields before a request arrives.

`light.fallback` is a persisted setting that the background lighting thread can apply when `b_received_set_command` is clear. The handler writes the 0x150-byte per-selector state to `/usr/share/k-daemon/apex_info.bin` under a mutex; the IOTA startup path reads that file and initializes the set-command flag to clear. The recovered `light.set` handler sets this flag, and no later clear path was found in the analyzed firmware. Thus a fallback request can update the persisted file without being applied while the flag remains set; its live effect depends on the thread entering the fallback branch. For selectors 1–3, the handler treats `intensity` only as zero versus nonzero: zero stores three zero values, while any nonzero value stores three `500` values, then it zeros the next three values. It parses the required `colors` array but the recovered handler does not read it. While the flag is clear, the output thread converts this state and sends command `1F` to GIDs 1, 2, and 26 for selectors 1, 2, and 3. A nonzero fallback therefore produces `(UI1, UI2, UI3, UI4, UI5) = (127,127,0,0,0)` for selectors 1–2 and `(127,120,0,0,0)` for selector 3; zero intensity produces five zero channel bytes. These are firmware-computed group values, not physical-output measurements. Selector 4 has no corresponding fallback writer branch. For example, selector 2's JSON request body is `{"light":{"fallback":{"device":2,"intensity":1,"colors":[0,0,0]}}}`; despite the `colors` member being required by the parser, those values do not affect the recovered fallback write.

This handler-level summary distinguishes light-output paths from JSON-shaped IOTA responses:

| IOTA operation | Recovered effect in AP9X WiFi 1.4.20 | Reaches K-Link output? |
|---|---|---|
| `light.set` | Stores a selector schedule and sets `b_received_set_command`; the background thread consumes it. | Yes, through command `1F` for selectors 1–3. |
| `light.fallback` | Persists zero/nonzero fallback state; the thread uses it only while the set-command flag is clear. | Conditionally, through command `1F` for selectors 1–3. |
| `light.get` | Builds a selector-specific status or capability template. | No K-Link read was found in the handler. |
| `light.status` | Serializes the supplied status fields into JSON. | No K-Link call was found in the handler. |
| `light.flash` | Serializes the supplied flash fields into JSON; the physical effect is unresolved. | No K-Link call was found in the handler. |
| `light.capability` | Serializes the supplied capability flags and colors into JSON. | No K-Link call was found in the handler. |

This summary describes the recovered handlers and their known call paths, not the intended external semantics of every Apex/IOTA message. In particular, the operation names alone do not establish that `flash`, `status`, or `capability` actuates or measures a fixture.

### Can K-Link proxy all Apex operations?

No. The recovered material is enough to implement a **limited output bridge** for AP9X `light.set` selectors 1–3: parse the IOTA schedule fields, reproduce the firmware's partially recovered channel conversion, then send command `1F` to the corresponding fixed GID (1, 2, or 26). `light.fallback` also has a recovered conditional command-`1F` output path, but its persisted state, zero/nonzero behavior, and precedence relative to `light.set` must be reproduced. These are GID-level writes: they affect every fixture in the group, not a selected KID. The schedule/ramp behavior and physical channel calibration remain unverified.

This is not a complete Apex-to-K-Link proxy specification. `light.get`, `light.status`, `light.capability`, and `light.flash` have no recovered fixture read/write path in the analyzed AP9X handlers; `light.flash`'s physical effect is unresolved. Apex discovery/UDP exchange, Dongle Apex-mode setup, and the Dongle's encrypted IOTA TCP transport are also separate from K-Link UART frames. The raw TCP-8888 tunnel is a second, independent way to forward K-Link frames, not a translation layer for IOTA requests. Its documented low-level coverage currently includes KID range discovery (`18`), KID identity (`1C`), KID-to-GID assignment (`1B`), and group channel output (`1F`); writes through that raw path bypass the JSON API's group checks and cached-state updates.

| Goal | Current evidence/status |
|---|---|
| Reproduce AP9X `light.set` output for selectors 1–3 | Static parser, conversion, fixed GID, and command-`1F` path recovered; no live Apex capture or fixture validation. |
| Reproduce `light.fallback` | Static conditional output path recovered; state precedence and runtime behavior need validation. |
| Address one fixture independently | K-Link command `1B` can assign a KID to a GID, then `1F` writes that GID; safe exclusivity, persistence, and fixture acceptance are not established by a successful tunnel write. |
| Proxy every Apex/IOTA operation | Not supported by current findings; several operations are serializers/templates, and `flash` actuation is unknown. |

Accordingly, describe an implementation as a target-specific partial schedule-to-K-Link bridge or a raw K-Link controller, depending on which interface it implements. A shared replacement can support both network profiles, but it must retain their different TCP transports, output mappings, fallback conversions, and state handling. Do not claim full Neptune Apex compatibility or a complete IOTA proxy without recovering missing operation semantics and validating each profile against captured traffic and hardware.

These are Apex/IOTA schemas on the port-26977 service. The separate port-8888 API uses top-level numeric `dtype` values and is the APK's normal dongle-control channel; do not substitute one envelope for the other. The `light.set` callback itself stores the parsed schedule in a per-selector state slot and marks a set command as received; it does not directly write UART. The separate `conn_iota_lighting_thread` consumes those slots and calls the K-Link group writer (command `1F`) for these selector/GID pairs:

| `light.set.device` selector | K-Link GID written by the lighting thread |
|---:|---:|
| 1 | 1 |
| 2 | 2 |
| 3 | 26 (`0x1A`) |
| 4 | No matching group-write branch was found in the analyzed firmware |

Thus the Apex selector does affect K-Link output for selectors 1–3, but it is a fixed schedule/group selector, not a KID or a freely chosen physical-lamp address. Those writes affect every fixture assigned to the corresponding GID. This is not the per-fixture `KID`→exclusive-`GID` workflow exposed by the port-8888 API, and the selector-4 behavior is unresolved. These call paths are static findings; no live Apex exchange has been captured.

The lighting thread also exposes part of the `light.set` value conversion. In its `b_received_set_command` branch it reads the stored **start** period's intensity and colors, then calls the group writer with integer channel values. For selectors 1 and 2, the visible arithmetic first uses integer division for the inputs: `UI1 = trunc((intensity / 10) × 2.55)` and `UI2 = trunc((50 − colors[0] / 20 + colors[1] / 20) × 2.55)`; selector 3 uses the same expressions except the `UI2` multiplier is `2.4`. Selector 1 derives `UI3`–`UI5` as `trunc((intensity / 1000.0) × (colors[n] / 10))` for `n=2..4`, where `colors[n] / 10` is integer division. Selectors 2 and 3 send zero for `UI3` and derive `UI4`–`UI5` from `colors[2..3]` with the same calculation. The decompiled code applies a minimum value of 1 to those derived channels when the product is positive but truncates to zero. These formulas describe the recovered AP9X firmware path and its integer operations; they are not a verified physical-output calibration. Although the handler stores `time`, `ramp`, and the `end` period, this command-received conversion branch reads the `start` period; the analyzed path does not establish how or whether those schedule fields are interpolated or applied later. Treat the request as a fixed-group write with a partially recovered conversion, not as a proven schedule engine.
