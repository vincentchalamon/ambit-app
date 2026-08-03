# Picking the project back up - state, prerequisites, remaining work

This document is for someone taking over this project **on another machine and another
account**, with no prior context. It states what is settled, what is missing, and what is
left to do, milestone by milestone.

## Goal

Send a GPX route from a smartphone to a **Suunto Ambit3** over Bluetooth, with no server
and no account. Movescount is dead and the watch is no longer maintained. Intended base
for the application: a fork of `guiguoz/opensportsync` (React Native + `libambit` C core).

## Where things stand

| Milestone | State |
|---|---|
| 1 - analysis tooling | **done** |
| 2 - remaining format fields | **done** |
| 3 - serializer, Python and C | **done**, bit-exact against the captures |
| 4 - first real write (reset) | **ready**, needs the watch |
| 5 - first real route | **ready**, needs the watch |
| 6 - Android USB-OTG | **to do**, see below |
| 7 - BLE | **to do**, token research started |
| 8 - iOS | out of scope for now: Xcode, therefore macOS |

The binary format of the navigation database is **fully decoded and verified**. The
complete specification is in [`tools/README.md`](tools/README.md): memory map, structures,
coordinate formula, simplification, closing hash, reproduced quirks. Do not duplicate it
here, refer to it.

## What the repository does not contain

`assets/` and `full-assets/` are deliberately kept out of the repository: proprietary
Suunto software, a Microsoft DLL, a decompiled Movescount APK, and captures that carry the
watch serial number as well as personal POIs. **Without them, nothing can be verified.**
Get them from the project owner and put them back as follows:

```
assets/ambit3 pcap/
    route12km  route128km  routedelete  poiimport  sync  ambit3full
    orbitsync  orbitsync2  firmware
    Gare-du-Nord-to-114-Av.-André-Morizet.gpx        (1066 rtept, no <ele>)
    Grand-Tour-HDF---Partie-1---Lille-_-Arras.gpx    (2911 rtept, with <ele>)
assets/
    route.js  route_simplifier.js  navigation.js  poi.js  util.js  sttalg.js
    SDSApplicationServer.exe  SDSApplicationServer.exe.c  Devices.xml  production.json
    descr+<SERIAL>+2.4.17                            SBEM schema, Ambit3 Peak
    descr+<SERIAL>+2.0.5                             SBEM schema, Kailash
    library.xml  sgee.7d  logbook/*.bin               from the 2026-08-03 dump
full-assets/stuff/APK/movescountapp/
    lib/armeabi-v7a/libkomposti-ng.so
    ghidra/libkomposti-ng.so.c                       (49 MB export)
    sources/com/suunto/komposti/*.java
```

What each artifact unlocks:

| Artifact | Role |
|---|---|
| the 9 pcap files | ground truth for the format and the transport; `routedelete` is the minimal complete sequence |
| the 2 GPX files | inputs paired with the captures, required for end-to-end tests |
| `route_simplifier.js` | the simplification loop (the metric was recovered by fitting) |
| `SDSApplicationServer.exe.c` | the arbiter when a field resists; `EmuDevice::writeRoutesBinaryArea` |
| `descr+...+2.4.17` | names and types every SBEM field of the firmware, see `tools/README.md` |
| `libkomposti-ng.so` + Ghidra export | fallback route to the BLE session token (milestone 7) |

Still missing, and never provided: `SUUNTO-V2-HANDOFF.md`, cited six times by
`AMBIT3-SUUNTO-HANDOFF.md` (iOS pairing notes, modern AGPS host). Low value for routes.

## Checking that everything works

Python 3 alone, no dependencies. `gcc` and `libm` for the C part.

```
make -C csrc          # builds the C serializer harness
python3 tools/selftest.py
```

Expected: **19/19**. If `assets/` is absent, the script stops and says where it was
looking. If the C harness is not built, or the SuuntoLink descriptor is missing, it says so
and skips that step.

## The reference watch

Model `Emu` = Ambit3 Peak, serial `<SERIAL>`, **fw 2.4.17**, hw `70.2.17414`,
i.e. `fw_gen = AMBIT3_FW_GEN4` in openambit's `device_driver_ambit3.c`. The 9 captures come
from this watch: **the format is only validated for this model/firmware combination.** On
another model, read region addresses and sizes from the `0x0b21` response rather than
trusting the constants.

## Available hardware, and what it decides

| Machine | Role |
|---|---|
| Lenovo X230, i5-3210M, 16 GB, dual boot Windows 10 + Linux Mint | Windows side has SuuntoLink, so it produces the USBPcap captures. Linux Mint side has openambit, so it is where `write_nav.py` runs and where milestones 4 to 6 happen. x86_64, so also the only host for `qemu-user` on a 32-bit ARM library. |
| MacBook Air M4, 16 GB, macOS Sonoma | Xcode, so milestone 8 (iOS) is no longer blocked on hardware. Apple Silicon has no AArch32, so it cannot run the `armeabi-v7a` libraries even under Rosetta: keep that work on the X230. |
| iPhone 13 mini, iOS 27 beta, Suunto app installed | Target for milestone 8. Not usable for dynamic analysis of the Suunto app without a jailbreak. |
| Suunto Ambit3 Peak + cable | The reference watch. Everything hardware-dependent goes through it. |
| Suunto Kailash + cable | Model `Hoopoe`, fw 2.0.5. Its schema descriptor is in `assets/`, 165 entries against the Ambit3's 324: a free point of comparison when a field resists. |

No Android device yet, and that is the one real constraint: the Frida route of milestone 7
below is unavailable until one arrives. Static analysis of the APK libraries works from any
of these machines; only the dynamic hooking needs Android. Which is another reason to try
the USB whitelist read first.

## What to know about openambit before touching it

Four observed traps that will otherwise cost time:

1. `distance_calc()` (haversine, R = 6367 km, truncation) **does not reproduce** Ambit3
   coordinates: only 124/336 then 508/852 points. The Ambit3 formula is different, see
   `tools/README.md`. Do not reuse that function.
2. `routepoint_start_index` is computed there as `total - count - offset`, which yields the
   **reverse** of what the Ambit3 expects (an increasing running sum).
3. The `legacy_format` parameter of `libambit_protocol_command` derives `send_recv` **and**
   `format` from a single integer. They are independent: the `firmware` capture shows a
   `0x0102` with `send_recv=1` and `format=9`, a combination that enum cannot produce.
4. `libambit_sbem0102_data_add` only writes single-byte lengths. The watch emits an
   extended form (`0xff` followed by a u32) that openambit therefore cannot produce.
   Blocking if you want to write POIs or large sport modes.

On the other hand, reusable as is: `crc16_ccitt_false`, `sha256`,
`libambit_pmem20_data_write` (chunking to a parameterized address, unlike
`ambit_navigation_route_write_to_packs` which hardcodes `0x041EB0`), and
`get_memory_maps()`, which already reads addresses and sizes from the watch.

## Milestones 4 and 5 - real writes, need the watch

Everything is ready on the software side. `tools/write_nav.py` produces exactly the
SuuntoLink bytes: verified payload by payload against `routedelete` (4 payloads) and
`route12km` (12 payloads), SHA-256 hash included. HID framing is proven by a round trip
over **4724 messages and 47117 reports of 64 bytes** re-encoded identically.

Before the first write: **write down by hand the routes and POIs present on the watch**,
because a successful write overwrites the whole navigation database. Also try reading the
region back through the pmem20 path (`0x0b17` with an address): if it works, we gain a real
backup and a `navigation_read`.

Three experiments, in this order, one command each:

```
./tools/write_nav.py reset --compare "assets/ambit3 pcap/routedelete"  # check first
./tools/write_nav.py reset --write                                      # 1. correct hash
```

1. With the correct hash: does the route list empty out?
2. With an arbitrary `u32` at offset 4 of the `0x0b18`: does the watch care? This is the
   only field of the protocol still unidentified.
3. With a deliberately wrong hash: is there a rejection, and is it detectable despite the
   empty response?

Writes return nothing: a rejection is silent. The only check is to read the region hash
back via `0x0b21` afterwards. Then escalate: a regenerated `route12km`, then two routes.

`--write` is explicit and dry-run is the default. A malformed body can reboot or hang the
watch. Never touch the firmware: that is the only write that can brick it.

## Milestone 6 - Android USB-OTG

### Prerequisites to install

Taken from the fork's `android/build.gradle`, recheck in case it moved:

```
minSdkVersion 28   compileSdkVersion 36   targetSdkVersion 36
ndkVersion "27.1.12297006"        react-native 0.84.1
```

- JDK 17, the Android SDK with platform 36 and matching build-tools, and the NDK at exactly
  `27.1.12297006`. The Gradle wrapper ships with the repository, do not install Gradle
  globally.
- Node and npm for the React Native side.
- The fork: `github.com/guiguoz/opensportsync`. Its C core is in
  `android/app/src/main/cpp/libambit/`, and `CMakeLists.txt` is one level up, in
  `android/app/src/main/cpp/`. That core is identical to openambit's for the protocol files.

### Work

1. Copy `csrc/device_driver_ambit3_navigation.{c,h}` into
   `android/app/src/main/cpp/libambit/` and add it to
   `android/app/src/main/cpp/CMakeLists.txt`. The file is written to drop in there
   unmodified: it only depends on `crc16.h` and `sha256.h`, both already present.
2. **Port the simplification to C.** Today it exists only in Python
   (`tools/ambit_simplify.py`, about a hundred lines). It is the only piece of the pipeline
   missing on the native side, and it is indispensable: a GPX with more than 1000 points is
   rejected without it.
3. Write the driver-level function that chains `get_memory_maps()`, the plan's writes via
   `libambit_pmem20_data_write`, each group's `data_tail_len` closure, then the `0x0b04`
   commit. Wire its pointer into the vtable in `device_driver_ambit3.c`, where
   `navigation_write` is `NULL` (around line 130). Careful: `0x0b04` is sent **after** the
   writes, unlike openambit's Ambit2 path which sends it first.
4. Expose it through `jni_bridge.cpp` and `AmbitUsbModule.kt`, then wire a GPX import into
   the React Native UI.
5. Do **not** send the `0x0b25`: it is the watch's complete POI list, and omitting it should
   preserve them. To be confirmed on hardware.

### Verify

Without a device: `make -C csrc test` validates the serializer, and the C port of the
simplification must reproduce 336 points out of 1066 and 852 out of 2911, figures to compare
against the output of `tools/ambit_simplify.py`. With a device: the route shows up and
navigation starts, and the hash read back via `0x0b21` matches.

## Milestone 7 - BLE

### What is known about the transport

GATT service `98ae7120-e62e-11e3-badd-0002a5d5c51b`, write `c6339440-...`, notify
`d0fd6b80-...`, manufacturer id `0x009F` in the advertisement. The BLE frame is a 20-byte
fragmenter with no delimiter and no checksum. Above it, the NSP layer is identical over USB
and BLE: a 12-byte header
`[u8 msgId][u8 subId][u8 flags][u8 errFlags][u16 connId][u16 pktNum][u32 dataSize]`.
The route serializer is the same in both cases.

### The token is probably not derived at all - read it over USB first

Over USB there is **no** login at all. Over BLE there is a login local to the link, not tied
to an account.

The 2026-08-03 SuuntoLink dump supplied the firmware's SBEM schema dictionary
(`assets/descr+<SERIAL>+2.4.17`, see `tools/README.md`). It names entry `0x41` of
the `DeviceSettings` tree:

```
sml.DeviceSettings.WhitelistedBleDevices.Device.DeviceId              uint32
                                              .AddrClass             enum Public/Static/Resolvable/Nonresolvable
                                              .MAC                   utf8
                                              .IdentityResolvingKey  utf8   16 bytes, hex, colon-separated
                                              .EncodingKey           utf8   16 bytes, likewise
                                              .EncodingRnd           utf8   8 bytes, likewise
                                              .EncodingDiv           uint16
                                              .IsAuthenticated       bool
                                              .IsNspCapable          bool
```

That is a Bluetooth LE legacy-pairing bond: LTK, EDIV, Rand, IRK. The watch keeps its own
whitelist, **and the `ambit3full` capture already contains it, in the clear**, with one fully
populated entry for the paired phone (`IsAuthenticated=1`, `IsNspCapable=1`) and one keyless
entry for the HR belt. Read it yourself:

```
./tools/sbem_schema.py --capture "assets/ambit3 pcap/ambit3full" | grep -A1 Whitelisted
```

`EncodingKey` is 16 bytes, the same width as the token `Task::NSP::Login::preparePacket`
copies. The working hypothesis is therefore that there is no derivation to recover: the
token is a per-bond key stored on the watch, and both halves of it are reachable over the
USB channel this project already drives - `0x1100` reads the whole settings tree (payload:
four zero bytes), `0x1101` writes it (the `ambit3full` capture has one).

Test it in this order, cheapest first:

1. Read the live whitelist off the watch over USB and check it matches what the capture
   shows. Non-destructive, read-only, one `0x1100` round trip:

   ```
   ./tools/write_nav.py settings --from "assets/ambit3 pcap/ambit3full"   # rehearsal
   ./tools/write_nav.py settings                                          # the watch
   ```
2. Write a whitelist entry of your own: your phone's MAC, a 16-byte `EncodingKey` you
   choose, `IsNspCapable=1`. Then open BLE and send those same 16 bytes as the NSP LOGIN
   body. If the watch accepts, the milestone is closed with no reverse engineering at all.
3. Only if that fails, go back to recovering a derivation (below).

Caveat: the whitelist is written by the pairing flow, so a key you inject may be overwritten,
and link-layer encryption is a separate matter from the NSP login - Android will not let you
supply an LTK to its own stack without root. What matters here is the NSP body, not the link
key, and those may well be the same 16 bytes.

### Fallback: recover a derivation from the native library

What reading `libkomposti-ng.so.c` established:

- `NspEndDevice::login(this, ptr)`, line 943310 of the export: copies **16 bytes** from its
  argument to `this+0x69`, then starts `Task::NSP::Login`.
- `Task::NSP::Login::preparePacket`, line 1073969: writes `dataSize = 8` at offset 8 of the
  packet then copies **16** bytes at offset `0xc`. The header therefore announces 8 while 16
  bytes go out, which confirms the original handoff.
- `EndDevice::login(void)`, line 742402: a plain stub returning `-1`.

Leads already ruled out, do not redo them:

- The Java layer (`sources/com/suunto/komposti/`, including `BLECentralImpl.java` and
  `SuuntoDeviceServiceWrapper.java`) contains **no** trace of a login, a token, or a
  derivation. It is all native.
- The library's MD5 uses are vendored libcurl/NTLM code (`gethostname`, `DES_cblock`, HTTP
  authentication). Nothing to do with the token, despite the tempting 16-byte coincidence.
- `SDS::WB::BypassRouter::serialKey` decompiles to the static string `"serial"`: a settings
  tree key name, not a cryptographic key.

`NspEndDevice::login` is referenced nowhere but in its own definition: the call goes through
the **vtable**. Three routes, and note that in all of them the function to look at is the
**caller**, not `login` itself, which only copies the 16 bytes it is handed:

1. **Static.** Find the index of `login` in the `NspEndDevice` vtable (search for
   `PTR__NspEndDevice`, around lines 942695 and following in the export), then the indirect
   calls to that slot. The caller builds the 16 bytes.
2. **`libmds.so` from the current Suunto app** (`com.stt.android.suunto`), reported
   2026-08-02 as a symbol-bearing descendant of `libkomposti-ng.so`: still stripped, but the
   dynamic symbol table survives, so `nm -D --defined-only libmds.so | c++filt` gives the
   class map for free, including `EndDevice::login(unsigned char*)`. Hook it with Frida on a
   device rather than emulating it - the caller and the buffer both come out in one shot:

   ```
   Interceptor.attach(Module.findExportByName('libmds.so', '_ZN9EndDevice5loginEPh'), {
     onEnter(args) {
       console.log(hexdump(args[1], {length: 16}));
       console.log(Thread.backtrace(this.context, Backtracer.ACCURATE)
         .map(DebugSymbol.fromAddress).join('\n'));
     }
   });
   ```

   `qemu-arm` on the `armeabi-v7a` build is the weakest of the three: the library is
   bionic-linked, so it needs an Android sysroot pulled off a device, and `login` is a
   non-static member function that wants a constructed `NspEndDevice` for `this`. Calling it
   cold proves nothing. The QEMU plan in `AMBIT3-SUUNTO-HANDOFF.md` Appendix A was for
   dumping `StructMap` to get the route-point encoding, which is done - it no longer applies.
3. **HCI snoop log** on an Android device while the Suunto app connects to the watch: read
   the serial in the `QueryDevice` response and the 16 bytes of the `LOGIN` body. Two
   sessions are enough to tell a fixed key from a derived one. This is also the way to
   confirm or kill the `EncodingKey` hypothesis above.

The token only blocks wireless. Milestone 6's USB-OTG does not need it, and already delivers
a usable application.

## What remains unknown

| Unknown | Risk | How to close it |
|---|---|---|
| The `u32` at offset 4 of the `0x0b18`. Deterministic from the content, but neither a CRC32, nor a sum, nor a size | low, the hash carries verification | experiment 2 of milestone 4 |
| The exact epoch of the route timestamp. Same clock as the waypoint tail date, verified to the second, but the epoch is not a known round date (empirically `1953-11-25T17:31:44`) | low, metadata | hardware |
| `distance`, `ascent`, `descent` in the descriptor: supplied by the application, not re-derivable from a GPX to better than 0.13 % | low | copy or approximate |
| The semantics of the route index's `@12` field, which the `sync` capture contradicts | low, explained by a SuuntoLink bug | hardware |
| Whether the `0x0b25` is mandatory to write a route | medium: determines POI survival | hardware, milestone 4 |
| The BLE session token. Probably `WhitelistedBleDevices.Device.EncodingKey`, readable and writable over USB, rather than something derived | blocks wireless | milestone 7, step 1 |
| The `BlePairingInfo` region declared by the `0x0b21`, 450 bytes at address 1332, never read by SuuntoLink in any capture | low, the whitelist is the shorter path | read it via `0x0b17` |

## The original documents are partly wrong

`AMBIT3-SUUNTO-HANDOFF.md` and the neighbouring files are the preliminary study. They remain
useful for context, but five of their claims have been disproved at the byte level. The
details are in the commit messages and in `tools/README.md`. When a document and a capture
disagree, **the capture wins**. And if a document claims Ghidra or QEMU are needed for point
encoding: that is false, it is done.
