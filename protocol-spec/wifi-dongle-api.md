# WiFi Dongle v1.5.13 network API (legacy app protocol)

This reference covers the legacy Android app API and firmware-derived request schemas. The supported individual-light sequence is at the top; detailed K-Link frames and a high-level firmware-update summary are in separate references. Evidence is primarily the supplied app v1.15.3 and Dongle firmware v1.5.13. No live device capture has validated the exchanges.

## Connect to the Dongle: direct AP or home router

The WiFi Dongle running the analyzed v1.5.13 firmware supports both connection arrangements; they change how the client reaches the device, not the recovered local command protocol. In direct mode, join the Dongle's Kessil WiFi network (the Dongle serial number is the network name; the package QR code carries credentials, and the manual fallback password is the serial reversed in uppercase). The analyzed v1.5.13 first-boot script independently shows the AP key is the character-reversed SSID and configures OpenWrt encryption as `psk2` (WPA2-PSK); it prefers an SSID from the `wifi_ssid` boot environment and falls back to the serial. Although the script reads a `wifi_key` boot variable, its code that would use it is commented out. This establishes the bundled image's AP configuration, not the negotiated cipher on a physical unit or later firmware. Keep the phone/computer on that network while controlling it. Kessil describes the connected direct/AP state as a solid blue indicator. In router mode, configure the Dongle for the home WiFi network and connect the client to that same LAN; Kessil describes a solid green indicator for a successful station/router connection. The app's `dtype:201` request passes the selected router's encryption string through as `ENC`; the firmware helper stores it as the station interface's encryption setting. The manual names these “Connect Directly Via WiFi” and “Through Wireless Router.” [Kessil WiFi App manual](https://kessil.com/downloads/accessories/kessilwifi_UserManual/wifi_user_manual.html) · [Dongle troubleshooting guide](https://www.kessil.com/downloads/accessories/WiFi%20Dongle%20Troubleshooting.pdf)

In direct AP mode, Dongle firmware 1.5.13 sets `network.lan.ipaddr` to `10.0.100.1`, configures that interface as static, and runs DHCP with start address `100` and limit `32` (client leases `10.0.100.100`–`10.0.100.131`). Therefore the default host for a direct connection to this firmware is `10.0.100.1`. These values come from the extracted `/etc/uci-defaults/51_linkit_config`, `/usr/bin/wifi_mode`, and `/etc/config/dhcp`; they describe the bundled 1.5.13 image and could differ in another release or after configuration changes. The `192.168.1.1` literals in `/etc/preinit` are OpenWrt failsafe addresses, not the normal AP-mode address.

In router/station mode, the extracted `/bin/ap_to_sta` helper stores the router SSID, password, and encryption setting in the `wireless.sta` configuration, then runs `wifi_mode sta`. That mode hides the Dongle AP and attaches the station interface to `wan`; the bundled initialization sets `network.wan.proto=dhcp`, so the router assigns the IP used for control. The helper stops `tcpsvd` while changing modes, so any active FTP session is interrupted.

The APK's `APs.toDtype201Command` builder recovers the `dtype:201` JSON shape directly. It maps the selected access point's `ESSID`, `password`, `enc`, and integer `band` fields to request keys `SSID`, `PWD`, `ENC`, and `BAND`, then adds integer `SAVE: 1`:

```text
{
  "dtype": 201,
  "data": {
    "SSID": <string>,
    "PWD": <string>,
    "ENC": <string>,
    "BAND": <integer>,
    "SAVE": 1
  }
}
```

This is a typed schema sketch, not a literal JSON request. It is the app's serialized request shape; it does not by itself establish the accepted range or meaning of `ENC`/`BAND`, what `SAVE` changes, or the firmware's validation policy. Firmware analysis independently shows that the `dtype:201` path logs SSID/password and invokes `/bin/ap_to_sta`, whose configuration writes SSID/password/encryption. No live request or mode transition has been captured.

In either arrangement, the command transport is TCP port `8888`. In router/station mode, use the router-assigned IP from its DHCP client list or a local discovery response. Legacy Dongle discovery uses UDP port `8888` with a `POLO` response; it is separate from JSON control over TCP. No discovery utility is included in this publication.

## Network endpoints and message model

| Finding | Evidence | Confidence |
|---|---|---|
| Dongle UDP service listens on port 8888 | Firmware strings: `DONGLE UDP running on 8888`, `Waiting for incoming connections... @ port 8888`, and `udp recv: (%lu)[%s]` | High for listener/port; packet semantics still need capture |
| Android app sends command requests over TCP port 8888 | APK creates a `SocketAddress(host, 0x22b8)` (`0x22b8` = 8888) in its product-check/socket helper paths, opens buffered streams, and writes the command string to the output stream | High for app TCP destination and raw write behavior |
| Legacy TCP `8888` request path has no recovered authentication or TLS | The APK writes raw JSON bytes; the firmware's `connection_handler` dispatches each received buffer as JSON or raw K-Link based on its marker/length/terminator and shows no credential handshake or TLS layer in that path | High for the analyzed app/firmware path; this does not establish behavior of newer firmware or a physical capture |
| Dongle TCP port 8888 also accepts raw K-Link frames | `connection_handler` (`0x00402f68`) classifies a receive as raw when it begins `A9`, its length byte equals received-byte-count minus 3, and its last byte is `5C`; it forwards those bytes through `WriteData` to the UART | High for the static bridge path; no live raw-tunnel exchange has been captured |
| AP9X UDP discovery uses port 19276 and a `dtype:300` request / `dtype:301` reply | AP9X `k-daemon` enables `SO_BROADCAST`, binds UDP port `0x4b4c` (19276) on all interfaces, and parses a JSON object with top-level integer `dtype:300` and string `data`. The parser maps a string beginning `android` to platform code 0, one beginning `iphone` to 1, and any other string to 2; all three codes pass the worker's `<3` check. It replies with `dtype:301` to the datagram's source address/port. The response serializer emits `serial`, `rssi`, and `dtype`; the Android APK listens on the same port and accepts `dtype:301`. | High for the firmware-accepted fields, prefix mapping, transport, reply destination, and JSON response fields; no live exchange has been captured |
| The Android app receives AP9X UDP replies | APK configures its receive helper with a 1024-byte buffer, local port `0x4b4c` (19276), 15,000 ms timeout, and a fourth boolean set to `true`; the receiver parses incoming data as JSON into `UdpBroadcastReply.JsonModel` and accepts `dtype:301`. The model contains `serial`, `version`, and `rssi`; the resulting wrapper also receives sender IP and the constant `101` as its cell-type value. The supplied APK has no `lib/` native-code entries, and a scan of both DEX files found UDP socket/packet receive and close references but no `DatagramSocket.send`, `DatagramChannel.send`, or `MulticastSocket.send` call site. | High for the app-side receive configuration and JSON gate; the AP9X request schema is recovered from firmware, but the APK's sender/origin remains unknown |
| Local application commands use JSON-like objects with `dtype` and `data` | App command builder emits `{"dtype":202, "data":{"timestamp":"...","timezone":"..."}}`; firmware response templates use `dType` (capital T) with JSON fields | High for this request shape example; casing and exact envelope for every command remain to be checked |
| Dongle/AP9X Apex integration uses TCP and UDP port 0x6961 (26977) when enabled | Both firmware generations start Apex TCP and UDP services when Apex mode is enabled. Dongle 1.5.13's TCP IOTA JSON uses a Salsa20/12 stream cipher; AP9X WiFi 1.4.20's TCP IOTA listener parses plaintext JSON. The command TCP path remains on 8888. | High for mode, transports, ports, cipher split, and dispatch shape; no live encrypted or plaintext exchange has been captured |
| UDP port 8888 replies with ASCII `POLO` | `UDPServer` binds `0.0.0.0:8888`/UDP and sends the four bytes `POLO` back to the source address after receiving a datagram | High for this firmware's behavior; request contents are logged but not interpreted in the recovered function |

The APK's application code creates command objects containing a JSON data string, retry/wait settings, and callback state, then queues them per dongle IP. Its socket helper writes the command string's bytes directly to a buffered TCP output stream and flushes; no newline, length prefix, or other wrapper is added in that path. It accumulates available response bytes and attempts Gson/JSON parsing until the response parses, then returns the accumulated text. This establishes the official client's TCP write/read behavior, though a capture is still needed to confirm actual response packetization and whether all commands use this same helper. No authentication handshake or TLS layer was recovered for this port-8888 path; the direct-AP WPA2 password only controls WiFi association, and station/router mode places the service on the LAN. Treat network access to TCP `8888` as control access. UDP discovery remains a separate flow; do not send the app's raw JSON command over UDP based only on the shared port number.

The AP9X firmware-side UDP parser provides a minimal accepted request example for port 19276:

```json
{"dtype":300,"data":"android"}
```

`data` is a top-level string (not an object nested under `data`); the sample above shows the parser-accepted form and the matching Python probe. The parser selects an internal platform code from the string prefix (`android`, `iphone`, or other), then the worker obtains the AP9X serial and Wi-Fi RSSI and serializes a `dtype:301` reply to the sender. The supplied APK only contains the reply receiver, so this schema does not establish that this exact app emits the request or whether another discovery component sends extra fields.

Static analysis of the Dongle server resolves an important transport constraint: `initServer` (`0x0043b9b0`) accepts a TCP socket and starts `connection_handler` (`0x00402f68`), which calls `recv` into a 2048-byte buffer and immediately parses each positive read as one request. It does not accumulate partial JSON or split multiple coalesced requests. TCP does not preserve `send` boundaries, so use one small request at a time and wait for its response before sending another; even this is best-effort because a single request can be split by TCP. `writeOut` (`0x0043c2e0`) writes the supplied response bytes without adding a common length prefix or terminator. It calls `write` once and ignores its return value, so the recovered code has no retry for a partial TCP write. The app's read-until-JSON-parses behavior is consistent with this raw response path; no live capture has verified segmentation or failure behavior.

The legacy JSON `dispatch_cmd` only enters its normal command switch when `APEX_FLAG` is clear; with Apex mode enabled, the gate permits only command IDs `201` and `210` through that switch. Therefore the `dtype:0`–`117` Kessil API, including cached-info `dtype:114` and the individual-light `dtype:3`/`4` path, assumes normal Kessil mode. Enabling Apex mode (`dtype:210`) rewrites assignments for inventoried lamps; disabling it clears Apex mode/data but does not restore the prior KID-to-GID assignments. The raw `A9` tunnel is a separate connection-handler branch and is not subject to this JSON dispatcher gate.

## Minimal legacy API sequence for one-light control

This request sequence is derived from the APK builders and Dongle 1.5.13 dispatcher and assumes the Dongle is in normal Kessil mode (`APEX_FLAG` clear). Connect by TCP to port `8888`, send one JSON request, and wait for its JSON response before sending the next request; the APK writes raw JSON bytes without a newline or length prefix. Inspect cached lamp records with `dtype:114`, choose an existing GID that is not shared with another lamp, and assign the target KID to it. `dtype:114` does not query K-Link and can be stale. `dtype:0`, `dtype:1`, and `dtype:5` can trigger group-reconciliation writes; see the query and scan side-effect notes before using them. Angle-bracketed values below are documentation placeholders and must be replaced with concrete values before sending:

```json
{"dtype":114,"data":{}}
{"dtype":4,"data":{"KID":"<target-KID>","GID":<exclusive-GID>}}
```

Then write that GID's five channel slots (replace the example values with values valid for the selected group type and fixture):

```json
{"dtype":3,"data":{"GID":<exclusive-GID>,"UI1":<0-255>,"UI2":<0-255>,"UI3":<0-100>,"UI4":<0-100>,"UI5":<0-100>,"save":1,"enabled":1}}
```

`save:1` stores the group values and `enabled:1` requests the live K-Link write. These commands address the GID, not the KID, so every lamp still assigned to that GID receives the same group write. The cached-inventory/setup sequence and message shapes are code-derived; group availability, app/device timing, acknowledgements, and the resulting light output have not been validated in a live exchange. Do not use this as a fixture-independent write until inventory confirms the target is alone in the selected group.

## AP9X support for individual light control

The AP9X WiFi 1.4.20 `dispatch_cmd` also has the legacy low-numbered lamp operations. Its `dtype:3` branch calls the AP9X `group_set_knob` routine, and `dtype:4` calls `lamp_set_group`; `dtype:1`, `2`, and `5` dispatch to lamp query, find-me, and info-refresh routines. This establishes that AP9X has the same command roles needed for the individual-control workflow: assign a KID to a group, then write that group's channels. It does not prove every field, validation rule, or response is byte-for-byte identical to Dongle 1.5.13, and the sequence has not been captured against AP9X hardware. The AP9X `group_set_knob` implementation accepts GIDs 1–40, requires `UI1`/`UI2` below 256 and `UI3`–`UI5` below 101, and when `save` is `1` stores those values in group state. The parser maps JSON `enabled` to the field used for the direct K-Link write gate: its request-state template initializes that field to `-1`, and `group_set_knob` calls `klink_set_input_by_group` only when the field equals `-1`. Therefore omit `enabled` to get the recovered default direct-write path; explicitly sending `enabled:0` or `enabled:1` prevents that immediate K-Link call. This is the reverse of the Dongle example's `enabled:1` behavior and is a critical compatibility-profile difference. The AP9X `conn_app_con_handler` (`0x00415b2c`) reads one TCP chunk of up to 2048 bytes and passes it to `parseJsonCmd` before dispatch; it does not accumulate a fragmented JSON request in this handler. The server's response behavior and reuse of the connection still need live validation.

For an ordinary single-GID lamp, AP9X parses `dtype:4` fields `KID` and `GID`. Its setter checks that the selected destination group's configured type matches the lamp's type, updates the saved group membership, and issues K-Link `klink_save_gid`. The firmware also parses `Z2TL_GID` and `Z2TR_GID` for its dual-group path; these are not substitutes for a single GID on an ordinary lamp. The AP9X-shaped assignment and channel requests are:

```json
{"dtype":4,"data":{"KID":18001234,"GID":7}}
{"dtype":3,"data":{"GID":7,"UI1":10,"UI2":20,"UI3":0,"UI4":40,"UI5":50,"save":1}}
```

In the `dtype:3` request, omit `enabled` to preserve the firmware's initialized `-1` write gate. `save:1` also clears the group's active program markers while storing the manual channel values, so this is a persistent manual-mode change. These exact effects are static findings from AP9X WiFi 1.4.20; do not run the example until GID 7 is confirmed unused or exclusive and has the matching lamp type. The numeric KID shown is illustrative and must be replaced with the actual discovered KID.

On the recovered AP9X `dtype:4` success branch, the response is serialized with `code:0`, `msg:"Success"`, `dType:4`, and a `lamp` object containing `KID`, `GID`, `rGID`, `lGID`, `model`, `FW`, `type`, and `timestamp`. This firmware-side format is supported by the static response template; exact live response formatting and the values returned after a group change have not been captured.

Use the TCP `8888` JSON API for AP9X; AP9X discovery is the separate UDP `19276` path described above. Do not assume the Dongle's raw `A9` K-Link tunnel is present on AP9X: its network module controls a separate K-Link MCU, and the analyzed AP9X TCP dispatcher has no recovered Dongle-style raw tunnel branch. The existing `kessil_control_light.py` client is consequently documented and tested against the Dongle API; AP9X support should be treated as a compatible-looking but not yet live-validated target until request parsing and group-write behavior are checked against its firmware and hardware.

## Firmware command families visible in response templates

The firmware contains response templates keyed by `dType`. For most IDs, only the response side is mapped; `dtype:201` is an exception because the supplied APK exposes its request builder above. This index remains a response-side reference, not a fully interoperable command specification for every row.

| `dType` | Firmware-visible operation/result | Fields or evidence |
|---:|---|---|
| 0–5 | Lamp/group operations and discovery-related results | `dType:5` success contains `lamps:[...]`; error templates distinguish invalid command/GID and collisions |
| 100–113 | Groups and programs | Response templates mention group/program existence, save/delete/copy/rename, points, and program collections |
| 114 | Dongle/system information | `sn`, `ver`, `SSID`, `ENC`, `timezone`, `demo`, `BAND`, `lamps`, `num_prog_ids`, `group_ids` |
| 115–117 | Dongle configuration and sibling/APEX information | 116 returns `siblings`; 117 returns `SN`, `siblings`, `APEX` |
| 200 | WiFi access point scan | Success returns `APs` array; AP item template has `ESSID`, `signal`, `enc`, `channel`, `BAND` |
| 201 | Configure station WiFi | APK builder sends `SSID`, `PWD`, `ENC`, integer `BAND`, and `SAVE:1`; firmware logs SSID/password and calls `/bin/ap_to_sta`. The success response also has `SSID`, `PWD`, and `ENC`. Accepted values, firmware use of `BAND`/`SAVE`, and validation policy remain unresolved. |
| 202 | Set time and timezone | APK builder confirms request `data` contains `timestamp` and `timezone`; firmware invokes `/bin/time_change` |
| 204 | Radio performance statistics | Returns `TxPER`, `RxCRC`, `AMPDU` |
| 205 | Start FTP service | Firmware starts `ftpd`; response reports success/failure |
| 206–208 | Firmware transfer/upgrade | Strings reference merge/transfer, `dongleUpgrade`, and `sysupgrade /tmp/Data/*.bin` |
| 209 | Generate configuration | APK command enum labels this `cmd_config_generation` |
| 210 | Enable Apex mode | Firmware creates/removes `/etc/.apex` |
| 213 | Query Apex mode | APK command enum labels this `cmd_get_apex_mode` |
| 300–301 | UDP broadcast / reply | APK enum names these `cmd_udp_broadcast` and `cmd_reply_udp_broadcast`. The receive worker validates `dtype:301`; `dtype:300` is present in the enum, but no request builder or call site was recovered in the supplied DEX files. |

Lamp records include `KID`, `GID`, `rGID`, `lGID`, `model`, `FW`, `type`, and `timestamp`. Program point records expose `minutes`, a five-value `UI` array, `eid`, `duration`, and `freq`. The firmware also names light parser/dump routines for Get, Color, Set, Flash, Status, Fallback, and Capability, and identifies five UI channels. These names strongly indicate the shape of the light-control model but do not, by themselves, establish the exact command field mapping or channel order for every model.

In the checksum-mismatched v1.4 catalog candidate, the explicit `dType:209` branch formats a success reply shaped like `{"msg":"success","code":0,"dType":209,"UID":"<value>","timestamp":<milliseconds>}`, with the timestamp supplied by `current_timestamp()`. This adds a response-side detail to the APK enum's `cmd_config_generation` label, but the request fields and `UID` meaning were not recovered. Treat it as candidate-only comparative evidence, not as a verified behavior of the catalog's advertised image.

## Request schemas recovered from the Android app

For payload-bearing commands, the APK builds an object containing an integer `dtype` and a nested `data` object, then serializes it with Gson. Ghidra's decompilation of the APK enum `p9.c` exposes each command's explicit wire value (the third constructor argument; distinct from the enum ordinal). The common envelope is `{"dtype":N,"data":{...}}`; the scan-lamps builder is an explicit exception and emits only `{"dtype":361}`.

| App builder | Serialized `data` fields | Likely operation | Wire `dtype` |
|---|---|---|---|
| `function.b::s` | empty object | Discover/scan lamps (`cmd_discover`) | 0 |
| `function.b$j` (`classes2.dex`) | empty object | Refresh/get lamp info (`cmd_get_lamps_info`); live `getInfo` path can reconcile assignments | 5 |
| `function.b::f` | `KID`; for lamp type 0, also `SIDE` | Find/identify lamp (`cmd_find_me`) | 2 |
| `function.b::i` | `GID` (integer), `UI1`…`UI5` (integers), `save` (integer); conditionally `enabled` (integer, one traced caller passes `0`) | Set five manual UI/channel values (`cmd_set_knob`) | 3 |
| `function.b::w` | `KID` (string), `GID` (integer) | Assign lamp to group (`cmd_set_group`) | 4 |
| `function.b::n` | `GID`, `EID`, `FREQ`, `duration` | Run effect (`cmd_run_effect`) | 101 |
| `function.b::o` / `p` | `GID`, `PID` | Run program (`cmd_run_program`) | 100 |
| `function.b::g` | `GID` | Get groups (`cmd_get_groups`) | 102 |
| `function.b::b` | `GID`, `name`, `type` | Create group (`cmd_create_group`) | 103 |
| `function.b::d` | `GID` | Delete group (`cmd_delete_group`) | 104 |
| `function.b::j` | `PID`, `type` | Get program (`cmd_get_program`) | 106 |
| `function.b::q` / `r` | `PID`, `type`, `name`, `points`; `r` normalizes child keys `minute`→`minutes`, `eid`→`EID` | Save program (`cmd_save_program`) | 107 |
| `function.b::c` | `PID`, `type` | Delete program (`cmd_delete_program`) | 108 |
| `function.b::l` | `from`, `to`, `type` | Reorder programs (`cmd_reorder_program`) | 109 |
| `function.b::t` / `u` / `v` | `GID` plus `period`/`start`, `enabled`, or `reset` respectively | Set acclimation (`cmd_set_acc`) | 111 |
| `function.b::z` / `A` / `B` / `C` | `GID` plus lunar settings (`end`, `begin`, `high`, `phase`, `color`, `enabled`, or `reset`) | Set lunar mode (`cmd_set_lunar`) | 112 |
| `function.b::h` | `GID`, `pause` | Pause/resume group program (`cmd_pause_group`) | 113 |
| `function.b::y` | empty object | Get Dongle/system info (`cmd_get_system_info`) | 114 |
| `function.b::e` | `enabled` | Set demo mode (`cmd_demo_mode`) | 115 |
| `function.b::D` | `siblings` | Set sibling Dongles (`cmd_set_siblings`) | 116 |
| `function.b::E` / `function.g::H` | `timestamp` (decimal string), `timezone` (string) | Set time (`cmd_set_datetime`); firmware invokes `/bin/time_change` | 202 |
| `function.b::a` / `x` | `enabled`; `a` also has `host` | Start FTP (`cmd_start_ftp`) | 205 |
| `function.b::k` | `host` | Set file-transfer target host (`cmd_put_file_to_target_host`) | 206 |
| `function.b::m` | `enabled` | Restart Dongle (`cmd_restart_dongle`) | 207 |
| `elite.tasks.a` | `checksum` (string); optional `checksum_st` (string) | Start firmware upgrade (`cmd_upgrade_dongle_FW`); values come from the selected catalog firmware record | 208 |
| `function.b$q` (`classes2.dex`) | `gid` (integer), `save` (boolean), `UI` (integer array) | Newer manual-mode group update (`cmd_modify_manual_mode`) | 323 |
| `function.b$k` / `b$l` (`classes2.dex`) | `gid` (integer array); accepts a list of `Group` objects or integers | Get multiple group info (`cmd_get_multiple_group_info`) | 325 |
| `function.b$r` (`classes2.dex`) | `gid` (integer), `pause` (boolean) | Pause/resume group v2 (`cmd_pause_group_v2`) | 326 |
| `function.b$v` (`classes2.dex`) | `gid` (integer), `bStatus` (boolean) | Switch group on/off (`com_switch_on_off`) | 329 |
| `function.b$b` (`classes2.dex`) | `name`, `type`, `points`, `effect_points` from `RequestSets.OutProgram` | Add program (`cmd_add_a_program`) | 340 |
| `function.b$f` (`classes2.dex`) | `pid` (integer) | Delete program (`cmd_delete_a_program`) | 341 |
| `function.b$p` (`classes2.dex`) | `gid`, `pid` (integers) | `cmd_move_group`; exact side effect unresolved | 345 |
| `function.b$m` (`classes2.dex`) | `pid` (integer) | Get program info (`cmd_get_program_info`) | 348 |
| `function.b$t::c` | no `data` member; serialized command is `{"dtype":361}` | Scan K-Link lamps (`cmd_scan_lamps`) | 361 |

Firmware dispatch confirms the core legacy individual-lamp operations for Dongle 1.5.13: `dtype:1` queries a lamp, `dtype:2` runs find-me, `dtype:3` changes manual values for a GID, and `dtype:4` moves a KID to a GID. `dtype:0` scans and `dtype:5` refreshes lamp information. The firmware command parser confirms that `dtype:1` and `dtype:2` take the exact input key `KID` inside `data` and convert its token text with `atoi`; the dispatcher requires a positive result. The parser does not check the JSON token type, so both a JSON integer (`"KID":18001234`) and a quoted decimal string (`"KID":"18001234"`) are accepted by this recovered path. The APK itself uses a quoted string for `dtype:4` assignment, so that remains the canonical assignment form; number/string equivalence is a static parser result, not a live-tested compatibility guarantee. The statically recovered query request shape is `{"dtype":1,"data":{"KID":18001234}}`; the request serializer was not found in either code DEX of the supplied APK, so this shape comes from firmware parsing rather than an app-generated example. `dtype:1` calls `getInfo` and returns `{"code":0,"msg":"Success","lamp":{...}}`, but its live query can also reconcile a lamp to its default GID if its type conflicts with the saved group type. The shared `printLamp` serializer used for this response and the inventory array emits `KID`, `GID`, `lGID`, `rGID`, `model`, `FWVer`, `type`, `temp`, and `timestamp`; the internal command-`35` status slot is omitted. `dtype:2` sends a find-me command and returns `code` plus `msg`. For recovered schemas, use the APK builder table and keep the firmware's field casing separate because they are not identical everywhere.

### Fixture type and default group

The app's product enum and the Dongle's type-reconciliation defaults provide this type map. Default GIDs are starting assignments; for independent control, use a group whose positive type matches the lamp and whose membership is exclusive to that lamp.

| Dongle `type` | Fixture class | Default GID | Notes |
|---:|---|---:|---|
| 1 | Tuna Blue (`TB`) | 1 | Firmware `getInfo` reconciliation selects GID 1 for type 1. |
| 2 | Tuna Sun (`TS`) | 2 | Firmware `getInfo` reconciliation selects GID 2 for type 2. |
| 3 | Refugium (`RF`) | 26 | Firmware `getInfo` reconciliation selects GID 26 for type 3; `UI2` is capped at 240. |
| 0 | Untyped/generic group | No model default | Not suitable for full five-channel writes: the Dongle sends `UI1`/`UI2` and zeros `UI3`–`UI5`. |

The three model-to-GID defaults are also documented by Kessil's [Signal Converter Box guide](https://kessil.com/downloads/accessories/Kessil%20Signal%20Converter%20user%20guide-20240108A.pdf), while the firmware reconciliation path independently recovers the same mapping. GID is a group selector, not a lamp identity; use KID for the fixture-specific assignment request.

For example, firmware parsing plus the recovered APK builders establish this legacy lamp query/setup/control sequence (values below are illustrative, not captured live):

The example assumes GID 7 already exists and has the same positive type as the selected lamp (for Tuna Sun, type 2). Query it with `dtype:102` before assignment: `dtype:4` accepts an existing target group whose type is 0 or equals the lamp's reported `type`, and rejects a positive type mismatch. Although type 0 passes the initial validation, the handler's follow-up `getInfo` sees the type mismatch and can immediately send `SetGid` to the lamp type's default group; use a matching positive group type to keep the assignment stable. Type 0 also is not suitable for full five-value output: the firmware sends only `UI1`/`UI2` and zeros `UI3`–`UI5`. If no suitable group exists, `dtype:103` creates one only for GIDs 3–25, with a nonempty `name` and `type` 0–3. The handler does not require the GID to be unused; it overwrites that group's name and type, so choose an unused GID rather than repurposing an occupied group.

The lines in this block represent **six separate TCP requests**, sent in order to the Dongle on port 8888, with each response handled before sending the next request. This all-five-value example assumes the cached `dtype:114` response contains the target KID and the preflight `dtype:102` query returns GID 7 successfully with the selected lamp's positive type (type 2 for Tuna Sun). If current K-Link identity must be verified, use a raw `1C` query for that known KID; JSON `dtype:1` and `dtype:5` can reconcile group assignments. The JSON objects are shown one per line for readability; do not concatenate them as one JSON document or append line-feed delimiters. The APK writes each serialized JSON object's bytes directly and flushes the TCP stream. A TCP client may split or combine reads arbitrarily, so read/parse behavior should follow the device response rather than assuming one socket read equals one reply.

```json
{"dtype":114,"data":{}}
{"dtype":102,"data":{"GID":7}}
{"dtype":4,"data":{"KID":"18001234","GID":7}}
{"dtype":3,"data":{"GID":7,"UI1":10,"UI2":20,"UI3":30,"UI4":40,"UI5":50,"save":1,"enabled":1}}
{"dtype":102,"data":{"GID":7}}
{"dtype":114,"data":{}}
```

`dtype:114` returns the cached lamp inventory as a `lamps:[...]` array using the `printLamp` fields: KID, GID, left/right GIDs, model, firmware version, type, temperature, and timestamp. Use `dtype:102` to verify GID 7 exists and has a compatible type, assign the KID to GID 7 with `dtype:4`, write the five manual channel values to that GID with `dtype:3`, then query `dtype:102` again to read back the group's stored manual values and `dtype:114` to inspect the cached assignment. The cached inventory may be stale, and `dtype:102`'s `group.UI` is stored configuration rather than measured light output. Keep GID 7 exclusive to the target lamp for individual control. This order is inferred from the command purposes and schemas; it has not been captured live. The APK also builds a newer manual-mode request, `{"dtype":323,"data":{"gid":7,"save":false,"UI":[10,20,30,40,50]}}`, but Dongle firmware 1.5.13 has no corresponding dispatch case; that shape is an APK schema finding, not a compatible substitute for the legacy `dtype:3` / `dtype:4` path on this firmware.

### Control checks and response semantics

Before changing a group assignment, compare cached inventory (`dtype:114`), group configuration (`dtype:102`), and a validated raw K-Link `1C` identity reply for the target. Confirm the destination group exists, has a compatible type, and belongs exclusively to the intended fixture. After assignment, query the fixture identity again before a group-wide channel write. After a write, cached inventory and group state can confirm stored configuration, but cannot establish measured light output or guarantee complete discovery of every group member. No runnable control client is included here.

The `dtype:4` assignment request targets one fixture through its `KID`; the `dtype:3` manual write affects all lamps assigned to GID 7. The APK serializes the legacy `save` flag as an integer, whereas the newer `dtype:323` builder serializes its `save` field as a boolean. Use a group containing only the intended fixture for independent control. Firmware 1.5.13 validates GID in 1–40, `UI1`/`UI2` in 0–255 (with `UI2` limited to 0–240 for group type 3), and `UI3`–`UI5` in 0–100. In the handler, `save:1` persists the five values into the saved group configuration; `enabled:1` causes a K-Link write. If `enabled` is omitted the firmware defaults it to 1, while `enabled:0` suppresses the live K-Link write. Thus a request with `save:1, enabled:0` changes saved values without applying them live; this explains why not every app `dtype:3` call is itself a light-control write.

The `dtype:3` response contains `code`, `msg`, `group`, and `dType`. `group` is the selected group's state object, including `gid`, `pid`, `eid`, `type`, `hasLamps`, `name`, `pause`, `startDate`, the five-value `UI` array, and `accl`/`lunar` configuration objects. The `UI` array is built from the group's stored manual values. Consequently, after `save:1` it reflects the submitted values; with `save:0`, it reflects the previous saved values even when `enabled:1` sent the submitted values over K-Link. With `enabled:0`, `code` remains initialized to zero and `msg` is empty because no K-Link write was attempted, so this response must not be treated as confirmation that the physical light changed. This response provides group-state readback that is separate from a measured output/channel readback from the fixture.

The `dtype:4` handler moves the KID's recorded GID from its previous group to the requested group, updates the groups' lamp counts, persists the group configuration, sends the K-Link GID assignment, and then refreshes lamp information. That follow-up `getInfo` can send another `SetGid` if the requested group's saved type conflicts with the lamp type; this is why a type-0 group may not remain assigned. The corresponding lower-level K-Link operation is command `1B` (`SetGid`): it carries the target KID as a four-byte big-endian value and the destination GID as one byte, as in the example frame above. This is the per-lamp assignment step; the later command `1F` still addresses the GID and applies to every lamp in it. The target group must exist; when its configured type is positive, it must match the lamp's type or the request returns a type-inconsistency error. On success, the response includes a `lamp` object with `KID`, `GID`, `rGID`, `lGID`, `model`, `FW`, `type`, and `timestamp`. An exclusive group therefore remains necessary for individual control.

The recovered `dtype:4` response branches are useful when recovering from a failed or interrupted assignment:

| Response `code` / `msg` | Recovered condition | State implication |
|---|---|---|
| `-1`, `Invalid command format` | Missing/invalid KID or GID, or GID above 40 | Rejected before the `setGroup` path. |
| `-1`, `lamp does not exist.` | KID is not in the current lamp inventory | No group move is attempted. |
| `-1`, `group is not exists.` | Destination group is absent | No group move is attempted. |
| `-1`, `type is inconsist.` | Destination has a positive type different from the lamp type | No group move is attempted. |
| `-1`, `Save file failed.` | Persisting the updated group state fails | The old group record is restored in memory; no K-Link write is reached. |
| negative `code` with the `WriteData` message | The K-Link `SetGid` write reports an error | Group state was already saved and the lamp's in-memory GID changed; the handler does not roll them back. |
| `-1`, `Lamp is not existed.` | The write returned success, but the subsequent `getInfo` refresh did not recover the lamp | The assignment frame was sent; the response does not establish the fixture's resulting state. |
| `0`, `Success` | Write and follow-up query succeeded | `lamp` contains refreshed identity/group fields. A later lamp-record save failure is logged but does not change this response. |

These branches are statically recovered from Dongle 1.5.13's `dispatch_cmd` and `setGroup`; they have not been induced on a device. Treat any nonzero result as a reason to query `dtype:102` and inspect cached records with `dtype:114`; if the current K-Link assignment must be checked, use a raw `1C` query for that known KID. Avoid `dtype:1`/`5` as passive retry checks because `getInfo` can reconcile mismatched assignments.

The `dtype:3` and `dtype:4` handlers are not atomic across saved configuration and K-Link transmission. They persist group state before calling `WriteData`; `dtype:4` also updates the in-memory KID-to-GID record before its bus write. If `WriteData` returns an error, the handler reports failure but does not restore the already-persisted group state or the updated lamp record in that path. A nonzero response therefore does not prove that no configuration changed; inspect `dtype:102` and cached `dtype:114` state before deciding whether a retry is needed. A live `dtype:1`/`5` check may trigger additional assignment writes.

The `dtype:102` request takes `data.GID` and returns the same single group-state object in a `group` field (`msg`, `code`, and `dType` are also returned). It is useful to read back persisted manual values and group configuration independently of issuing a control write. Like the `dtype:3` response's `group.UI`, these are saved group values, not a sensor measurement of the fixture's emitted channels.

Retries, wait intervals, and callbacks are client queue behavior, not JSON fields. The field schemas come from explicit Gson object construction in the app builder.

## Android command ID enum and chain-related commands

Ghidra decompilation of the APK's `p9.c` enum static initializer recovers explicit wire values (the third constructor argument). The app defines these command IDs:

| Wire ID | APK command name |
|---:|---|
| 0–5 | `cmd_discover`, `cmd_query`, `cmd_find_me`, `cmd_set_knob`, `cmd_set_group`, `cmd_get_lamps_info` |
| 100–117 | `cmd_run_program`, `cmd_run_effect`, `cmd_get_groups`, `cmd_create_group`, `cmd_delete_group`, `cmd_get_programs`, `cmd_get_program`, `cmd_save_program`, `cmd_delete_program`, `cmd_reorder_program`, `cmd_rename_program`, `cmd_set_acc`, `cmd_set_lunar`, `cmd_pause_group`, `cmd_get_system_info`, `cmd_demo_mode`, `cmd_set_siblings`, `cmd_get_siblings` |
| 200–210 | `cmd_scan_routers` (200), `cmd_set_WAN` (201), `cmd_set_datetime` (202), 203 unused, `cmd_get_per` (204), `cmd_start_ftp` (205), `cmd_put_file_to_target_host` (206), `cmd_restart_dongle` (207), `cmd_upgrade_dongle_FW` (208), `cmd_config_generation` (209), `cmd_enable_apex` (210) |
| 213 | `cmd_get_apex_mode` |
| 300–301 | `cmd_udp_broadcast`, `cmd_reply_udp_broadcast` |
| 320–326 | `cmd_cmd_add_a_group` (320), `cmd_delete_a_group` (321), `cmd_rename_a_group` (322), `cmd_modify_manual_mode` (323), `cmd_get_multiple_group_identifiers` (324), `cmd_get_multiple_group_info` (325), `cmd_pause_group_v2` (326) |
| 329 | `com_switch_on_off` |
| 340–341 | `cmd_add_a_program` (340), `cmd_delete_a_program` (341) |
| 345 | `cmd_move_group` |
| 348 | `cmd_get_program_info` |
| 360–364 | `cmd_modify_lamp_group` (360), `cmd_scan_lamps` (361), `cmd_find_me_v2` (363), `cmd_query_all_lamps` (364) |
| 381 | `cmd_delete_device` |
| 386 | `cmd_set_multiple_wifi_connection` |

This list is the app's enum, not proof every command exists in every Dongle firmware version. Values 320 onward include multi-group and program operations. The secondary DEX (`classes2.dex`) provides payload builders for 323, 325, 326, 329, 340, 341, 345, 348, and 361; see the table. `dtype:323` serializes one `GroupIdUiArray` as `data` with fields `gid` (integer), `save` (boolean), and `UI` (integer array). The five values in the example below are illustrative and match the legacy five-channel controls, while the builder itself accepts an integer array without validating its length. `dtype:325` serializes an array of group IDs under `data.gid`; it accepts either a list of `Group` objects or integers and extracts each group's ID. The APK also serializes `cmd_scan_lamps` as the no-data request `{"dtype":361}`. No request builders were found for `cmd_get_multiple_group_identifiers` (324), `cmd_modify_lamp_group` (360), `cmd_find_me_v2` (363), or `cmd_query_all_lamps` (364). A field-reference scan across both code DEX files finds the 360/363/364 enum fields only in enum initialization and the generated values array, with no builder or call site. Firmware 1.5.13 has no dispatch cases for 320–386, so none of these newer IDs can be assumed to work against the bundled firmware. These schemas also do not tell us how the two physical K-Link ports forward traffic. Enum values not listed in the table are unused in this app build.

The `dtype:340` program payload serializes `RequestSets.OutProgram`: `name` is a string, `type` an integer, `points` a list of `OutProgramPoint` objects (`minute` integer and `UI` integer array), and `effect_points` a list of `OutEffectPoint` objects (`minute`, `eid`, `freq`, and `duration`, all integers). The model constructor initializes the point lists and copies the supplied lists into them. This illustrates the APK's serialized shape only; the payload's acceptance, array lengths, value ranges, and execution semantics are not established, and neither bundled network firmware dispatches dtype 340.

```json
{"dtype":340,"data":{"name":"example","type":2,"points":[{"minute":0,"UI":[0,0,0,0,0]}],"effect_points":[{"minute":60,"eid":1,"freq":2,"duration":300}]}}
```

The point values above are illustrative placeholders for the serialized field types; they do not establish valid program ranges, effect IDs, timing units, or device acceptance.

The builders for 341, 345, 348, 326, and 329 expose their serialized keys directly in the table, but the request constructors alone do not establish their complete device-side semantics.

The app's `siblings` commands (116–117) and `cmd_set_multiple_wifi_connection` (386) appear to concern sibling/multiple WiFi dongles. They should not be equated with K-Link's two-port fixture chain without payload/handler evidence. Likewise, `KID`, `GID`, `lGID`, and `rGID` are lamp/group record fields; the labels alone do not prove that `lGID` and `rGID` identify the physical left/right ports.

### Legacy lamp discovery in Dongle firmware 1.5.13

The Dongle firmware's legacy scan request is `dType:0`; its scan handler unmutes all K-Link devices, scans KIDs in the interval 18,000,000–36,000,000, mutes individual responders, then reads lamp information and returns a `lamps` array. Discovery is recursive: `hasLamp(lo,hi)` sends a `Branch(lo,hi+1)` UART request; one valid 11-byte reply identifies a KID, no reply means empty range, and a collision causes the interval to be bisected and retried. The `Branch` builder emits this exact 15-byte frame:

| Offset | Value | Meaning |
|---:|---|---|
| 0 | `A9` | Start marker |
| 1 | `0C` | Length |
| 2–3 | `00 00` | Reserved |
| 4 | `18` | Branch/range-discovery command |
| 5–8 | `lo`, big-endian | Inclusive lower KID bound |
| 9–12 | `hi + 1`, big-endian | Exclusive upper bound; `hasLamp` adds one before calling the builder |
| 13 | checksum | Two's-complement of bytes 1–12 modulo 256 |
| 14 | `5C` | Terminator |

For the initial scan call `scanLamps(18000000, 36000000)`, `hasLamp` passes `hi + 1` into `Branch`, so the first frame is `A9 0C 00 00 18 01 12 A8 80 02 25 51 01 28 5C` (range `[0x0112A880, 0x02255101)`, covering decimal KIDs 18,000,000 through 36,000,000). The firmware recursively bisects ambiguous ranges, with the second recursive lower bound set to midpoint plus one. This is direct firmware evidence for how multiple K-Link lights are enumerated on a shared bus, though the reply/collision behavior still needs validation against a physical chain.

There is a version mismatch worth preserving: APK 1.15.3 exposes `cmd_scan_lamps` as `dtype:361`, but Dongle firmware 1.5.13's recovered dispatch switch has handlers for IDs 0–117 and 200–210; it has no 361 case. Thus the legacy `dType:0` scan is present in this firmware, while the APK's newer `dType:361` scan cannot be assumed to work against v1.5.13. A newer Dongle firmware may implement it.

The APK's newer command builders also do not match either bundled network-firmware dispatcher. Dongle 1.5.13 has no handlers for IDs 320–386. Independently, AP9X WiFi firmware 1.4.20's `dispatch_cmd` (`0x00405d98`) routes those IDs to its unknown-command error path (`code:0x3e`); its explicit high-ID commands are in the legacy 200–210 range, plus a separate 900 case. Thus the APK builders for `dtype:323`, `325`, `326`, `329`, `340`, `341`, `345`, `348`, and `361` are not supported by either bundled firmware image. This is static dispatcher evidence, not a live request/reply capture; a different firmware release could add them.
