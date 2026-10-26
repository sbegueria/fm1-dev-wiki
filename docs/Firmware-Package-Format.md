---
title: "Firmware Package Format (.fwsc)"
parent: Boot and install
nav_order: 2
---

# Firmware Package Format (.fwsc)

The `.fwsc` file that M-VAVE's updater and the community installers send. This page describes it as Felucca's
open package builder writes it (`tools/fm1pkg_make.py`, GPL-3.0-only; the same in felucca-bsp's `tools/fm1pkg.py`,
MPL-2.0). M-VAVE's own packages have the same structure (FM-1-RE).

**All integrity checks in the format are CRC16 (polynomial 0x1021, initial value 0).** **[CODE]**

## Layers, outside in

```
.fwsc
 └─ the identity: one marker byte after each of the first 20 blocks of 0x2F bytes
     └─ UFW container ("AC791N"): a 0x40-byte header, a file list, the files
         ├─ flash.bin   (type 0)   the flash image, 0x93000 bytes
         │    ├─ head [0, 0x4000): flash header, JLFS directory, uboot.boot at 0xA0, isd_config (chip key)
         │    └─ app area 0x4000..: app_area_head, app.bin, cfg_tool.bin, region descriptors, cfg/eq_cfg_hw.bin
         │                          all SFC-encrypted with the chip key
         └─ ota.bin     (type 100) the update loader
```

### The identity

The package's model and version (for example `FM-1_900`) is hidden in the file: after each of the first 20
blocks of `0x2F` bytes comes one marker byte, `(char + i + 1) & 0xFF` for character `i`, or `0x7D` for none.
Strip the 20 marker bytes and you have the "logical image" that the device reads during an update. **[CODE]**

```python
# from Felucca's tools/fm1_install.py (GPL-3.0-only)
def product_of(raw):
    return "".join(chr((m - i - 1) & 0xFF) for i in range(20) if (m := raw[i * 0x30 + 0x2F]) != 0x7D)
```

Installers compare the package's model (the part before `_`) with what the running device reports. Identities
seen: `FM-1_…` (stock and many open firmwares), `ota-FM-1_900` (Felucca's loader answering), and for M-VAVE's FM1
Open, `FM1_OFFICIAL_OPEN` and `FM1_COMMUNITY_nn` (per fm1-base's installer). **[CODE]**

### The UFW container

A 0x40-byte header (CRC16 of the header and of the file list, the total size, the file count, the chip name
`AC791N`), then one 0x50-byte entry per file (type, index, CRC16, offset, size, padded size, name at +0x40),
each encrypted with JieLi's ENC stream cipher (key `0xFFFF`). Files start on 0x100-byte boundaries. **[CODE]**

### The flash image

- **Head** (`[0, 0x4000)`): a 32-byte flash header with JieLi's SDK default values; JLFS directory entries for
  `uboot.boot`, `isd_config.ini`, `app_dir_head` and `key_mac`; the SDK's bootloader at `0xA0`; an `isd_config`
  blob that decodes to the chip key. Each 32-byte header and entry is ENC-encrypted with `0xFFFF`. **The update
  loader never writes the head.**
- **App area** (`0x4000`…): an `app_area_head` entry, then entries for `app.bin` (padded to the full slot,
  `0x8DFBC` bytes), `cfg_tool.bin`, region descriptors (VM, protection range, BTIF, USR), and a nested `cfg`
  directory holding the SDK's default `eq_cfg_hw.bin`. The whole area is encrypted with the **SFC cipher**: ENC
  per 32-byte block, the key mixed with the block address (`key ^ (offset >> 2)`), with the chip key.

### Ciphers

```python
# from Felucca's tools/fm1pkg_make.py (GPL-3.0-only)
def enc(buf, off, size, key=0xFFFF):          # JieLi ENC stream cipher, in place
    for i in range(size):
        buf[off + i] ^= key & 0xFF
        key = ((key << 1) ^ (0x1021 if key & 0x8000 else 0)) & 0xFFFF

def sfc(buf, off, size, base, key):           # SFC cipher: ENC per 32-byte block
    for i in range(0, size, 32):
        enc(buf, off + i, min(size - i, 32), key ^ ((off + i - base) >> 2))
```

The chip key on known FM-1s is `0x980F`. **[HW]**

## The SDK files inside

Three files come from JieLi's AC79 SDK (Apache-2.0): `uboot.boot` (the bootloader), `cfg_tool.bin` and
`cfg/eq_cfg_hw.bin`. Open projects don't commit them; the build reads them from an SDK checkout and some check
them by SHA-256. Because the SDK's `uboot.boot` is in the head, **installing a full package with a stock head puts
the stock bootloader back** where the installer writes the head. The Felucca loader writes only the app area.

## Tools that read or write packages

- Felucca / felucca-bsp: `fm1pkg_make.py` / `fm1pkg.py` (build), `fm1_install.py` (install).
- [jl-misctools](https://github.com/kagaimiq/jl-misctools): JieLi container utilities.
- [FM-1-RE](https://github.com/AL-255/FM-1-RE): unpacked stock packages and their metadata.
- [M-Flasher](https://github.com/jvitkauskas/m-flasher): installs M-VAVE's `.fwsc` through download mode and
  shows a flash map of what will change.

## Sources

- Felucca's `tools/fm1pkg_make.py`, `tools/fm1_install.py` (GPL-3.0-only, © Leo Kuroshita).
- [m-bootloader SPEC.md](https://github.com/jvitkauskas/m-bootloader/blob/main/docs/SPEC.md) §5.
- [FM-1-RE architecture](https://github.com/AL-255/FM-1-RE/blob/main/docs/architecture.md) §2.

---

Unofficial community wiki. Not affiliated with M-VAVE or JieLi. Text under [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/); code excerpts keep their own licences ([License](License.md)). Facts are tagged **[HW] [SDK] [RE] [CODE] [?]**: see [Home](Home.md). Found an error? See [Contributing](Contributing.md). [Home](Home.md)
