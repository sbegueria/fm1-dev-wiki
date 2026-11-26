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

**A name to ignore.** FM-1-RE calls this CRC `jl_crc16` and names it "CRC-16/CCITT-FALSE", whose initial value is
`0xFFFF` ([`io/11-ota-protocol.md`](https://github.com/AL-255/FM-1-RE/blob/main/docs/io/11-ota-protocol.md), "Known open
items"). M-VAVE's own files say initial value 0, which is CRC-16/XMODEM: in the stock V13 package as FM-1-RE unpacked it,
the stored CRCs of the `uboot.boot`, `isd_config.ini` and `app_dir_head` entries, and both CRCs in the header of the
stock `ota.bin` (`analysis/device/ota-loader/ota_stock.bin`), match with 0 and not with `0xFFFF` (this wiki's check,
2026-11-26, with Python's `binascii.crc_hqx`). felucca-bsp's `tools/fm1pkg.py` and `fm1_install.py` and m-bootloader's
`tools/bankcb.py` all start from 0. So FM-1-RE's name is off, not the algorithm. **[RE] [CODE]**

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
Not every package uses 20: M-VAVE's stock SMK-37 Pro V16 package uses **36** marker slots; the Elite's and SMK37 Open's
use 20 (SMK37 Open `sdk/WebUpdate.js`, b2d6bcb). **[CODE]**

```python
# from Felucca's tools/fm1_install.py (GPL-3.0-only)
def product_of(raw):
    return "".join(chr((m - i - 1) & 0xFF) for i in range(20) if (m := raw[i * 0x30 + 0x2F]) != 0x7D)
```

Installers compare the package's model (the part before `_`) with what the running device reports. Identities
seen: `FM-1_…` (stock and many open firmwares), `ota-FM-1_900` (Felucca's loader answering). M-VAVE's FM1 Open
releases are `FM-1_8000012` to `FM-1_8000014` (V12–V14, `app/version.h`); its packager takes only `FM-1_8` and six
digits, and its web updater also accepts the names `FM1_OFFICIAL_OPEN` and `FM1_COMMUNITY_nn`
(`sdk/WebUpdate.js` line 58, 4c7496d). SMK37 Open uses `SMK-37 Pro_8000001` and `SMK-37 Elite_8000001`. **[CODE]**
The full list, and M-UPGRADE 3.11's limit of three digits, are in
[Update Protocol and Installers](Update-Protocol-and-Installers.md#package-identities).

### The UFW container

A 0x40-byte header (CRC16 of the header and of the file list, the total size, the file count, the chip name
`AC791N`), then one 0x50-byte entry per file (type, index, CRC16, offset, size, padded size, name at +0x40),
each encrypted with JieLi's ENC stream cipher (key `0xFFFF`). Files start on 0x100-byte boundaries. **[CODE]**

### The flash image

- **Head** (`[0, 0x4000)`): a 32-byte flash header with JieLi's SDK default values; JLFS directory entries for
  `uboot.boot`, `isd_config.ini`, `app_dir_head` and `key_mac`; the SDK's bootloader at `0xA0`; an `isd_config`
  blob that decodes to the chip key. Each 32-byte header and entry is ENC-encrypted with `0xFFFF`. **Felucca's
  update loader never writes the head; M-VAVE's does** (an official V15 → V16 update replaced m-bootloader with the package's stock bootloader:
  [Boot Chain and Bootloaders](Boot-Chain-and-Bootloaders.md)).
- **App area** (`0x4000`…): an `app_area_head` entry, then entries for `app.bin` (padded to the full slot,
  `0x8DFBC` bytes), `cfg_tool.bin`, region descriptors (VM, protection range, BTIF, USR), and a nested `cfg`
  directory holding the SDK's default `eq_cfg_hw.bin`. The whole area is encrypted with the **SFC cipher**: ENC
  per 32-byte block, the key mixed with the block address (`key ^ (offset >> 2)`), with the chip key.

**M-VAVE's V15 and V16 differ in size.** **[CODE]** From the values Felucca's installer pins for the unmodified files
(`web/fm1pkg.js`, [5044cce](https://github.com/hugelton/Felucca/commit/5044cce), 2026-11-15):

| | V15 (`FM-1_015`) | V16 (`FM-1_016`) |
| --- | --- | --- |
| `.fwsc` size, SHA-256 | 699,956 B, `db1642b2…` | 718,196 B, `d017fb65…` |
| `flash.bin` size | 602,112 B (`0x93000`) | 692,224 B (to `0xA9000`) |
| `app.bin` in `flash.bin` | at `0x4120`, `0x8DFBC` B | at `0x4140`, 597,928 B |
| `ota.bin` | 19,969 B | 19,969 B |

V16's package also carries a 5-byte `farg.cfg` whose purpose is unknown (m-bootloader `docs/STOCK-V15-V16.md`).
**[RE]**

### The update loader (`ota.bin`)

**[RE]** M-VAVE's `ota.bin` (19,969 bytes; the same in V13 and V14), as FM-1-RE checked it byte by byte
([`io/11-ota-protocol.md`](https://github.com/AL-255/FM-1-RE/blob/main/docs/io/11-ota-protocol.md), "Known open items",
"ota.bin container format"):
- an outer 32-byte header: header CRC, data CRC (over the rest of the file), data offset `0x20`, data length `0x4DE1`,
  the name `usb_hid_ota.bin`;
- an inner boot header: its own two CRCs, the image size `0x5B1C`, the load address **`0x01C0A800`**, the same name;
- six **LZ4 block** chunks that share one dictionary, decompressing to 4 KiB each (the last `0xB1C`). The inner data CRC
  covers the decompressed image.

The bootloader decompresses it to `0x01C0A800` and starts it ([Update Protocol and Installers](Update-Protocol-and-Installers.md)).

### More entries: resources and extra regions

felucca-bsp's loader (fm1-base builds) knows two more entry types. Both are written **after** the app, and a failure
in them never fails the install. **[CODE]**
- **Type 101, `res.bin`**: an app's resource region, with its flash address at offset `0x14`, "as Melodee's". It is
  kept when the flash already holds it whole, left alone over saves, and written only in whole sectors of the data
  area ([18eadad](https://github.com/hugelton/felucca-bsp/commit/18eadad), 2026-11-13). So a host that serves only
  the package (The Firmware Hub, M-UPGRADE) brings the app's resources too.
- **Type 102, extra regions**: an app's read-only data outside the app and data areas (fm1-base
  `tools/fm1pkg_make.py --extra`), each written first sector last
  ([d5e5d90](https://github.com/hugelton/felucca-bsp/commit/d5e5d90), 2026-11-26).

A firmware may also ask for a bigger app area, ending up to `0xE0000` instead of `0x93000` (felucca-bsp `FL_APP_END`,
[13e4c44](https://github.com/hugelton/felucca-bsp/commit/13e4c44); fm1-base `app_end`, whose README gives `app.bin`
up to 823,228 bytes at `0xCE000`). fm1-base's loaders erase an older app's code above their own end; Felucca's and
M-VAVE's loaders don't (fm1-base README, "A bigger app slot"). **[CODE]**

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
`cfg/eq_cfg_hw.bin`. Two `uboot.boot` versions are in use: SDK 1.1.9's in FM-1 V15 and the SMK-37's 016, SDK
1.2.13's in FM-1 V16 and FM1 Open's kit ([Boot Chain and Bootloaders](Boot-Chain-and-Bootloaders.md)). On the licence
of these binaries, see the same page. Open projects don't commit them; the build reads them from an SDK checkout and some check
them by SHA-256. Because the SDK's `uboot.boot` is in the head, **installing a full package with a stock head puts
the stock bootloader back** where the installer writes the head. The Felucca loader writes only the app area.

## Tools that read or write packages

- Felucca / felucca-bsp: `fm1pkg_make.py` / `fm1pkg.py` (build), `fm1_install.py` (install).
- [jl-misctools](https://github.com/kagaimiq/jl-misctools): JieLi container utilities.
- [FM-1-RE](https://github.com/AL-255/FM-1-RE): unpacked stock V13 and V14 packages and their metadata
  (`firmware-images/`), and `scripts/extract_ota_loader.py`, which checks and unpacks the stock `ota.bin`.
- [M-Flasher](https://github.com/jvitkauskas/m-flasher): installs M-VAVE's `.fwsc` through download mode and
  shows a flash map of what will change. **[HW]** It installs community packages too: DUB-1's `.fwsc` went on
  at the first try over SHRUB v1.29 on a UC25HQ80 unit (`download-mode`, then FLASH FIRMWARE), and back to
  stock V15 the same way (fm1-dub#14). M-UPGRADE (3.11.1211, macOS) refuses a community package: "Invalid OTA
  firmware: missing or incomplete model, version, or metadata".

## Sources

- Felucca's `tools/fm1pkg_make.py`, `tools/fm1_install.py` (GPL-3.0-only, © Leo Kuroshita).
- [m-bootloader SPEC.md](https://github.com/jvitkauskas/m-bootloader/blob/main/docs/SPEC.md) §5;
  [docs/STOCK-V15-V16.md](https://github.com/jvitkauskas/m-bootloader/blob/main/docs/STOCK-V15-V16.md) (121cfbe).
- Felucca's `web/fm1pkg.js` (commit 5044cce).
- [felucca-bsp](https://github.com/hugelton/felucca-bsp) commits 18eadad, d5e5d90, 13e4c44 (MPL-2.0);
  [fm1-base](https://github.com/charlesvestal/fm1-base) README "A bigger app slot" (3039770).
- M-VAVE's [FM1 Open](https://github.com/m-vave/fm1_open) `sdk/WebUpdate.js`, `app/version.h` (4c7496d);
  [SMK37 Open](https://github.com/m-vave/smk37_open) `app/version.h`, `sdk/WebUpdate.js` (b2d6bcb).
- [FM-1-RE](https://github.com/AL-255/FM-1-RE) (WTFPL): [architecture](https://github.com/AL-255/FM-1-RE/blob/main/docs/architecture.md) §2;
  [`docs/io/11-ota-protocol.md`](https://github.com/AL-255/FM-1-RE/blob/main/docs/io/11-ota-protocol.md) ("Known open items");
  `firmware-images/v13/raw_fw/FM-1.fwsc_unpack/decrypted.bin` and `analysis/device/ota-loader/ota_stock.bin` (the CRC check).
- felucca-bsp `tools/fm1pkg.py`, `tools/fm1_install.py` (MPL-2.0); m-bootloader `tools/bankcb.py` (MIT): the CRC16's initial value.
- [fm1-dub#14](https://github.com/sbegueria/fm1-dub/issues/14): M-Flasher and M-UPGRADE with DUB-1's package (2026-11-22).

---

Unofficial community wiki. Not affiliated with M-VAVE or JieLi. Text under [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/); code excerpts keep their own licences ([License](License.md)). Facts are tagged **[HW] [SDK] [RE] [CODE] [?]**: see [Home](Home.md). Found an error? See [Contributing](Contributing.md). [Home](Home.md)
