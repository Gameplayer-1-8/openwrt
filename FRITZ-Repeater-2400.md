# OpenWrt on the AVM FRITZ!Repeater 2400

Experimental but stable support for the AVM FRITZ!Repeater 2400 (`avm,fritz2400`).
This device is **not** supported by upstream OpenWrt and there is no wiki page
for it. Everything here was worked out on a single unit; it has been running
without a reboot since the day it was flashed.

**Both radios work**, both use AVM's own calibration data taken out of the
`urlader` partition:

| | |
|---|---|
| 2.4 GHz | QCN5502 WMAC in the SoC, 3x3, `ath9k` |
| 5 GHz | QCA9984 over PCIe, 4x4, VHT80/VHT160 capable, `ath10k-ct` |
| Ethernet | 1x GBit, AR8033 PHY over SGMII |
| LEDs | 5x RSSI, power, WLAN, LAN — 74HC595 shift register |
| Button | Connect |
| Sensor | LM75 (probes, but has never returned a temperature) |

802.11s mesh works too, with `wpad-openssl` or `wpad-mesh-openssl` installed
on top.

## Hardware

```
SoC     Qualcomm Atheros QCN5502
RAM     128 MB
Flash   64 MB SPI NOR — Winbond W25Q512JV, JEDEC 0xEF4020
        (NOT the Macronix MX25L51245G mentioned in the forum thread;
         the TFFS entry 0x01cb reads "spinor=0xEF4020,cpu=0x2170")
Boot    AVM EVA 1.11013, FTP on 192.168.178.1, user/pass adam2/adam2
```

Flash layout, taken from EVA's own environment variables `mtd0`..`mtd6`
(TFFS ids `0x01b0`..`0x01b6`), not from a boot log:

```
0x00000000-0x00020000  urlader          EVA mtd2  boot loader + calibration
0x00020000-0x00090000  tffs (1)         EVA mtd3  config, MAC addresses
0x00090000-0x02000000  reserved-kernel  EVA mtd1  FRITZ!OS slot B
0x02000000-0x02020000  caldata          EVA mtd6  128 KiB, ERASED on this unit
0x02020000-0x02090000  tffs (2)         EVA mtd4  config backup
0x02090000-0x04000000  firmware         EVA mtd5  FRITZ!OS slot A  <- OpenWrt
```

### Calibration data is not where you would look for it

The `caldata` partition is **completely erased** — 131072 of 131072 bytes are
`0xFF`. Both calibration blobs live zlib-compressed inside `urlader`, as a
chain of `[be16 id][be16 len][payload]` records starting at `0x980`,
terminated by id `0xffff`:

| id | contents | extraction |
|---|---|---|
| `0x0207` | 2.4 GHz WMAC EEPROM, 0x540 bytes | `fritz_cal_extract -r -i 4 -s 0x980 -e 0x207 -l 0x540` |
| `0x0208` | QCA9984 pre-calibration, 12064 bytes | `fritz_cal_extract -i 1 -s 0x980 -e 0x208 -l 12064` |

`-r` (byte-reverse) is only correct because the decompressed size is **odd**
(4097 bytes): `buffer_reverse(buf, datasize - 1)` leaves two bytes in the
middle untouched on an even length.

The EEPROM must be read as `0x540` bytes, not the `0x440` of
`struct ar9300_eeprom`. That is what `compatible = "qca,qcn5502-wifi"` selects,
through the device ID. With the wrong compatible there is no error message —
just silent misbehaviour, because the blob is large enough either way.

Do not try id `0x212` for the QCA9984: that is the FRITZ!Repeater **3000**.
And do not let the ath10k handler try `0x0207` first — it returns a false
success (rc=0, 4096 bytes, magic `5aa5`) that ath10k pushes into the card
without checking the length.

## Building

```sh
git clone https://github.com/Gameplayer-1-8/openwrt.git
cd openwrt
git checkout avm-fritz-repeater-2400-6.12
./scripts/feeds update -a && ./scripts/feeds install -a
cp FRITZ-Repeater-2400.config .config
make defconfig
make -j$(nproc)
```

`FRITZ-Repeater-2400.config` is the diffconfig of the build that produced the
released images, LuCI included. Optional lines you can drop: everything
`luci*`/`uhttpd*`/`rpcd*` (saves ~2.2 MB) and everything `openssl`/`wpad-*`
(those only build extra packages for 802.11s mesh, they are not in the image).

Four images are built. The diagnostic variants exist because bringing this
device up without a serial console is otherwise guesswork:

| variant | 5 GHz | 2.4 GHz | purpose |
|---|---|---|---|
| *(none)* | yes | yes | what you want |
| `-nopcie` | no | yes | fallback if PCIe misbehaves |
| `-min` | no | no | no LEDs, no I2C, no wireless |
| `-heartbeat` | no | yes | LEDs only, power LED blinks |

### Two traps in the build system

**`DEVICE_PACKAGES` only sets `CONFIG_DEFAULT_<pkg>=y`.** That acts as
`default y if DEFAULT_<pkg>` on `CONFIG_PACKAGE_<pkg>` — but only while that
symbol has no value yet. If your `.config` already contains
`# CONFIG_PACKAGE_kmod-ath10k-ct is not set`, `make defconfig` keeps it and the
package is never built. Silently. On a fresh tree this cannot happen; if you
are adapting an existing config, set the symbols explicitly.

**`ath79/generic` has no default for `TARGET_INITRAMFS_COMPRESSION`.** Without
`CONFIG_TARGET_INITRAMFS_COMPRESSION_LZMA=y` the cpio sits uncompressed in the
kernel: 26 MiB unpacked from `0x80060000`, ending at `0x81A62BB2` — 9.5 MiB
past EVA's `firstfreeaddress = 0x8114A204`. The image then overwrites EVA's own
structures and the device does nothing at all. Six completely different device
trees behaved identically before this was found.

Always check field `+0x14` of the EVA header (`datasize32`, little endian)
against `firstfreeaddress - loadaddress` = 17 736 196 bytes:

```sh
python3 -c "import struct,sys
d=open(sys.argv[1],'rb').read(32)
u=struct.unpack_from('<I',d,20)[0]
print('%d bytes, %.2f MiB, %s'%(u,u/1048576,'OK' if u<17736196 else 'TOO BIG'))" \
  bin/targets/ath79/generic/openwrt-ath79-generic-avm_fritz2400-initramfs-kernel.bin
```

## Flashing

You need a PC with **192.168.178.10/24** (for EVA) and **192.168.1.10/24**
(for OpenWrt afterwards) on the same wired interface. Put both addresses on it
at once so nothing has to be changed halfway through.

EVA only listens for a few seconds after power-on, so start the tool first and
plug the device in afterwards.

### Step 1 — RAM boot, touches no flash at all

Do this first, every time. If anything is wrong, pull the plug and FRITZ!OS
comes back untouched.

```sh
python3 eva_ramboot.py --offset 0x80060000 192.168.178.1 \
    openwrt-ath79-generic-avm_fritz2400-initramfs-kernel.bin
```

Two things that cost a lot of time here:

* Send `TYPE I` explicitly. Switching from `storbinary()` to `transfercmd()`
  silently drops binary mode and the upload arrives corrupted.
* `end - start` announced in the `STOR` command **must** equal the number of
  bytes actually sent. `eva_ramboot.py` rounds the load address down with
  `& ~0xfff`, so pad the image to a multiple of 4096 first — otherwise EVA
  waits forever for the rest.

After 30–60 seconds the device answers on **192.168.1.1**, SSH included.

### Step 2 — tell EVA which slot to boot

**Do not skip this.** `sysupgrade` always writes to `firmware` at `0x02090000`.
Which of the two 32 MiB slots EVA boots is decided by `linux_fs_start`, and
that variable **does not exist** on this device — so EVA boots the other slot
at `0x00090000`. Without this step `sysupgrade` reports success and the device
cheerfully starts FRITZ!OS again.

```
ftp> quote SETENV linux_fs_start 1
ftp> quote GETENV linux_fs_start      # must answer 1
```

To go back to FRITZ!OS later, set it to `0`.

### Step 3 — install

```sh
scp -O openwrt-ath79-generic-avm_fritz2400-squashfs-sysupgrade.bin \
    root@192.168.1.1:/tmp/
ssh root@192.168.1.1 "setsid sh -c 'sysupgrade -n /tmp/openwrt-*.bin' </dev/null &"
```

`-O` is required: dropbear has no SFTP server, and newer OpenSSH defaults to
SFTP for `scp`.

`setsid` is required too. `sysupgrade` announces *"Closing all shell sessions"*
and kills its own SSH session along with them, before it can hand over to
`do_stage2` via ubus. You then get

```
Command failed: ubus call system sysupgrade { ... } (Connection failed)
```

which is misleading — on the first attempt it had already written the flash.
Never trust that message; look at the target slot instead:

```sh
dd if=/dev/mtd5 bs=1 count=16 2>/dev/null | hexdump -C
# 81 12 ed fe 37 cf 24 00   stock FRITZ!OS
# 81 12 ed fe e4 44 2a 00   this OpenWrt image (reclength 0x002a44e4)
```

### After the first boot

`mtdsplit_eva` now splits the image and you get `kernel`, `rootfs` and a
24.5 MB `rootfs_data` overlay. On the **first** boot the overlay is still a
tmpfs (`jffs2 not ready yet, using temporary tmpfs overlay`) — settings only
survive from the second reboot onwards.

## Recovery

Nothing here ever touches EVA itself, and `reserved-kernel` at `0x00090000`
keeps an untouched copy of stock FRITZ!OS. To go back:

1. `quote SETENV linux_fs_start 0` over EVA's FTP, then power-cycle, or
2. flash AVM's recovery image (`FRITZ.Repeater_2400-08.20.image`) the usual way

## What the patches do

Four commits, the first three of which are not specific to this device:

**`generic: mtdsplit_eva: return -ENOENT`** — `add_mtd_partitions()` treats
every error but `-ENOENT` as fatal and deletes the *entire* partition table of
the parent device. `mtdsplit_eva` returned `-EINVAL` from all three "no EVA
image here" paths, so a firmware slot holding stock FRITZ!OS took all of
`/proc/mtd` down with it. Affects every AVM device with two slots.

**`ath79: fix PCIe bring-up on QCA956x`** — three independent causes, each one
enough on its own:

1. The PCIe PLL is at `PLL + 0x14`, not `0x10`. On QCA956x offset `0x10` is
   the CPU/DDR/AHB clock control; clearing what the driver believes to be the
   PLL bypass bit reprograms the clock tree underneath the running CPU and
   hangs the machine with no console output. EVA leaves `PLL + 0x14` at
   `0x40010400` — power down and bypass both set — so the block has no clock.
2. Root complex 1 is gated by `RST_RESET2` (`0x180600c0`) bits 6 and 7 as well,
   which `hw_init()` never released. The CRP window faults until they are
   cleared; afterwards it reads `0xa02a168c`.
3. PERST# must be deasserted **after** the reference clock is stable. A
   `gpio-hog` releases the endpoint during gpiolib init, about seven seconds
   before `hw_init()` brings up the PLL, and a QCA9984 never trains its link
   after that. The reset line therefore belongs to the driver, not to a hog.

**`mac80211: ath9k: add QCN550x support`** — Wenli Looi's series from
[openwrt PR #9389](https://github.com/openwrt/openwrt/pull/9389), ported to
mac80211 6.18.

**`ath79: add support for AVM FRITZ!Repeater 2400`** — device tree, image
recipe, LED and network setup, and the two calibration hotplug handlers.

## Credit

The problem was first described and narrowed down by **neorob85** in the
OpenWrt forum thread
[Adding support for Fritz!Repeater 2400](https://forum.openwrt.org/t/adding-support-for-fritz-repeater-2400/248477),
including the original device tree from FRITZ!OS 8.20 and the observation that
the boot hangs in `ath79_pll_wr()` inside `pci-ar724x.c`. That pointer was
correct and is what the PCIe fix above is built on.
