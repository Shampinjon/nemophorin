# Semaphorin --diagnose: audit, findings, and the iPhone X (A11) tethered-downgrade bug

This document covers the work done on branch `claude/wizardly-cray-167egh`:
a source audit of `semaphorin.sh`, a new safe `--diagnose` mode, and a
targeted fix to the error handling in the `--restore` backup step where the
reported APFS/SEP failures actually happen.

Backup of the pre-change code: branch `backup/pre-diagnose-audit` and tag
`backup-pre-diagnose-audit` (both point at the original commit `26ddd36`,
before any of this work).

## 1. What's in this repo vs. what isn't

`semaphorin.sh` (~4,800 lines) is the entire tool: a single bash script that
drives `irecovery`/`gaster`/`img4`/`img4tool`/etc. (prebuilt binaries under
`Darwin/` and `Linux/`) to pwn the device with checkm8, build and boot a
signed SSH ramdisk, and then drive the restore over SSH/SCP. There is no
PongoOS or SEP C source in this repository - `src/drivers/sep/sep.c` from
checkra1n/PongoOS does not exist here, because Semaphorin does not use
PongoOS as its primary boot path; it boots Apple's own `RestoreRamDisk.dmg`
(patched by `kairos`/`iBoot64Patcher`) via `irecovery`, built from the
`ramdisk.img4` assembled in `_download_ramdisk_boot_files()`
(`semaphorin.sh:355` onward in the pre-change code). PongoOS is only invoked
for the `16.x`/`17.x` kernelcache path (the `pongo=1` branches), which does
not apply to this downgrade chain (16.7.16 is reached via a full restore,
not this ramdisk).

Likewise, `mount_apfs`, `mount_filesystems`, and `seputil` are **not**
Semaphorin binaries - they are Apple's own tools baked into
`RestoreRamDisk.dmg` for the matching iOS version (10.3.3 / 12.5.4 / 14.3
depending on device, selected in the big `if`/`elif` chain starting around
`_download_ramdisk_boot_files` call sites). They cannot be patched from this
repo; what *can* be fixed here is how `semaphorin.sh` calls them, checks
their result, and reports failure.

## 2. Where the reported errors come from

The exact sequence in the bug report -
`mount_apfs: volume could not be mounted` /
`seputil: Gigalocker initialization completed` /
`seputil: load fw returned 0xe00002d5` / SSH drop - happens during the
**backup step of `--restore`**, before any wipe: Semaphorin boots the SSH
ramdisk, then runs (previously, pre-fix):

```bash
"$bin"/sshpass ... ssh ... "bash -c mount_filesystems" 2> /dev/null
...
"$bin"/sshpass ... scp ... root@localhost:/mnt1/System/Library/Caches/apticket.der ...
"$bin"/sshpass ... scp ... root@localhost:/mnt1/usr/standalone/firmware/sep-firmware.img4 ...
```

Three bugs compounded here, all now fixed on this branch:

1. **`mount_filesystems`'s stderr (and the APFS/SEP diagnostics in it) was
   thrown away** (`2> /dev/null`), which is exactly why the user only sees
   those lines on the device's own console/serial output and not in
   Semaphorin's own log - the script was actively discarding them.
2. **The script never checked whether `/mnt1`/`/mnt2` actually mounted**
   before immediately trying to `scp` `apticket.der` and
   `sep-firmware.img4` from them. If the mount failed, every subsequent
   `scp` also silently failed (also redirected to `/dev/null`), and the
   script only noticed at the very end, by which point it just printed a
   generic "An error occured" and rebooted the device - with no indication
   of *where* in the chain things broke.
3. **No distinction between an SSH-level disconnect (link gone) and the
   remote command simply returning an error.** Both looked identical to the
   script (output eaten, exit code mostly ignored), so there was no way to
   tell "mount_apfs failed" from "the device panicked and dropped SSH" from
   the script's own output.

This matches the user's experience: the mount failed, the backup step kept
going anyway into the SEP/activation-record copy logic, and the visible
symptom (SSH dropping, blank screen) is a plausible result of a ramdisk in
a half-mounted, inconsistent state rather than necessarily an actual SEP
panic - but **this could not be told apart with the old code**, which is
precisely the ambiguity flagged in the task ("нельзя заранее считать SEP
единственной причиной").

## 3. SEP-specific analysis (`seputil`, `0xe00002d5`)

What we can say without physical access to the device:

- `0xe00002d5` is an SEP/AppleKeyStore-style status returned by `seputil`'s
  firmware-load path. It is not a generic Semaphorin error string - it
  comes straight from the SEP ROM/firmware's own response to the load
  request, reported by the (Apple) `seputil` binary.
- The sequence **"Gigalocker initialization completed" then a non-zero
  `load fw` result** means `seputil` did get far enough to talk to the SEP
  and initialize its keystore/Gigalocker state, but the actual firmware
  image it tried to hand the SEP was rejected. That is consistent with
  either (a) a `sep-firmware.img4` that doesn't match what the currently
  active Boot Manifest/SEP epoch on the device expects (quite plausible
  right after a *tethered* downgrade via surrealra1n to 14.3, since a
  tethered boot does not go through Apple's normal signed-restore path and
  can leave the SEP's own notion of "current" firmware/anti-replay state
  out of step with what a stock 14.3 `apticket`/SEP firmware pairing
  assumes), or (b) the mount of `/mnt1` having already failed/returned
  stale or partial data, so the `sep-firmware.img4` `seputil` is being
  pointed at is not the one actually intended.
- On the 14.3-after-surrealra1n run, the log is shorter - it stops right
  after "GigaLocker initialization completed" with no `0xe00002d5` line
  at all before the SSH drop. That is a meaningfully different failure
  point: either `seputil` never got to the "load fw" call (crashed/hung
  before issuing it), or the message was in flight when SSH dropped and
  the terminal never printed it. **These are not the same bug**, and the
  old code's total silence on both (`2>/dev/null`, no result reported from
  that statement at all) is why they looked identical to the user. The new
  `--diagnose` mode's `_diag_ssh` wrapper distinguishes "the remote command
  returned an error" from "the SSH link itself died" (exit 255), which is
  the first step toward telling these two situations apart on real
  hardware.
- We deliberately did **not** try to invoke `seputil` directly from
  `--diagnose`, even read-only/"--help"-style, because there is no public
  documentation of its argv handling and the task explicitly calls for not
  forcing SEP firmware load automatically. If `seputil`'s no-argument
  behavior is confirmed safe on real hardware (e.g. by someone manually
  running it over the existing SSH ramdisk and observing it only prints
  usage), that could be added to `--diagnose` as a passive metadata check
  in a follow-up change - but it should not be assumed safe without that
  confirmation.
- **This is very likely a ramdisk/seputil/SEP-firmware compatibility
  problem, not a `semaphorin.sh` logic bug** - fixing the actual
  `0xe00002d5` will most likely require either a different/matching
  ramdisk+SEP-firmware pairing for the *current* (post-surrealra1n, 14.3)
  tethered state, or accepting that a full untethered SEP downgrade this
  way isn't possible and the restore path needs to go through whatever
  SEP-compatible route surrealra1n/palera1n itself uses for 14.3 A11
  devices. That is outside what a bash script talking to Apple's own
  `seputil` binary can patch.

## 4. APFS analysis

The script does **not** hardcode APFS volume UUIDs or roles anywhere - the
backup step addresses volumes purely by ramdisk-assigned mountpoint
(`/mnt1`, `/mnt2`, `/mnt6`) and well-known legacy paths
(`/dev/disk0s1s1`/`/dev/disk0s1s2`, used only for the pre-10.3 HFS+ case).
For APFS-era devices/versions it just shells out to `mount_filesystems`,
which is Apple's own volume-role resolver - Semaphorin has no independent
APFS parsing of its own to get wrong. The bug here, as above, was that a
failed `mount_filesystems` was never detected.

What `--diagnose` now adds, without mounting anything itself:

- Lists the *current* mount table (`/sbin/mount`) so you can see whether
  anything is mounted at all before any mount attempt.
- Lists raw `/dev/disk*` nodes so you can see whether the kernel even
  enumerated the NAND/APFS container at the block level (if `disk0` doesn't
  show up here, no amount of `mount_filesystems` retrying will help - that
  points at a kernelcache/driver/iBoot patch problem, not a ramdisk/SEP
  one).
- Does a read-only `find -xdev` for `sep-firmware`/`apticket` on whatever is
  *already* mounted (normally nothing extra is mounted this early, which is
  expected and reported as such rather than as an error).

The new `--diagnose-mount-test` stage (opt-in, typed confirmation) is where
an actual `mount_filesystems` call happens for real, with the following
instrumentation that was not there in the original backup step: mount table
snapshot before, full non-redirected output during, mount table snapshot
after (checked against `/mnt1`, `/mnt2`, `/mnt6` individually), then a clean
unmount of whatever did mount.

## 5. Impact of the prior surrealra1n tethered downgrade (16.7.16 -> 14.3)

This cannot be fully characterized from the host side - it depends on SEP
and Boot Manifest state that isn't readable without exactly the SEP
diagnostics being investigated here. What the audit can say:

- A **tethered** boot (via checkra1n/palera1n-family tools, which is what
  surrealra1n is) does not go through Apple's signed, untethered restore
  path. The main AP OS can be made to boot an older kernelcache/OS this
  way, but the **SEP itself is a separate, independently-booted coprocessor
  with its own firmware and its own anti-replay/epoch state**, and tethered
  tools do not necessarily bring the SEP all the way down to a "clean
  14.3 SEP" state - they are primarily aimed at getting the AP kernel to
  boot.
- This means: by the time Semaphorin's ramdisk talks to `seputil`, the SEP
  may still be running whatever firmware/epoch state is left over from
  16.7.16 (or an intermediate state), even though the AP side reports
  "14.3". A `sep-firmware.img4` built for a stock, untethered 14.3 install
  can reasonably be rejected by an SEP that isn't actually in a stock 14.3
  state. This is the single most likely explanation for `0xe00002d5` in
  this exact chain, but it is a hypothesis to verify with the new
  diagnostics on real hardware, not a confirmed root cause.
- Practically, this means a successful `11.3` tethered downgrade from this
  starting point may need a `seputil`/SEP-firmware pairing that matches the
  device's **actual current SEP state** (whatever surrealra1n left it in),
  not necessarily the stock pairing for "device currently reports 14.3"
  that Semaphorin's `$r` prompt assumes.

## 6. What `--diagnose` does and does not do

Safe by default - **never** calls `mount_filesystems`, `mount_apfs`, or
`seputil`, never touches NVRAM, never writes to any APFS volume:

```
sudo ./semaphorin.sh 11.3 --diagnose
```

- Detects device model/CPID/mode (reuses the existing detection code), boots
  the SSH ramdisk via the existing pwn/boot infrastructure (same path as
  `--ramdisk`), and waits for you to confirm it booted (same prompt as
  every other mode).
- Confirms SSH reachability with a timeout (`ConnectTimeout=8`,
  `ServerAliveInterval=3`/`ServerAliveCountMax=2`), so a dead link is
  detected in seconds instead of hanging.
- Lists the current mount table, raw disk device nodes, and the
  availability/location of `mount_apfs`, `mount_filesystems`, `seputil`,
  `fsck_apfs`, `newfs_apfs` on the ramdisk (existence/path/size only - none
  of them are executed).
- Does a read-only search for `sep-firmware*`/`apticket*` on whatever is
  already mounted.
- Sanity-checks the locally downloaded ramdisk boot components
  (`iBSS.img4`, `iBEC.img4`, `kernelcache.img4`, `devicetree.img4`,
  `trustcache.img4`, `ramdisk.img4`) for existence and zero-byte corruption.
- Reports what's already backed up locally under `<deviceid>/0.0/`
  (existence + timestamp only - never file contents).
- Logs every step with a timestamped `[INFO]/[OK]/[WARN]/[ERROR]/[STOP]`
  line to both the console and `logs/diagnose_<deviceid>_<timestamp>.log`.
- If the SSH link dies at any point, the exact step is reported as `[STOP]`
  and diagnostics end immediately (no attempt to "push through").

Explicit, separately-confirmed extra stage:

```
sudo ./semaphorin.sh 11.3 --diagnose --diagnose-mount-test
```

Prompts `Type YES (all caps) to proceed...`, then runs the real
`mount_filesystems` once with full captured output, checks the resulting
mount table, and unmounts again. This is the only place in `--diagnose`
that performs a real mount, and it still never touches `seputil` or SEP
firmware.

Never does: restore, wipe, nand/activation backup-and-restore, NVRAM writes,
or anything from the "Main operation mode" list other than booting the
ramdisk.

## 7. Hardening applied to `--restore` itself

Separately from `--diagnose`, the actual backup step inside `--restore`
(the one that produced the original report) was hardened in place, minimal
diff, same file layout fetched:

- `mount_filesystems`'s output is now captured to
  `logs/restore_backup_<deviceid>_<timestamp>.log` instead of being
  discarded.
- `/mnt1` and `/mnt2` are checked against the live mount table
  (`_remote_is_mounted`) **before** any `scp` is attempted. If `/mnt1`
  didn't mount, the step now aborts immediately with a clear `[ERROR]`
  pointing at the log, instead of silently failing every subsequent `scp`
  and only noticing at the very end.
- `apticket.der` and `sep-firmware.img4` existence on the remote mount is
  checked (`_remote_path_exists`) before copying, so a missing file is
  reported by name instead of `scp` just silently failing.
- The final "could not back up required files" message now lists exactly
  which file(s) are still missing and points at the saved log, and the
  script exits with status `1` (was `0`) so calling scripts/CI can tell
  success from failure.

## 8. Testing performed

- `bash -n semaphorin.sh` - passes (no syntax errors).
- Mock-SSH unit tests of the new helper functions
  (`_diag_ssh`/`_remote_path_exists`/`_remote_is_mounted`), run against a
  fake `sshpass` binary simulating: a successful remote command, a remote
  command returning a non-zero exit, and a simulated dropped SSH connection
  (`exit 255`). All 13 assertions pass, confirming:
  - a normal success is logged `[OK]` and does not set
    `ssh_connection_lost`;
  - a remote command error is logged `[WARN]` with its exit code, and also
    does **not** set `ssh_connection_lost` (correctly distinguished from a
    dead link);
  - a dropped connection (exit 255) is logged `[STOP]`, correctly named by
    description, and does set `ssh_connection_lost=1`;
  - `_remote_path_exists`/`_remote_is_mounted` correctly report
    exists/missing/mounted/not-mounted and propagate the 255 case.
- **Not tested against real iPhone X hardware.** No physical device was
  available in this environment. Do not take this change as a verified fix
  for the `0xe00002d5`/APFS-mount issue - it is a diagnostics and
  error-handling improvement that should make the *next* real attempt on
  hardware tell you exactly where and how it fails, which the original
  script could not do.

### Suggested test plan for real iPhone X hardware

1. `sudo ./semaphorin.sh 11.3 --diagnose` from the current post-surrealra1n
   14.3 tethered state. Confirm it reaches "SSH ramdisk is reachable" and
   completes without needing `--diagnose-mount-test`. Save the log.
2. Re-run with `--diagnose --diagnose-mount-test`, confirm with `YES`, and
   capture the full `mount_filesystems` output this produces - this is the
   first time the *real* stderr from `mount_apfs`/`mount_filesystems` will
   be visible in a saved file rather than lost to `/dev/null` or scrolled
   off a console.
3. Compare `/dev/disk*` enumeration and the mount table before vs. after
   the test to see whether the container is even recognized at the block
   level.
4. If `/mnt1` does mount successfully under `--diagnose-mount-test`, re-run
   `--restore` for real - the hardened backup step should now either
   succeed cleanly or fail with a specific, logged reason instead of the
   previous generic message.
5. If `/mnt1` does *not* mount, the next investigation step is outside
   `semaphorin.sh`: try the same ramdisk/mount_filesystems combination
   immediately after a **fresh** checkra1n/palera1n tethered boot of 14.3
   (not surrealra1n) to see if the mount behavior differs, which would
   point squarely at surrealra1n's tethered-boot APFS/SEP state rather than
   at Semaphorin's ramdisk selection.

## 9. Remaining obstacles / known gaps

- `seputil`'s behavior with no arguments is unverified; `--diagnose`
  deliberately does not call it at all, even passively.
- There is no source for `mount_apfs`/`mount_filesystems`/`seputil` in this
  repo or in Semaphorin upstream - they are Apple binaries inside the
  RestoreRamDisk, so the actual `0xe00002d5` behavior cannot be patched or
  even disassembled from here; it can only be worked around by matching
  ramdisk/SEP-firmware pairing or avoided entirely.
- No physical-hardware verification was possible in this environment.
- `--diagnose`'s component-size sanity check is a best-effort heuristic
  (zero-byte/missing check), not a full IMG4 manifest/signature
  cross-check against the current Boot Manifest epoch.

## 10. Roadmap toward iOS 11.3

1. Run the test plan above on real hardware and capture the full
   `mount_filesystems`/`mount_apfs` output via `--diagnose-mount-test`.
2. If the mount succeeds and `seputil`'s `0xe00002d5` reproduces with full
   output now visible, that output (plus the SEP firmware epoch/version it
   names, if any) is the next concrete lead - cross-reference it against
   what surrealra1n's own SEP-handling code expects for A11/14.3.
3. If the mount itself fails, treat it as a ramdisk/driver issue separate
   from SEP: check whether the same ramdisk version mounts cleanly on a
   **freshly, cleanly** tethered-booted 14.3 (via checkra1n/palera1n
   directly) vs. after surrealra1n specifically, to isolate whether
   surrealra1n's tethered-boot state is the variable.
4. Only after APFS mounts and the real SEP firmware load result is known
   should any SEP-firmware-pairing change be attempted, and that change
   belongs in which ramdisk/SEP-firmware version Semaphorin selects for
   this specific post-tethered-downgrade state - not in forcing `seputil`
   to proceed past an error.

## 11. Findings from real hardware (iPhone10,6 / D221AP, 14.3 SSH ramdisk)

Source: `logs/diagnose_iPhone10,6_*.log` from a `--diagnose` run on a physical
device (read-only; `mount_filesystems` was not run). Facts first, then
hypotheses, clearly separated.

### Facts (observed)
- The SSH ramdisk is stable; no link loss during the passive checks. Ramdisk
  kernel is `xnu-7195.62.1` (iOS 14.3), `RELEASE_ARM64_T8015`.
- Only `/` (hfs, read-only) and `/dev` are mounted before any mount attempt.
- Block devices: `disk0`, `disk0s1`, `disk0s1s1`..`disk0s1s7` (7 APFS
  volumes), plus `disk1`, `disk2`. There are no `disk1sN` nodes, so the
  volumes are addressed as `disk0s1sN`.
- `/usr/bin/mount_filesystems` is a 3820-byte **bash script**, not a binary.
  It calls `/sbin/mount_apfs` (symlink into
  `/System/Library/Filesystems/apfs.fs/`), `apfs.util` and
  `/usr/libexec/seputil` (171760 bytes) by absolute path. An earlier
  PATH-based lookup that reported them "not found" was wrong.

### What `mount_filesystems` does on a non-16/17 ramdisk
1. `mount_apfs /dev/disk0s1s1 /mnt1` - **System is assumed to be slot 1**.
2. For slots 3-7, `apfs.util -p` is used to find `Preboot` (mounted at
   `/mnt6`) and `xART` (mounted at `/mnt7`, followed by
   `seputil --gigalocker-init`).
3. `seputil --load` the SEP firmware from
   `/mnt6/$(cat /mnt6/active)/usr/standalone/firmware/sep-firmware.img4` if
   `/mnt6/active` exists, otherwise from
   `/mnt1/usr/standalone/firmware/sep-firmware.img4`.
4. `mount_apfs /dev/disk0s1s2 /mnt2` - **Data is assumed to be slot 2**.

There is no `set -e`: every step runs even if an earlier one failed. This
answers where `seputil` gets `sep-firmware.img4`: from the **installed OS's**
Preboot/System volume, i.e. whatever SEP firmware the device currently has
on disk, not from the ramdisk.

### Hypotheses (not yet verified)
- Slots 1 (System) and 2 (Data) are hard-coded, while only Preboot/xART are
  detected by role. After a tethered downgrade the volume order may differ.
  `--diagnose-apfs-ro` prints the role of all seven slots to check this.
- `mount_apfs: ... No such file or directory` is a single failing mount;
  which one is unknown. A missing mountpoint directory is one candidate
  (the passive diagnose now lists `/mnt1 /mnt2 /mnt6 /mnt7`). If System fails
  and `/mnt6/active` is absent, `seputil --load` is pointed at a path on an
  unmounted volume, which would explain "failed to load .../sep-firmware.img4".
- From memory of `IOReturn.h` (please verify), `0xe00002d5` is
  `kIOReturnBusy`, which would suggest the SEP was already loaded/started in
  that boot rather than a bad image. Cheap test: boot a fresh ramdisk and run
  the load only once.
- The SEP firmware found on disk may belong to a different OS generation than
  the 14.3 ramdisk kernel after the tethered downgrade. The new stage prints
  `SystemVersion.plist` and the `sep-firmware` file metadata to compare.

### New opt-in stage: `--diagnose-apfs-ro`
```
sudo ./semaphorin.sh 11.3 --diagnose --diagnose-apfs-ro
```
After a typed `YES`: runs `apfs.util -p` on slots 1-7, mounts System and
Preboot **read-only**, prints `SystemVersion.plist`, the `/mnt6/active` slot
and the `sep-firmware` file metadata, then unmounts. It never mounts xART or
Data and never runs `gigalocker-init` or `seputil`, so a failure here is an
APFS failure, not a SEP failure. Not yet run on hardware.

### First `--diagnose-apfs-ro` run (what it did and did not show)
- Passive checks: ramdisk is `ProductVersion 14.3 (18C66)`; `/mnt1 /mnt2 /mnt6
  /mnt7` exist, so a missing mountpoint is **ruled out** as the cause of
  `mount_apfs ... No such file or directory`.
- `apfs.util -p /dev/disk0s1s1` printed `System` (slot 1 is System, as
  `mount_filesystems` assumes) and the run was then reported as an SSH loss,
  within the same second, before any mount or SEP call.
- That report was **ambiguous**: `ssh` exits 255 both for a connection error
  and when the remote command itself exits 255, and no ssh error text was
  captured. `_diag_ssh` now confirms a 255 with a separate liveness probe and,
  on a real drop, logs the USB mode 1/3/6 s later (`ramdisk` = kernel still
  up, `recovery`/`dfu` = device reset, `none` = USB gone). This is the
  first mechanism in this repo that can tell a panic/reset from a SEP error.

## 12. Second real-hardware run (iPhone10,6, 14.3 ramdisk, `--diagnose-apfs-ro`)

### Facts (observed)
- The "SSH connection lost" in the previous run was a **false positive of the
  tool**: `apfs.util -p` always exits 255 (it prints the role and returns -1).
  With the liveness probe the link was alive for all seven calls.
- APFS volume roles: `s1=System s2=Data s3=xART s4=Baseband Data s5=Hardware
  s6=Preboot s7=Update`. The slots `mount_filesystems` hard-codes (System=s1,
  Data=s2) are correct on this device, and Preboot/xART are found by role, so
  **a changed volume order is ruled out**.
- `mount_apfs -o rdonly` of System (`/dev/disk0s1s1`, "sealed") and Preboot
  (`/dev/disk0s1s6`) both **succeeded**, with no ENOENT and no SSH loss.
- System volume `SystemVersion.plist`: 14.3 (18C66), matching the ramdisk.
- `/mnt6/active` exists, so `seputil --load` uses
  `/mnt6/<active>/usr/standalone/firmware/sep-firmware.img4`: a 1,415,361-byte
  file with an mtime of the same day as the surrealra1n restore. The System
  volume itself has no `sep-firmware` under `/usr/standalone/firmware`.
- Missing mountpoints are ruled out (all four exist).

### What this changes
- The earlier "volume order / hard-coded slot" hypothesis is not supported.
- `mount_apfs` itself works read-only, so the original
  `mount_apfs: ... No such file or directory` comes from something this run
  did **not** do: a mount without `-o rdonly` of the sealed System volume,
  the xART mount, or the Data volume mount (which `mount_filesystems` runs
  last, after `seputil --load`). `--diagnose-apfs-ro` now also mounts xART
  read-only to test that candidate. Still untested: `seputil --gigalocker-init`
  and `seputil --load`, the two steps that involve the SEP.

### Log size
Default `--diagnose` logs are now compact (about 25 lines instead of 300):
roles, tool presence and listings are single summary lines. The full dumps
(`mount_filesystems` source, directory scans, apfs.fs listing) moved behind
`--diagnose-verbose`. The SEP firmware copied for inspection and the
`img4tool` output are stored next to the log and are not written into it
(the latter contains the device ECID).
