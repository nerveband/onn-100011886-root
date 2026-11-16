# mtkclient / BROM notes for MT8168 (failed route)

Kept because it is the only way to image the flash directly, and the next person will try it.

## State reached

- BROM entry does work: `ioreg` shows `MT65xx Preloader`, and mtkclient identifies
  `CPU: MT8168/MT6357`, `HW code: 0x8168`, config `0xe0`, then tries a handshake.
- `fastboot` on this bootloader has no `fetch`, and `boot` is root-only, so BROM is the only
  on-device dump path.

## Failure modes seen

1. `Upload data was accepted. Jumping to stage 2...` then `USBError(5, 'Input/output Error')`,
   `Stage wasn't executed. Maybe dram issue ?.`, `Failed to upload da`.
2. Repeated `Handshake failed, retrying` when the device was not cleanly off (a power-on with keys
   held lands in *factory mode* instead, which has no adbd on this build).
3. `No emmc info, can't parse existing preloaders` and `No preloader given. Operation may fail due
   to missing dram setup.` There is **no MT8168 preloader** in mtkclient's shipped set (838 entries,
   zero match `8168`).
4. The device went flat during the attempts and stopped enumerating until charged; BROM will not
   come up on a dead cell.

## Environment fixes required to even run it

- Python 3.9 is too new-era-broken for mtkclient (`HeapParams | None`); use 3.12.
- `mfusepy` raises `OSError` not `ImportError` when libfuse is missing, and macOS `libfuse` fails
  with "Linux is required". Patch both guards to `except (ImportError, OSError)`.

## Correct BROM key sequence

Device fully off (screen black, cable out), hold **vol up + vol down** (or all three with power),
insert the cable while holding, hold ~3 s, release. Insert the cable *after* holding the keys.

## Untried remedies

- Supply the model's own `preloader_boot.bin` to `--preloader`; the XDA thread also carries a
  `preloader.bin` attachment (4 MB) and a modified mtkclient build the poster says allows larger
  buffers.
- A different host than this Apple Silicon Mac, where the DA stage-2 USB error may not occur.
