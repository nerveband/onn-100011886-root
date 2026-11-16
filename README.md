# onn. 100011886: unlock and root notes

Working log for rooting a Walmart **onn.** tablet, model **100011886**. Transient and permanent
root notes both live here; the goal is a usable rooted device, not a purist one.

## Device

| | |
|---|---|
| Model / product | 100011886 (`tb8168p1_64_bsp`) |
| Brand | onn (Walmart house brand) |
| SoC | MediaTek MT8168 |
| Android | 11 (SDK 30) |
| Build | `100011886_20220903`, `onn/100011886/100011886:11/RP1A.200720.011/1699824658:user/release-keys` |
| Security patch | 2022-08-01 |
| Kernel | 4.14.186 (built 2022-09-03) |
| Build type | `user`, `ro.debuggable=0`, `ro.secure=1`, `ro.adb.secure=1` |
| Bootloader | MTK fastboot, initially `unlocked: no`, `secure: yes` |
| OEM unlock props | `ro.oem_unlock_supported=1`, `sys.oem_unlock_allowed=1` |
| Serial used here | 11862107A012635 |

## Status

- [x] Identified, surface enumerated
- [x] Bootloader unlocked
- [ ] Boot image obtained
- [ ] Magisk patched and flashed
- [ ] Verified root survives reboot
- [ ] Debloated

## Bootloader unlock: done

Unlike the Samsung `U` models, this tablet advertises OEM unlock support and allows it, so this
needs no exploit.

```
adb reboot bootloader
fastboot getvar all            # unlocked: no, secure: yes, product: tb8168p1_64_bsp, slot-count: 0
fastboot flashing unlock       # OKAY, "Start unlock flow", ~166 s
fastboot getvar all            # unlocked: yes, secure: no
```

Notes:

- Unlocking **wipes userdata** and resets the device to first-boot setup.
- `fastboot getvar unlocked` and `fastboot oem device-info` are both rejected on this bootloader
  (`unknown command`); use `fastboot getvar all` and read the `unlocked:`/`secure:` lines.
- No on-screen confirmation prompt appeared; the command just blocked until it finished. Do not
  interrupt it.
- `slot-count: 0`, so there is no A/B slot to worry about, and `boot` is a single partition.

## Root route

`fastboot boot` (temporary boot of a patched image) is unreliable on MT8168, and an Android 11
MTK device will loop on dm-verity/AVB if `boot` is flashed without also handling `vbmeta`. So the
route is: obtain the stock `boot.img` for this exact build, patch it with Magisk, then flash it
together with a disabled verity/verification state.

Places to get the stock image, in order of preference:

1. `dd` from the device once anything grants a root shell.
2. A firmware dump for `100011886` (community archives; the SourceForge `onn-tablet-firmwares`
   project carries other models, not this one).
3. `mtkclient` in BROM mode to read the `boot` partition straight off eMMC.

Flash step, once a patched image exists:

```
fastboot --disable-verity --disable-verification flash vbmeta vbmeta.img
fastboot flash boot magisk_patched.img
fastboot reboot
```

## Attack surface if the fastboot route stalls

Enumerated while locked, worth keeping because it is identical to the A32 find:

- Open to `shell`: `/dev/binder`, `/dev/hwbinder`, `/dev/mali0`, `/dev/ion`, `/dev/uhid`,
  `/dev/uinput`, and a named `/dev/ashmem<uuid>` node.
- Denied despite mode 0666: `/dev/ashmem` and `/dev/tzdev`.
- Kernel 4.14.186 with a 2022-08 patch level, and DDK-backed `mali0` plus a fully usable
  `/dev/ion` (seven heaps, five mappable). `/dev/videogo` and `/dev/mtk_cmdq` are absent, so
  `mtk-su` / CVE-2020-0069 is structurally out.
- `perf_event_open` allowed. No user namespaces (`unshare` EINVAL), no `io_uring`, no
  unprivileged BPF, no `userfaultfd`.

## Safety

Everything here is reversible except the data wipe that unlocking already caused. MTK devices can
generally be recovered from a bad boot image with `fastboot flash boot` of the stock one, or via
`mtkclient` in BROM mode. Keep the stock `boot.img` and `vbmeta.img` before flashing anything.
