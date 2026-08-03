# Review of the 2026-08-02 notes, and what the SuuntoLink dump actually unlocked

For André. Answers `ROUTE_ENCODING_UPDATE-2026-08-01.md` and
`BLE-FINDINGS-APK-ANALYSIS-2026-08-02_1.md`.

Thanks for both notes and for the dump. Honest verdict: the dump is a very good find, but
not for the reason the route note gives. Details below, then what is left to do, and above
all what only you can do given the watch is at your place.

Everything asserted here was checked against the 9 pcap captures in `assets/ambit3 pcap/`;
the commands to reproduce each check are inline.

---

## 1. The route encoding note: do not apply it

Your Claude worked from `ROUTE_ENCODING_SUMMARY.md`, which is the preliminary study. It has
been out of date for two months, superseded by `tools/README.md`. Its "coordinate scale
unsolved" was closed and verified a while ago, and the `int32 = degrees x 1e7` figure has
been in the code since:

```
grep -n "1e7" tools/ambit_format.py tools/build_route.py
```

One thing in particular not to apply: the note recommends replacing "the placeholder/guessed
scale factor in the route point encoding logic" with `degrees * 1e7`. That is not how a route
point is encoded. It is a fixed 12-byte record:

```
0  i32 x_rel        metres east of the bbox centre, signed
4  i32 y_rel        metres north of the bbox centre, signed
8  u16 altitude     metres, 30000 = no data
10 u16 rel_distance 0 at the first point, 65535 at the last
```

with `R = 10800 * 1852 / pi`. Verified on 1188 points, and the point body is byte-for-byte
identical to `route12km` and `route128km`. The `x 1e7` scale is the one used by **waypoints
and POIs**, a different structure in a different flash region. Applying it to route points
would break a working serializer.

The rest of the note is accurate, but already known. The reading of the `<MOD>` radian
formula is correct: it is SML-internal bookkeeping, it does not reach the wire.

## 2. What the dump really unlocks: `descr+<SERIAL>+2.4.17`

That file is not a lat/lon lookup. It is the firmware's whole SML schema: 278 typed fields,
27 record layouts, 19 queryable objects. And the important part, which the note does not
pick up: **its field numbers are exactly the entry identifiers that travel in SBEM payloads
on the wire.** So every SBEM message in the protocol becomes readable without guessing.

New tool in the repo:

```
./tools/sbem_schema.py                                       # dump of the schema
./tools/sbem_schema.py --group 85                            # detail of one id
./tools/sbem_schema.py --capture "assets/ambit3 pcap/poiimport"
./tools/sbem_schema.py --verify                              # 110/110 payloads conform
```

Verified across the 9 captures: `0x0b21` carries only `0x4a` =
`BinaryDataArea{Name,Checksum,Address,Size}`, `0x0b24` and `0x0b25` only `0x55` =
`WayPoint`, `0x1100` the 66 `DeviceSettings` groups, `0x1200` the `DeviceLogBook` entries. A
request is just the identifier of the object you want.

Three consequences:

- **The region table is nine entries, not three**: `Waypoints`, `Routes`, `Apps`, `GpsSGEE`,
  `CustomModes`, `TrainingProgram`, `ExerciseLog`, `EventLog`, `BlePairingInfo`.
- **The logbook index decodes completely**, 47 named fields per activity including
  `MemArea.StartAddress1` / `EndAddress1`, that is the flash address of every move. The whole
  read-activities path, handed over. Your 11 moves are in `poiimport`, with duration,
  distance, HR, activity type and the rest.
- **The POI record is named**: `Name, RouteName, Timestamp, RouteIndex, Type, SubType,
  TypeIndex, Flags, Latitude, Longitude`. What our parser treated as padding is an empty
  `RouteName` plus five typed bytes. It reads the right coordinates on every capture we have
  because those five bytes are zero there, but it would misalign on a typed POI.

The 10 `logbook/*.bin` files in the dump (`PMEM` magic) are a decode target for activities
that needs no capture and no hardware.

## 3. The BLE milestone: the answer may have been in our captures all along

This is the most useful find in the dump. The descriptor names entry `0x41` of the settings
tree:

```
sml.DeviceSettings.WhitelistedBleDevices.Device.DeviceId              uint32
                                              .AddrClass             enum Public/Static/...
                                              .MAC                   utf8
                                              .IdentityResolvingKey  utf8    16 bytes
                                              .EncodingKey           utf8    16 bytes
                                              .EncodingRnd           utf8     8 bytes
                                              .EncodingDiv           uint16
                                              .IsAuthenticated       bool
                                              .IsNspCapable          bool
```

LTK, EDIV, Rand, IRK: a Bluetooth LE legacy-pairing bond, which **the watch stores itself**,
in a settings tree that `0x1100` reads over USB with a four-zero-byte payload and that
`0x1101` writes.

And the `ambit3full` capture has been carrying the populated entry since day one:

```
./tools/sbem_schema.py --capture "assets/ambit3 pcap/ambit3full" | grep -A1 Whitelisted
```

One complete entry for the paired phone (`IsAuthenticated=1`, `IsNspCapable=1`, a 16-byte
`EncodingKey`) and one keyless entry for the HR belt. Incidentally: those are your MAC and
your BLE link keys, in the clear. The captures are not versioned, and it has to stay that
way.

`Task::NSP::Login::preparePacket` copies 16 bytes. `EncodingKey` is 16 bytes. Hence the
hypothesis: **there may be no derivation to recover at all.** The token would be a per-bond
key stored in the watch, and both operations, reading it and writing it, go through the USB
channel we already drive completely.

It is only a hypothesis. It is tested in this order, cheapest first:

1. A `0x1100` round trip against the watch, compared with the capture. Non-destructive.
2. Write your own entry: your phone's MAC, an `EncodingKey` you choose, `IsNspCapable=1`.
   Then open BLE and send those same 16 bytes as the NSP LOGIN body. If the watch accepts,
   milestone 7 is closed with no reverse engineering.
3. Only if that fails, go back to looking for a derivation.

Caveats, stated plainly: a key you inject may be overwritten by the pairing flow, and Android
will not let you hand an LTK to its own stack without root. But the NSP LOGIN body and the
link key are two distinct things, and it is the body we need.

A 450-byte curiosity: the `BlePairingInfo` region at address 1332, which SuuntoLink never
reads in any capture.

## 4. The APK note, point by point

**The GATT UUIDs.** Confirmation by a shipping app of what section 8.1 of the handoff said.
Useful.

**Reversed GATT role, phone as server and watch as central.** This is the most interesting
point in the note, and it is worth more than a static reading, because it settles in two
minutes: open nRF Connect on a phone and scan. If the watch is the one advertising
`98ae7120-e62e-11e3-badd-0002a5d5c51b`, it is the peripheral and the note is wrong. If
nothing shows up until you start "connect to mobile app" on the watch and it is the phone
that must advertise, the note is right. Settle it that way before scoping anything on
`CBPeripheralManager` for iOS. Note that the handoff records manufacturer id `0x009F` in
**the advertisement**, which sits badly with a phone as advertiser: one of the two is
imprecise.

**`libmds.so` as a better target.** Agreed, and for the right reason: a stripped C++ library
keeps its dynamic symbol table, its typeinfo and its vtable symbols. That is a real
improvement over `libkomposti-ng.so`.

**"Direct hit: the login function is exported".** This is the point to correct. `login` is
not where the answer is. We had already read the old library: `NspEndDevice::login(this,
ptr)` copies the 16 bytes **from its argument** into `this+0x69` then starts the task. It
does not build them. Decompiling `EndDevice::login` will show a `memcpy`. The target is the
**caller**, and that is exactly where the symbol-bearing library helps: the
`vtable for NspEndDevice` symbol gives you `login`'s slot index, and the indirect calls
through that slot give you the caller.

## 5. The commands you asked for, and why QEMU comes third

### Step 0 - symbols only, no emulation, two minutes

```bash
unzip -o com.stt.android.suunto_6.7.12.apk 'lib/*' -d apk
nm -D --defined-only apk/lib/arm64-v8a/libmds.so | c++filt | sort > syms.txt
grep -nE 'login|[Tt]oken|Whitelist|Encoding|[Pp]air|[Bb]ond|Task::NSP' syms.txt

# the reason this library is worth the trouble: the vtables survive
nm -D apk/lib/arm64-v8a/libmds.so | c++filt | grep -E "vtable for (Nsp)?EndDevice"
nm -D apk/lib/arm64-v8a/libmds.so | c++filt | grep -E "typeinfo for .*EndDevice"
```

Then in Ghidra: open the vtable at that address, count to `login`'s slot, and look for the
indirect calls through that offset.

### Step 1 - Frida on a device, the fastest dynamic answer

Gets you the 16 bytes **and** the caller in one shot, and needs no sysroot.

```bash
pip install frida-tools
# rooted device, frida-server of the matching ABI:
adb push frida-server-android-arm64 /data/local/tmp/frida-server
adb shell 'su -c "chmod 755 /data/local/tmp/frida-server; /data/local/tmp/frida-server &"'
frida-ps -Uai | grep -i suunto
frida -U -f com.stt.android.suunto -l login.js --no-pause
```

`login.js`:

```javascript
const sym = Module.findExportByName('libmds.so', '_ZN9EndDevice5loginEPh');
console.log('EndDevice::login @', sym);
Interceptor.attach(sym, {
  onEnter(args) {
    console.log('token:\n' + hexdump(args[1], { length: 16 }));
    console.log(Thread.backtrace(this.context, Backtracer.ACCURATE)
      .map(DebugSymbol.fromAddress).join('\n'));
  }
});
```

Without root: repackage with frida-gadget (`objection patchapk -s app.apk`), same script.

If the modern app no longer pairs with an Ambit3 at all, this route dies and we fall back to
the HCI snoop log: developer options, enable Bluetooth HCI snoop log, then
`adb pull /data/misc/bluetooth/logs/btsnoop_hci.log`, open in Wireshark, filter `btatt`. Two
sessions are enough to tell a fixed key from a derived one. That is also the way to confirm
or kill the `EncodingKey` hypothesis of section 3.

### Step 2 - qemu-user on the 32-bit build

This is what you asked for. It works, but read the caveat at the end before spending an
evening on it.

```bash
sudo apt install qemu-user qemu-user-static

# 1. an Android sysroot. The NDK's libc.so is a link-time stub and will not run:
#    you need real bionic, pulled off a device or an AVD that still ships 32-bit libs
#    (API <= 30; 64-bit-only phones have no /system/lib).
mkdir -p sysroot/system/lib sysroot/system/bin
adb pull /system/lib/. sysroot/system/lib/
adb pull /system/bin/linker sysroot/system/bin/
cp apk/lib/armeabi-v7a/*.so sysroot/system/lib/

# 2. a harness, built for the same ABI
cat > call_login.c <<'EOF'
#include <dlfcn.h>
#include <stdio.h>
int main(void) {
    void *h = dlopen("libmds.so", RTLD_NOW);
    if (!h) { printf("dlopen: %s\n", dlerror()); return 1; }
    void *f = dlsym(h, "_ZN9EndDevice5loginEPh");
    printf("EndDevice::login = %p\n", f);
    return 0;
}
EOF
"$NDK/toolchains/llvm/prebuilt/linux-x86_64/bin/armv7a-linux-androideabi21-clang" \
    -o sysroot/call_login call_login.c

# 3. run it
QEMU_LD_PREFIX=$PWD/sysroot qemu-arm -E LD_LIBRARY_PATH=/system/lib ./sysroot/call_login
```

Caveat, and the reason it is ranked last: `EndDevice::login` is a non-static member function.
Calling it needs a constructed `NspEndDevice` as `this`, with a valid vtable and plausible
internal state, or it faults. And even when it runs, it copies the buffer you handed it: it
cannot tell you where that buffer came from. QEMU is the right tool for calling a leaf
function with a known input and reading its output. That is not the shape of this problem.

For the record: the QEMU plan in appendix A of `AMBIT3-SUUNTO-HANDOFF.md` existed to dump
`Communist::Serialization::StructMap` and recover the route-point encoding. That is done, by
fitting against the captures. The appendix no longer applies to anything.

## 6. Two things in the dump that were in neither note

### AGPS is still reachable without an account, tested today

`AMBIT3-SUUNTO-HANDOFF.md` already had the host and the app key, with the path marked
`not yet captured (TODO)` twice. It does not need a capture: the decompiled server has it,
along with the query parameter.

```
grep -n "gpsorbit\|?appkey=" assets/SDSApplicationServer_exe.c
```

```
https://devices.suunto-operations.com/devices/gpsorbit/binary?appkey=<key>      -> sgee.7d
https://devices.suunto-operations.com/devices/glonassorbit/binary?appkey=<key>  -> glonass.7d
```

The key is the static string logged in `suuntoapp.log` next to every
`GET suunto://SDS/SGEE/Version`. Not an OAuth token, no account, the same endpoint that still
serves firmware per `library.xml`.

Tested 2026-08-03, both answer:

```
/devices/gpsorbit/binary        HTTP 200   72016 bytes
/devices/glonassorbit/binary    HTTP 200   45171 bytes
```

Same magic as `sgee.7d` (`62 12 37 09`), and the header carries the build date: `07 ea 08 03`,
big-endian year then month and day, so 2026-08-03, against `07 ea 07 1f` in the dumped file,
which is the day you downloaded it. Fresh data, one GET. So AGPS is a feature the app can
ship, not a dead end. 72016 bytes fit inside the 140000 of the `GpsSGEE` region, but the size
drifts (41350 in the `orbitsync` capture), so nothing to hardcode.

Same decompilation, line 57601: a second path built from `"/devices/"` plus a codename, a
`pkgId` and a version, which is Movescount's historical firmware pattern moved to the new
host, `/devices/<Codename>/<pkgId>/<swVersion>/binary?appkey=`. Untested and not needed, the
dump already carries `Emu-fw_2.4.17-70.2.17414.zip`, but that closes the other half of the
same TODO.

The rest of the AGPS flow is confirmed by your logs rather than inferred:
`EmuDevice::updateSgeeFile: found sgee area named GpsSGEE` then `succeeded to location
460000`, which is `0x0704E0`, exactly the region the captures write. And the freshness check
is server-side: `GET /SGEE/Version` answers `"Available": true|false`. That is what made
`orbitsync2` a 384-packet no-op.

### Your Kailash

`library.xml` declares two watches, and the dump also contains
`descr+<SERIAL>+2.0.5`, the Kailash schema (model `Hoopoe`, fw 2.0.5): 165 entries
against 324. The parser reads it unmodified. Not a priority, but it is a free point of
comparison if an Ambit3 field ever resists.

## 7. What only you can do, in order

The watch is at your place, so everything below is blocked on you. All of it runs on the
**Linux Mint side of the X230**: Python 3 and `hid` (`pip install hid`, plus the hidapi
system package), no build step. openambit being already installed there, its udev rules
should already give you the watch without root; if not, that is the thing to check first.

The Windows side stays useful for what it is good at: producing USBPcap captures with
SuuntoLink. Nothing below needs Windows, Android, or a VM.

1. **Read the BLE whitelist over USB.** New in the repo, read-only, no `--write` accepted.
   Rehearse on the capture first so you know what a good output looks like, then plug the
   watch:

   ```
   ./tools/write_nav.py settings --from "assets/ambit3 pcap/ambit3full"
   ./tools/write_nav.py settings
   ```

   The first command works today and already prints your bond. The second is the actual
   test. What we want to know: does the live watch still report the same `EncodingKey`, and
   is `IsAuthenticated` still 1. That validates or kills section 3 in a few minutes.

   Honest caveat: the reassembly of a 589-byte reply is exercised by `--from`, but the live
   USB read path in `write_nav.py` has never run against hardware. If it hangs or returns
   short, that is the tool, not the watch, and it is a quick fix. `--verbose` dumps the
   64-byte reports.

2. **An nRF Connect scan** to settle the GATT role of section 4. Two minutes, any phone.
3. **Milestones 4 and 5, the first real writes.** Everything is ready on the software side,
   `tools/write_nav.py` produces exactly SuuntoLink's bytes, verified payload by payload.
   Before the first write: **write down by hand the routes and POIs present on the watch**, a
   successful write overwrites the whole navigation database. Dry-run is the default,
   `--write` is explicit. Never touch the firmware, that is the only write that can brick the
   watch. The three experiments to run are detailed in `HANDOFF.md`.
4. If you want to keep going on the RE: step 0 of section 5, the symbol and vtable triage,
   on the caller and not on `login`. It needs no device and runs anywhere.

Two notes on the machines you listed.

**No Android means step 1 of section 5 is unavailable.** Frida needs an Android device; the
iPhone cannot be hooked without a jailbreak. So until Android hardware arrives, the dynamic
route to the token is closed and the USB whitelist read is the only way forward on
milestone 7. Which is convenient, since it is also the cheapest.

**The Mac and the iPhone unblock milestone 8.** `HANDOFF.md` had iOS out of scope for want
of Xcode, and that is no longer true. Worth knowing before you start: per section 4 the
transport probably needs `CBPeripheralManager`, not `CBCentralManager`, so settle the GATT
role first. And the M4 has no AArch32, so it cannot run the `armeabi-v7a` libraries even
under Rosetta: keep any QEMU work on the X230, which is what appendix A of the handoff
assumed anyway.

## 8. Housekeeping: what to remove, what to rotate

**`suuntolink_data.json`: revoke, then delete or placeholder.** Your note is right. Those are
live OAuth tokens for your Suunto account, so editing the file is not sufficient by itself:
revoke them first in the account's connected-apps page. Then, before the dump is shared
again, either drop that single file from the zip or replace the two token values with
`REDACTED`. Nothing in this project needs it.

**`ambit3full`: do not redact the capture, re-pair the watch.** It carries your phone's MAC
and the pairing's link keys in the clear, and the `settings` command prints them by design,
since reading that key is the whole point. But patching the pcap is the wrong fix twice over:

- it is byte-exact ground truth for the entire format, and the selftest leans on it;
- the HID frames carry a CRC16 over their payload, and nothing in our tooling verifies
  incoming CRCs, so edited bytes would pass silently and leave a capture whose checksums are
  quietly wrong. That is a worse artifact than an honest one kept private.

The equivalent of rotating a credential, for a BLE bond, is to **unpair and re-pair the watch
with the phone**: the watch generates a fresh `EncodingKey` and the copy in the capture
becomes worthless. Two notes on that:

- **Order matters.** Run the `settings` read of section 7 *before* re-pairing, or the live
  value will no longer match the capture and you lose the comparison that is the point of the
  test.
- Re-pairing afterwards is a free extra experiment. If `EncodingKey` changes, that on its own
  supports the per-bond-key reading of section 3; if it does not change, the hypothesis is in
  trouble and that is worth knowing early.

Neither the captures nor the SuuntoLink dump is in the repository, and `.gitignore` keeps them
out. That is the real protection, not redaction.

**And on our side, which matters more.** The repository is public, and it was carrying two
things that had no business being there: the reference watch's serial number, spelled out in
several documents and hardcoded in a tool path, and Suunto's static app key. Both are now
`<SERIAL>` and `<APPKEY>`; `tools/sbem_schema.py` locates the descriptor by globbing
`descr+*+2.4.17` rather than naming the serial, which also makes it work for any watch. No
personal data of yours is left in a tracked file.

The values remain in the git history of a public repository. Removing them from history means
a rewrite and a force-push, which is a separate decision.
