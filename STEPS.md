# Exact procedure, as executed

Result: Magisk root, persistent across reboot, SELinux still Enforcing, kernel untouched by any
exploit because the bootloader was unlocked and a patched boot image was flashed.

Nothing here is reversible-only; the one destructive step is the unlock (userdata wipe).

## 1. Confirm unlock support

```
adb shell getprop ro.oem_unlock_supported   # 1
adb shell getprop sys.oem_unlock_allowed    # 1
```

## 2. Unlock the bootloader

```
adb reboot bootloader
fastboot getvar all        # unlocked: no, secure: yes, product: tb8168p1_64_bsp, slot-count: 0
fastboot flashing unlock   # OKAY after ~166 s, no on-screen prompt
fastboot getvar all        # unlocked: yes, secure: no
```

Gotchas: `getvar unlocked` and `oem device-info` are both `unknown command` on this bootloader, so
read the `unlocked:`/`secure:` lines out of `getvar all`. `slot-count: 0` means no A/B.

## 3. Get a boot image (the hard part)

`/dev/block/mmcblk0p15` (boot) and `mmcblk0p28` (vbmeta) are `brw------- root`, so shell cannot
`dd` them, `adb root` is refused on a `user` build, and this bootloader has **no `fastboot
fetch`**. `fastboot getvar partition-size:boot` is also unsupported, though
`partition-size:vbmeta` answers `b40000`.

mtkclient BROM was attempted and abandoned: for MT8168 it needs a DA and a preloader, no MT8168
preloader ships with mtkclient (838 others do), and the DA stage-2 upload dies with
`USBError(5) ... Stage wasn't executed` even through a direct port. See `MTKCLIENT.md`.

What worked: a community partition dump for the same model,
`https://drive.google.com/file/d/1ER48dr0nSFkiTjcFS7b8qm6tvM51rXJS` (`tabdump.7z`, 826 MB). It
contains `boot.bin` (32 MB, Magisk-patched, verified by `magiskinit` and `.magisk` entries in the
```
ramdisk) and a stock `vbmeta.bin` (11,796,480 bytes, matching this device's vbmeta size).
```

## 4. Flash

```
fastboot --disable-verity --disable-verification flash vbmeta vbmeta.bin
fastboot flash boot boot.bin
fastboot reboot
```

Expected state after: `ro.boot.verifiedbootstate=orange`, `ro.boot.flash.locked=0`.

## 5. Re-patch with the current Magisk, because the dump's patch is stale

The app reported "Requires additional setup ... please reinstall Magisk within app", so:

1. Install `Magisk-v31.1.apk`.
2. Push the boot image to `/sdcard/Download/boot.img`.
3. App, Install, "Patch firmware / image", "Select from storage", pick the file.
4. It writes `/sdcard/Download/magisk_patched-31100_*.img`.
5. `adb pull` that, then `fastboot flash boot` it and reboot.

## 6. Grant adb shell root (the non-obvious step)

Magisk 31.1 has **no** "Superuser access: Apps and ADB" setting in the UI any more, so `su` from
`adb shell` returns `Permission denied` instantly, without any prompt, and the Superuser tab still
says no app has asked. The `settings` table holds only `bootloop`, and `policies` is empty. The
allowed values are `QUERY=0, DENY=1, ALLOW=2, RESTRICT=3` (`SuPolicy.kt`).

Fix by inserting a policy row as root. The app cannot run shell commands, so a Magisk module does
it, since `customize.sh` runs as root at install time:

```
# module.prop: id=shellgrant, plus the usual name/version/versionCode/author/description
magisk --sqlite "REPLACE INTO policies (uid,policy,logging,notification,until) VALUES (2000,2,1,1,0)"
```

Install it through Modules, then `adb shell su -c id` returns `uid=0(root) ... u:r:magisk:s0`.
The `until=0` value is required, the app's own DAO writes 0 for "permanent".

## 7. Verify

```
adb shell su -c 'id; getenforce'      # uid=0(root) u:r:magisk:s0 ; Enforcing
```

Confirmed again after a full reboot, so it persists.

## Caveats

- The flashed kernel is from the dump: `4.14.186 #1 SMP PREEMPT Mon Jun 7 19:30:41 CST 2021`,
  while this unit's stock kernel was the 2022-09-03 build. `ro.build.display.id` is unchanged
  (`100011886_20220903`) because only `boot` was touched. Anything that depends on the exact stock
  kernel (some vendor kernel modules) could misbehave; so far everything boots and works.
- The dump is for the `100011886CA` SKU. Same board, different SKU label.
- vbmeta is flashed with verity and verification disabled, so AVB no longer gates the boot image.

## Revert

```
fastboot --disable-verity --disable-verification flash vbmeta <stock vbmeta.bin>
fastboot flash boot <stock boot.bin or the dump's unpatched original>
fastboot reboot
```

Uninstalling Magisk from inside the app restores its backup of the boot image. Re-locking
(`fastboot flashing unlock` counterpart `fastboot flashing lock`) needs a stock, unpatched boot.

## Image hashes seen here

| file | sha256 (first 20) |
|---|---|
| dump `boot.bin` (Magisk, from archive) | `3096fa0866c09ee1d78a` |
| `magisk_patched-31100` from the app | `c6e36042130c957f4940` |
