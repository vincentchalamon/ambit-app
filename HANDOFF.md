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
| 4 - first real write (reset) | **done** on hardware 2026-08-04 |
| 5 - first real route | **ready**, needs the watch |
| 6 - Android USB-OTG | **to do**, see below |
| 7 - BLE | **in progress**: GATT roles settled, token hypothesis confirmed on hardware, one open flag |
| 8 - iOS | **unblocked**: a Mac and an iPhone are available, and a plain central is enough |

The binary format of the navigation database is **fully decoded and verified**. The
complete specification is in [`tools/README.md`](tools/README.md): memory map, structures,
coordinate formula, simplification, closing hash, reproduced quirks. Do not duplicate it
here, refer to it.

Everything that is blocked on the watch is blocked on one person, and this document is not
the right shape for them: it explains why rather than what to type. [`RUNBOOK.md`](RUNBOOK.md)
is that list, written to be followed without reading anything else. Keep it in step with the
milestones below.

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

Expected: **20/20**. If `assets/` is absent, the script stops and says where it was
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

The only Android on hand is an Android 4.0 device, which predates BLE central support and
cannot run a current nRF Connect build, so treat it as no Android at all. That is the one
real constraint: the Frida route of milestone 7 below is unavailable until modern Android
hardware arrives, and milestone 6 needs it too. Static analysis of the APK libraries works
from any of these machines; only the dynamic hooking needs a device. Which is another reason
the USB whitelist read came first. The GATT role test was done on the iPhone for the same
reason.

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

**Milestone 4 is done, 2026-08-04.** A reset was written to the real watch and the routes
disappeared from it, so the whole chain holds end to end: framing, payloads, closing hashes and
the commit. Two things the run also settled. The watch's own `0x0b21` reply matched every
region address and size this project had assumed, live, for the first time. And the POIs went
with the routes, which is the subject of point 5 of milestone 6 above and is now fixed.

Milestone 5, a real route, is the same procedure with one more command and has not been run.

`tools/write_nav.py` produces exactly the SuuntoLink bytes: verified payload by payload against
`routedelete` (5 payloads) and `route12km` (13 payloads), SHA-256 hash and POI restore included.
HID framing is proven by a round trip over **4724 messages and 47117 reports of 64 bytes**
re-encoded identically.

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
5. **Do send the `0x0b25`, it is what preserves the POIs.** This document used to say the
   opposite, that omitting the watch's complete POI list would leave it alone. Hardware says
   otherwise: a reset with no `0x0b25` erased every POI on 2026-08-04. Reading `routedelete`
   again with that in mind shows the whole shape of it - SuuntoLink asks for the list with
   `0x0b24`, wipes the database, then writes the list back with `0x0b25`. The wipe is not
   selective, and the last message is the restore. `write_nav.py` now does the same, and its
   `0x0b25` reproduces the capture's byte for byte.

### A POI has no altitude, and it is not an oversight

Asked 2026-08-04, on the theory that the Suunto app's import into the Ambit3 drops it. The
first half is right: the app does have an altitude and the watch does not keep it. The second
half is not, in that there is nowhere for it to go.

The application side carries one. `POST suunto://SDS/LegacyPOI/<serial>` in `suuntoapp.log`
sends, per POI, `{"creation":..., "type":0, "latitude":..., "longitude":..., "name":...,
"altitude":25.3}`, with real values.

The device side has no field for it, three ways:

- the 2.4.17 schema gives `WayPoint` and `PointsOfInterest.PointOfInterest` exactly ten fields
  each, and neither includes an altitude. Every `Altitude` in the whole descriptor belongs to
  the unit setting, the alti/baro profile, the logbook or a live sample;
- the Kailash's 2.0.5 descriptor, a different watch and a much older firmware, declares the
  **identical** ten fields. So this is the family-wide SML definition, not a 2.4.17 quirk -
  which is what André's own note suggested checking, and it holds;
- the 52-byte binary waypoint descriptor is fully accounted for: 4 + 4 + 16 + 16 + 12, and the
  12-byte tail is a magic, five date bytes, a rank, a type and three zero bytes. There is no
  spare field, and the mutation test already covers every written byte.

So SuuntoLink drops the altitude because the destination cannot represent it. Worth stating
plainly in the application: a POI keeps its name and position, and loses its elevation.

**A POI's `Timestamp` is the SDS `creation` field, plain Unix epoch rendered as ISO 8601 in
UTC.** `creation=1774640967` is `2026-03-27T19:49:27`, which is exactly what the watch stores
for that POI, and the same holds for the other three. That closes a small unknown and it is a
welcome contrast with the **route** timestamp, whose epoch was never pinned down and sits in
`ambit_format.ROUTE_TIME_EPOCH` as an empirical `1953-11-25T17:31:44`. Writing a POI needs no
such guess.

Incidentally the SDS list holds more POIs than the watch does, which is the "use on the watch"
toggle doing its job.

**What the watch itself produces, asked 2026-08-04.** Every POI and route examined so far
reached the watch through Komoot, then the Suunto app, then SuuntoLink, so all of it is
evidence about what SuuntoLink writes rather than what the watch can hold. Fair objection, and
the answer splits:

- **An activity the watch records does carry altitude.** Its own logbook index gives
  `Header.Altitude.Min`, `.Max`, `.MinTime`, `.MaxTime` and a plain `Header.Altitude` per move,
  and they are populated: 15 to 21 m on a run, with `Ascent=9` and `Descent=6`. Nothing of
  SuuntoLink's is involved in producing those. `-32768`/`32767` on other moves is the nillable
  sentinel, a recording with no altitude fix.
- **A POI still cannot, whoever writes it.** Read back from the watch, a POI it created
  itself, with a live fix and a barometer at hand, has the same ten fields and no altitude.
  That closes the question: it is the format, not the software.

  The same read answered what SuuntoLink had been hiding, since it writes zero in all four:
  **`Type=17`, `SubType=0`, `TypeIndex=1`, `Flags=1`.** 17 is the value `ambit_format` already
  knew as `WAYPOINT_TYPE_DEFAULT`, the `type` byte of the binary waypoint tail - so the watch
  types its own POI as a Waypoint while SuuntoLink leaves imported ones at 0, and `TypeIndex`
  numbers it, matching the auto-generated name `POI 01`. Those are the values to write when we
  create a POI rather than preserve one. `Flags=1` remains unexplained.

  Coordinates came back as `506236692`, all seven digits, so a POI captured from GPS keeps the
  full precision the record allows; only the typed-coordinate screen is limited to five.

  It also broke something. `parse_sbem_poi_list` used to find the coordinates by skipping runs
  of zero bytes, which only ever worked because those four fields were zero in every capture.
  On this POI it raised `ValueError: subsection not found`. It now uses the known ten-field
  layout, hardcoded rather than read from the schema so that `sbem_schema.py --verify` remains
  an independent check of one against the other.

Still open, and it is the interesting half of the objection: the watch can navigate a recorded
activity, Navigation then Logbook. Whether that materialises an entry in the Routes region, and
therefore a route with per-point altitude that the watch itself filled in, is unknown and worth
a look - `pois` and `logbook` are read-only, so it costs nothing to check before and after.

Decoding a move's samples is a separate job, not started: the logbook index gives each move's
flash range, `0x0b17` reads flash, and `assets/logbook/*.bin` holds ten of them saved by
SuuntoLink, `PMEM` magic, to work against offline.

**Not a casualty of the Suunto app transition**, which is the natural suspicion given how
grudging Ambit support became after Movescount: the watch was nearly abandoned outright and
SuuntoLink is the bare minimum added later, after community pressure. That era is real and this
project has measured it - see `IsNspCapable` above, a capability the old stack maintained and
nothing in the current software sets. But it does not explain this field, for one decisive
reason: **the watch does store altitude, on route points.** The 12-byte point record carries a
u16 at offset 8, and `route128km` fills it on 852 points, from 21 to 182 m. So the navigation
database is not altitude-blind; a route point carries elevation and a POI does not. That is a
scoping decision in the original firmware, in a fixed-stride flash record that the same ten
fields describe on a different product line two firmware generations apart. No application, well
written or not, can add a field the firmware has no room for.

### What the watch can already do with POIs, and what that leaves us

Established on hardware 2026-08-04. A POI can be created three ways without any of our code:

1. on the watch, from the current position;
2. on the watch, by typing a latitude and a longitude;
3. in the Suunto app, by toggling "use on the watch", then syncing by cable with SuuntoLink.

Two things follow for the application. Route 3 is the only exact one, and it depends on both
the Suunto app and SuuntoLink, which is precisely the dependency this project exists to
remove - so writing POIs ourselves is a real feature, not a nicety. It is also not needed to
finish milestone 5, and it has not been built: `write_nav.py` preserves the POIs it finds but
cannot yet create one. The format is fully known, `poiimport` shows a new POI going first in
the list, and there is a byte-exact template to build against, so it is a short job when it
comes up.

Route 2 has a precision limit worth knowing: the watch's own entry screen takes five decimal
places where the record stores seven, so a hand-typed POI lands within about a metre and its
last two digits are zero. Which also means a POI typed on the watch is a cheap way to create a
record with coordinates we chose, if the decoding ever needs checking against a known value.
What the screen displays depends on `sml.DeviceSettings.GpsPositionFormat`, the enum the schema
lists with fifteen values from `WGS84 d` to `NZTM2000`, so do not assume decimal degrees.

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

**GATT roles, settled on hardware 2026-08-03.** Scanned with nRF Connect on an iPhone, watch
idle then with "connect to mobile app" triggered:

- idle, the watch does not advertise at all, nothing is discoverable;
- once connect mode is triggered, the watch appears in the scan list, advertising
  `98ae7120-e62e-11e3-badd-0002a5d5c51b`.

The device that advertises is the peripheral, so **the watch is the peripheral and GATT
server, the phone is the central and client.** That is the ordinary arrangement, merely gated
behind an explicit trigger on the watch instead of advertising at rest.

This **refutes** section 2 of `BLE-FINDINGS-APK-ANALYSIS-2026-08-02_1.md`, which read
`createGattServer` in `AmbitDevice.connect()` as meaning the phone is the server and the watch
the central. Whatever that Java call is for - the phone may well run its own server in
addition, both roles can coexist on one device - it is not the transport that carries NSP.
The field report that produced the scan result above states the opposite conclusion in its
own summary line; the measurement is what counts, and the measurement says peripheral.

Practical consequence, and it is good news: a plain scan-and-connect central is enough.
`CBCentralManager` on iOS, ordinary `BluetoothGatt` on Android. The `CBPeripheralManager`
path that the APK note called for is not needed, which removes the one materially unusual
piece from the iOS design.

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

### Step 1 done on hardware, 2026-08-03: the hypothesis holds

Run against the live watch, before and after re-pairing it with the phone. Values are not
reproduced here, they are link keys; run the command yourself.

| | before re-pair | after re-pair, same phone |
|---|---|---|
| whitelist slots | 8, all empty | 8, one populated |
| `EncodingKey` | none | 16 bytes, **different from the one in `ambit3full`** |
| `EncodingDiv` | 0 | changed too |
| `IsAuthenticated` | 0 | 1 |
| `IsNspCapable` | 1 on every empty slot | **0** on the new bond |

Three things follow. The whitelist had been cleared since `ambit3full` was captured, so that
run was a genuine fresh pairing rather than a stale read. Re-pairing the same phone yields a
**new** `EncodingKey`, readable over USB within seconds and with `IsAuthenticated=1`: the key
is per-bond and not derived from anything stable like the serial, which is what the hypothesis
predicted. And as a side effect the key sitting in `ambit3full` is now stale, so that capture
no longer carries a live secret.

### `IsNspCapable` is not set by pairing, settled 2026-08-04

`IsNspCapable` had flipped from 1 in `ambit3full` to 0 on the new bond, and the cheap reading
was that the flag records how the pairing was made: from the phone's Bluetooth settings rather
than from inside an app that speaks NSP. **Tested by pairing from inside the Suunto app. It
still reads 0.** That reading is dead.

What the third read adds, and it is more useful than the answer itself:

- **The flag is 0 on all eight slots**, the seven empty ones included, where `ambit3full` had
  1 on all eight. So it is not a per-bond attribute a pairing flow sets, it is a table-wide
  value that something changed between the two.
- **SuuntoLink is not what sets it.** Its `0x1101` in `ambit3full` writes the personal
  settings and the eight `Pods`, and no `0x41` at all. Whatever wrote those 1s was something
  else, most likely the Movescount phone app, which did speak NSP over BLE and no longer
  exists.
- **`IdentityResolvingKey` is unchanged across all three reads**, while `EncodingKey` and
  `EncodingRnd` change at every pairing. That is exactly how BLE bonding behaves - the IRK
  belongs to the phone's identity and persists, the LTK is generated per bond - and it is a
  good independent check that these fields really are what the schema says.
- The reply shrank from 589 to 572 bytes, which is the HR belt's 30-byte entry becoming an
  empty 13-byte slot. The parsing accounts for every byte.

So nothing available today sets `IsNspCapable=1`. We have to write it, which makes step 2 not
just the next step but the only one. What remains genuinely unknown is narrower than before:
whether the watch *enforces* the flag when validating a login, or merely records it. Only
attempting the BLE login answers that.

**A useful side effect: `0x1101` accepts a partial tree.** SuuntoLink's write in `ambit3full`
sends 16 entries, not the 66 a read returns. So a whitelist entry can be written on its own,
without having to reproduce the whole settings tree - and there is a byte-exact precedent in
the capture to build it against, the same way the route writes were built.

### Step 2, the only remaining path

Write a whitelist entry through `0x1101`: the phone's MAC, a 16-byte `EncodingKey` of your
choosing, `IsAuthenticated=1` and `IsNspCapable=1`. Then open BLE and send those same 16 bytes
as the NSP LOGIN body. If the watch accepts, the milestone closes with no reverse engineering
at all. If it does not, fall back to recovering a derivation, below.

Note for whoever implements it: the write is the first thing in this project that modifies a
region other than navigation, so build it the same way - produce the payload, diff it against
the capture's `0x1101` for framing, and only then send it.

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

## Milestone 8 - iOS

No longer out of scope: a Mac with Xcode and an iPhone are both available, and the transport
turned out to be ordinary.

What is settled: the watch is the peripheral and GATT server, so a plain `CBCentralManager`
scan-and-connect is enough, and the unusual `CBPeripheralManager` design the APK note called
for is not needed. Scanning finds nothing until "connect to mobile app" is triggered on the
watch, so the UI has to tell the user to do that rather than waiting on an empty scan.

Also observed while testing milestone 7, and the asymmetry matters: **an unpaired watch does
not appear in iOS Settings > Bluetooth at all**, so pairing cannot be started from there. Once
the Suunto app has paired it, it does appear, and can be forgotten from Settings like any other
accessory. So pairing has to be initiated by the app, through CoreBluetooth, which drives the
SMP exchange itself when the peripheral asks for encryption; unpairing can happen in either
place. Our own app therefore owns the pairing flow, and has to cope with the bond vanishing
behind its back when the user forgets the device in Settings.

What is shared with the other transports: everything above the link. The NSP header, the
route serializer and the navigation database layout are identical over USB and BLE, so the
iOS work is a BLE transport plus a bridge to the same serializer, not a second
implementation of the format.

What is not settled: the session token, milestone 7, which blocks any BLE transport
including this one. And deploying to a device needs a signing identity; free provisioning
gives 7-day builds, which is enough to test.

## What remains unknown

| Unknown | Risk | How to close it |
|---|---|---|
| The `u32` at offset 4 of the `0x0b18`. Deterministic from the content, but neither a CRC32, nor a sum, nor a size | low, the hash carries verification | experiment 2 of milestone 4 |
| The exact epoch of the route timestamp. Same clock as the waypoint tail date, verified to the second, but the epoch is not a known round date (empirically `1953-11-25T17:31:44`) | low, metadata | hardware |
| `distance`, `ascent`, `descent` in the descriptor: supplied by the application, not re-derivable from a GPX to better than 0.13 % | low | copy or approximate |
| The semantics of the route index's `@12` field, which the `sync` capture contradicts | low, explained by a SuuntoLink bug | hardware |
| ~~Whether the `0x0b25` is mandatory to write a route~~ | answered 2026-08-04: **it is**. A navigation write erases the POI store, and the `0x0b25` puts it back | done |
| Whether the watch enforces `WhitelistedBleDevices.Device.IsNspCapable` when validating a login, or merely records it. Pairing does not set it, from inside the Suunto app or outside, and SuuntoLink never writes that entry, so it has to be written through `0x1101` | blocks wireless | milestone 7, step 2: write the entry, then attempt the BLE login |
| The `BlePairingInfo` region declared by the `0x0b21`, 450 bytes at address 1332, never read by SuuntoLink in any capture | low, the whitelist is the shorter path | read it via `0x0b17` |

## The original documents are partly wrong

`AMBIT3-SUUNTO-HANDOFF.md` and the neighbouring files are the preliminary study. They remain
useful for context, but five of their claims have been disproved at the byte level. The
details are in the commit messages and in `tools/README.md`. When a document and a capture
disagree, **the capture wins**. And if a document claims Ghidra or QEMU are needed for point
encoding: that is false, it is done.
