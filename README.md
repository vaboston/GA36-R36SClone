# GA36 (R36S clone, Allwinner A33) — SD card image with a large ROMS partition

> **Note on the chip marking:** the SoC package is printed **"RK3326"**, but this
> is a **fake marking**. The hardware is actually an **Allwinner A33** (quad-core
> Cortex-A7 tablet SoC), proven by the boot chain: `eGON.BT0` boot0 signature and
> `sun8i` U-Boot (both Allwinner-proprietary — a real RK3326 boots via Rockchip
> TPL/SPL with a completely different signature), a `sun8i` kernel, and
> `LIBREELEC_PROJECT="Allwinner"` / `COREELEC_DEVICE="A33"` in `/etc/os-release`.
> An Allwinner A33 firmware physically cannot boot on a Rockchip RK3326. The
> mislabeled chip is a known trick on these clones — which is also why no
> ArkOS/Batocera RK3326 build (or any other "R36S" custom firmware) can ever run
> on them.

## Background

The stock firmware (`r36s-a33-recovery.img`, EmuELEC 4.7-Nexus custom A33 build)
allocates only **5.4 MB** to the ROMS partition (`p1`, FAT32) — installing ROMs is
therefore impossible, and everything past 2.4 GB of the SD card goes unused.

On top of that, the console has **internal storage (eMMC)** that registers as
`mmcblk0` *before* the SD card. The stock firmware then mounted the eMMC's ROMS
partition (the ~100 factory-preloaded NES games) and completely ignored the SD
card, whatever its size.

## The fix: `r36s-ga36-a33-128GB-roms-SMALL.img`

A 2.4 GB image based on the stock firmware, with two modifications:

1. **MBR**: partition `p1` (ROMS, FAT32) resized from 5.4 MB → **115.64 GiB**
   (sized to fit on any "128 GB" card, with a safety margin).
2. **Patched initramfs**: at boot, the system locates the SD card (identified by
   its ~115 GiB partition — impossible to confuse with the internal eMMC) and
   mounts `/flash`, `/storage` and `/storage/roms` **from the SD card**, whatever
   `mmcblkX` number the kernel assigns.

Note: this image does **not** contain a pre-written FAT32 filesystem (hence its
small size and fast flash time). You must create it yourself (step 4).

## Download (Releases)

The GitHub **Releases** page hosts the compressed image: a single
`r36s-ga36-a33-128GB-roms-SMALL.img.gz` file (built with `gzip -k` on macOS).
This is the only file to download — the raw `.img` is too large to upload.

Decompress it before flashing:

```bash
# macOS / Linux
gunzip r36s-ga36-a33-128GB-roms-SMALL.img.gz
# or, to keep the .gz:
gzip -dk r36s-ga36-a33-128GB-roms-SMALL.img.gz
```

> On macOS, `gzip` may refuse to decompress a file whose uncompressed name
> already exists — remove the old `.img` first, or use `gzip -dkf`.

## Install (Linux)

```bash
# 0. Decompress the release archive (see "Download" above)

# 1. Identify the SD card (check the 128G size — never guess the device name)
lsblk -o NAME,SIZE,MODEL,RM,TRAN
#    → typically /dev/sda (USB reader) or /dev/mmcblk0 (internal reader)

# 2. Unmount all its partitions (the desktop auto-mounts them)
for p in /dev/sdX?*; do sudo umount "$p" 2>/dev/null; done

# 3. Flash (~2-3 min only)
sudo dd if=r36s-ga36-a33-128GB-roms-SMALL.img of=/dev/sdX bs=4M status=progress conv=fsync

# 4. Create the FAT32 on p1 (10 seconds) — MANDATORY
lsblk /dev/sdX          # sdX1 must show ~115.6G
sudo umount /dev/sdX1 2>/dev/null
sudo mkfs.vfat -L EEROMS /dev/sdX1
```

## Install (Windows) — ⚠️ NOT TESTED

The following was not tested (no Windows machine was used to build/validate this
image) — but the image itself is OS-agnostic raw bytes, so any raw-image writer
should work. Feedback welcome.

```powershell
# 0. Decompress the .img.gz
#    PowerShell 10+ ships with bsdtar, or use 7-Zip (right-click → Extract Here)
tar -xzf r36s-ga36-a33-128GB-roms-SMALL.img.gz

# 1. Identify the SD card (check the ~119 GB size — never guess the disk number)
Get-Disk
#    → note the DiskNumber, e.g. 2

# 2. Flash with a raw-image tool, e.g.:
#    - balenaEtcher: select the .img → select the disk → Flash
#    - Rufus: select the .img (show "*.*" in the file filter) → GPT/MBR doesn't
#      matter, it's a raw dd write → Start (accept the ISO-hybrid warning)
```

3. Create the FAT32 on p1 — **MANDATORY**, and this is the tricky part on
   Windows: the ~115 GiB partition is bigger than the 32 GB limit of the native
   Windows FAT32 formatter, so `format` / `Format-Volume` will **fail**. Use a
   tool that supports large FAT32 volumes, e.g.:
   - **Rufus** (it can reformat an existing large partition as FAT32), or
   - [guiformat (FAT32 Format)](http://ridgecrop.co.uk/index.htm?guiformat.htm),
     targeting the EEROMS partition.

   Note: there is no way to skip this step — the patched initramfs mounts the
   ROMS partition but does **not** create the filesystem, and the console cannot
   format it itself.

## Setting up ROMs

1. Insert the card into the console and boot once.
   → The system automatically creates the empty folders (`gba`, `nes`, `snes`,
   `psx`, …) at the root of the EEROMS partition.
2. Put the card back into the PC:
   - **ROMs** → into the folders on the big EEROMS volume, e.g.
     `roms/gba/mygame.gba` (a game dropped at the partition root is invisible to
     EmulationStation).
   - **BIOS** → in the same folders on the big EEROMS volume, next to your
     ROMs — e.g. `gba_bios.bin` in `roms/gba/`.
3. Put the card back into the console and reboot.

## Technical details of the stock image

| Offset | Size | Partition | Content |
|---|---|---|---|
| 8 KiB | — | raw | boot0 (eGON.BT0, Allwinner A33) |
| ~19.1 MiB | — | raw | U-Boot sun8i (14/07/2025) |
| 20 MiB | — | raw | Allwinner native partition table (`softw411`) |
| 36 MiB | 32 MB | p2 FAT16 "Volumn" | bootloader assets (bootlogo, battery icons, fonts) |
| 68 MiB | 16 MB | p5 | U-Boot environment (`root=mmcblk0p7`, `disk=mmcblk0p8`) |
| 84 MiB | 32 MB | p6 | Android bootimg: 12.6 MB kernel + 3 MB ramdisk (LibreELEC-style init) |
| 116 MiB | 768 MB | p7 FAT16 "EMUELEC" | `SYSTEM` squashfs 406 MB (EmuELEC 4.7-Nexus A33) |
| 884 MiB | 1.5 GB | p8 ext4 | `/storage` (configs, cores, saves, bios) |
| 2420 MiB | 115.64 GiB | p1 FAT32 "EEROMS" | **ROMS** (mounted at `/storage/roms`) |

The kernel always boots from the SD card (the Allwinner BROM prioritizes the SD
slot), but the stock firmware mounted root/storage/roms from `/dev/mmcblk0pX` —
the number given to whichever device registered first, here the internal eMMC.
The initramfs patch resolves the real devices through `/sys/block` by looking for
a ROMS partition ≥ 48 GiB.

OS updates without reflashing: drop a `.tar`, `.img` or `.img.gz` into the
`.update` folder on the ROMS partition (native EmuELEC mechanism).

### Why not ArkOS?

There is no official ArkOS for the Allwinner A33 (ArkOS targets RK3326/RK3566/
H700). Despite the "RK3326" marking printed on the package (fake marking, see
note at the top), no RK3326 firmware can boot on this hardware. Traces of an
experimental A33 port exist inside the stock firmware; worth watching the
GA36 / R36A communities.
