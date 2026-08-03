# Summary for André - 2026-08-03

Short version of `SUUNTOLINK-DUMP-REVIEW-2026-08-03.md`, which has the reasoning and the
commands. `HANDOFF.md` remains the project state of record.

## Your two notes

**The route encoding one: nothing to merge, and one thing not to apply.** It answers
`ROUTE_ENCODING_SUMMARY.md`, which is the preliminary study, out of date for two months and
superseded by `tools/README.md`. The `int32 = degrees x 1e7` scale was closed a while ago and
is already in the code. It is the scale of waypoints and POIs; a route point is a 12-byte
record of metre offsets from the bbox centre, verified on 1188 points and byte-exact against
the captures. So do not replace the route point encoding with it, that would break a working
serializer. Everything else in the note is correct but already known.

**The APK one: two good findings and one to correct.** The GATT UUIDs are a useful live
confirmation. The reversed GATT role, phone as server, is the most valuable claim in the note
and is worth settling with a scan rather than statically. `libmds.so` is genuinely a better
target than `libkomposti-ng.so`, because a stripped C++ library keeps its dynamic symbols,
typeinfo and vtables. But `EndDevice::login` is the wrong function: we had already read that
it copies the 16 bytes **from its argument**. It does not build them. The target is the
caller, and the vtable symbol is what gets you there.

## What the dump actually unlocked

`descr+<SERIAL>+2.4.17` is not a lat/lon lookup. It is the firmware's whole SML
schema, 278 typed fields and 27 record layouts, **and its field numbers are the entry
identifiers that travel in SBEM payloads on the wire**. Every SBEM message of the protocol is
now readable by name. Verified on all 9 captures, 110 payloads.

Three consequences: the `0x0b21` declares nine flash regions and not three; the logbook index
decodes completely, 47 fields per activity including the flash address of every move, which
is the whole read-activities path; and the `DeviceSettings` tree is readable and writable over
USB, including `WhitelistedBleDevices`.

That last one is the important one. `WhitelistedBleDevices.Device` carries a BLE legacy
pairing bond: LTK, EDIV, Rand, IRK, plus the paired phone's MAC and an `IsNspCapable` flag.
The watch stores it itself, and the `ambit3full` capture has been carrying your populated
entry since day one. `EncodingKey` is 16 bytes, exactly the width of the token
`Task::NSP::Login` copies.

So the working hypothesis for milestone 7 is that **there is no derivation to recover**: the
session token would be a per-bond key stored in the watch, and both reading it and writing it
go through the USB channel we already drive. It is a hypothesis, not a result, and it is
cheap to test.

Also closed: the AGPS TODO. `/devices/gpsorbit/binary?appkey=` on
`devices.suunto-operations.com`, tested 2026-08-03, HTTP 200, 72016 bytes dated the same day,
no account. AGPS is a feature the app can ship.

## New in the repo

- `tools/sbem_schema.py` - loads the descriptor, names and decodes any SBEM payload.
- `tools/write_nav.py settings` - read-only `0x1100` query, decodes the settings tree.
- `HANDOFF.md` milestone 7 rewritten; the whole repository is now in English.
- `python3 tools/selftest.py` is at 19/19.

## What we need from you, in order

All of it on the **Linux Mint side of the X230**. Python 3 and `hid`, no build step. openambit
being installed there, its udev rules should already give you the watch without root; if not,
that is the first thing to check. Windows keeps its job: producing USBPcap captures with
SuuntoLink. Nothing here needs Android or a VM.

**1. Read the BLE whitelist off the watch.** Read-only, a few minutes, and it decides
milestone 7. Rehearse on the capture first so you know what a good output looks like:

```
./tools/write_nav.py settings --from "assets/ambit3 pcap/ambit3full"
./tools/write_nav.py settings
```

The question is whether the live watch still reports the same `EncodingKey` with
`IsAuthenticated=1`. Caveat worth stating: the reassembly of the 589-byte reply is exercised
by `--from`, but the live USB read path has never run against hardware. If it hangs or
returns short, that is our tool and a quick fix, not the watch. `--verbose` dumps the raw
64-byte reports.

**2. Settle the GATT role.** nRF Connect on any phone, two minutes. If the watch is the one
advertising `98ae7120-e62e-11e3-badd-0002a5d5c51b`, it is the peripheral and your note's
section 2 is wrong. If nothing appears until you start "connect to mobile app" on the watch,
the note is right. This decides the iOS design before anyone writes a line of it.

**3. Milestones 4 and 5, the first real writes.** Everything is ready in software;
`write_nav.py` produces exactly SuuntoLink's bytes, verified payload by payload against the
captures. Before the first write, **write down by hand the routes and POIs on the watch**: a
successful write overwrites the whole navigation database. Dry-run is the default, `--write`
is explicit, and never touch the firmware - that is the only write that can brick it. The
three experiments are in `HANDOFF.md`.

**4. If you want to keep reversing**, do the symbol and vtable triage on `libmds.so` (section
5, step 0 of the review). No device needed, runs anywhere, and it targets the caller.

## Your hardware, three consequences

- **No Android closes the Frida route** to the token until you get one. The iPhone cannot be
  hooked without a jailbreak. So the USB whitelist read is currently the only way forward on
  milestone 7 - which is also the cheapest, so no loss.
- **The Mac and the iPhone unblock milestone 8.** iOS was out of scope for want of Xcode and
  no longer is. Settle the GATT role first: per your own note it likely needs
  `CBPeripheralManager`, which is a materially different design from a normal BLE integration.
- **The M4 has no AArch32.** It cannot run the `armeabi-v7a` libraries, not even under
  Rosetta. Keep any QEMU work on the X230, which is what the handoff's appendix A assumed.

## Housekeeping: what to remove, what to rotate

**The OAuth tokens in `suuntolink_data.json` - revoke them.** As your own note said. They are
live credentials to your Suunto account, so editing the file is not enough on its own: revoke
first, in the account's connected-apps page. Then, before the dump is shared again, either
delete that one file from the zip or replace the two token values with `REDACTED`. It is not
needed for anything we do.

**The BLE bond in `ambit3full` - do not edit the capture, re-pair the watch instead.** The
capture carries your phone's MAC and the pairing's link keys in the clear. Redacting them
inside the pcap is the wrong fix twice over: that file is byte-exact ground truth for the
whole format, and the HID frames carry a CRC16 over their payload, so patched bytes would
leave a capture whose checksums no longer match - a trap for whoever reads it next.

The equivalent of rotating, here, is to **unpair and re-pair the watch with your phone**. The
watch then generates a fresh `EncodingKey`, and the copy sitting in the capture is worth
nothing. **Order matters:** run the `settings` read *before* re-pairing, otherwise the live
value will no longer match the capture and you lose the comparison. Re-pairing afterwards is
in fact a bonus experiment: if the key changes, that alone supports the per-bond-key
hypothesis.

Everything else stays as it is. The captures and the SuuntoLink dump are not in the
repository and `.gitignore` keeps them out; that is the real protection.

**On our side.** The repository is public, and it was carrying two things it should not: your
watch's serial number, written out in the docs and even hardcoded in a tool path, and Suunto's
static app key. Both are now placeholders, `<SERIAL>` and `<APPKEY>`, and the schema tool
finds the descriptor by globbing instead of naming the serial. Nothing personal of yours is
left in a tracked file. The values do remain in the git history of a public repo, which is a
separate decision, and Vincent's to make.
