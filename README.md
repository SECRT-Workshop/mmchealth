# mmchealth

A small command line tool that reads the health and identity data from an eMMC chip and prints it in plain language.

eMMC controllers report wear in a register called EXT_CSD, and the values in it look like `0x03` or `0x01`. That is fine for a datasheet, but not much help when you are standing at a bench looking at a tablet, a single board computer, or a thin client and wondering how much life the flash has left. mmchealth reads the register, translates the numbers, and shows them as percentages, sizes and names.

## What it shows

- **Wear.** The two lifetime estimates from the controller, turned into a percentage range, a rough midpoint and a bar. The controller reports in steps of 10%, so `0x01` means somewhere between 0 and 10% used, `0x02` means 10 to 20%, and so on. `0x0B` means the chip has gone past its rated life.
- **Reserved blocks.** The pre-EOL status (normal, warning or urgent), which tells you how much of the spare block pool is gone.
- **An overall verdict.** GOOD, WARNING, CRITICAL, or UNKNOWN if the controller does not report health data at all.
- **Who made it.** Manufacturer, product name, serial number, manufacture date, and hardware and firmware revisions. The manufacturer is looked up from the ID the chip reports. If the ID is not in the built-in table, the raw ID is shown instead.
- **How big it is.** User area, boot partitions, RPMB, and any general purpose partitions that are set up.
- **Other details.** eMMC spec version, supported speed modes, current bus width and timing mode, cache size, background operations, command queueing and power-off notification.

## Install

Download the release tarball, extract using tar, then run the installation script:

```
tar xzf mmchealth-1.0.0-x86_64.tar.gz
./install.sh
```

The script asks for confirmation, then uses sudo to store the binary in `/usr/local/bin` with root ownership and mode 0755. If SELinux is in use, it also fixes the label. To remove it later, run `./install.sh --uninstall`.

The binary is statically linked for x86_64, you should be able to run this on most x64 based Linux systems without additional modification.

### Building from source

You need gcc and a static libc (on Debian and Ubuntu that is `libc6-dev`, on Fedora it is `glibc-static`).

```
make
```

That produces a stripped, statically linked `mmchealth` in the current directory.

## Usage

```
mmchealth [OPTION]... [DEVICE]
```

With no device given, mmchealth looks through `/sys/block` and reports every eMMC device it finds. SD cards are skipped. Reading EXT_CSD from a live device needs root, so run it with sudo.

| Option | Long form | What it does |
| --- | --- | --- |
| `-d DEV` | `--device=DEV` | Inspect one specific device, such as `/dev/mmcblk0` |
| `-f FILE` | `--file=FILE` | Decode a saved EXT_CSD dump instead of reading hardware |
| `-r` | `--raw` | Show the raw register values next to the decoded ones |
| `-x` | `--hexdump` | Print the full 512 byte EXT_CSD as a hex dump |
| | `--no-color` | Turn off colored output (`NO_COLOR` is also respected) |
| `-h` | `--help` | Show help and exit |
| `-V` | `--version`, `-ver` | Show version and license info and exit |

You can also pass the device as a plain argument, so `mmchealth /dev/mmcblk0` works the same as `mmchealth -d /dev/mmcblk0`.

## Examples

Check every eMMC device in the machine:

```
sudo mmchealth
```

Check one device and show the raw values too:

```
sudo mmchealth -r /dev/mmcblk0
```

Get the full hex dump along with the readable report:

```
sudo mmchealth -x -d /dev/mmcblk0
```

Decode a dump someone else sent you. Both a 512 byte binary file and the text file from debugfs (1024 hex digits) are accepted:

```
mmchealth -f ext_csd.bin
mmchealth -f /sys/kernel/debug/mmc0/mmc0:0001/ext_csd
```

Save the report to a file without color codes:

```
sudo mmchealth --no-color > emmc-report.txt
```

### Example output

This was decoded from a made up test dump, so there is no manufacturer section. On a real device the Device block also lists the vendor, product name, serial number and so on.

```
mmchealth (file) - eMMC health report

== Device ==
  Identity                   n/a (decoded from file; CID not available)
  Firmware (EXT_CSD)         0x0000A1
  eMMC spec version          5.1 / 5.1A

== Capacity ==
  User area                  7.28 GiB (7.82 GB)
  Boot partitions            2 x 4.00 MiB
  RPMB partition             4.00 MiB
  Logical sector size        512 bytes

== Health ==
  Wear, Type A (SLC/boost)   [#-------------------] ~5% used  (range 0-10%, about 95% remaining)
  Wear, Type B (MLC/TLC)     [#####---------------] ~25% used  (range 20-30%, about 75% remaining)
  Reserved block status      Normal (reserved blocks healthy)
  Overall                    GOOD

== Features ==
  Supported speed modes      HS26 HS52 DDR52-1.8V HS200-1.8V HS400-1.8V
  Current bus width          8-bit
  Current timing mode        HS400
  Cache size                 512 KiB (enabled)
  Background ops             supported, enabled
  Command queueing           supported
  Power-off notification     not enabled
```

## Reading the wear numbers

The two wear values are the controller's own guess, not a measurement taken by this tool. A few things worth knowing:

- Type A usually describes the part of the flash that runs in SLC mode (if any), and Type B usually describes the main MLC or TLC area. Vendors are free to interpret this differently, and many chips only fill in one of them.
- The estimate moves in 10% steps, so the midpoint shown by mmchealth is only a convenient number for reading, not a finer measurement.
- Some controllers report nothing at all and leave both values at `0x00`. In that case mmchealth says the data is not reported and gives an UNKNOWN verdict. That does not mean the chip is healthy or failing.
- Health reporting needs eMMC 5.0 or newer. Older chips are labeled as not supported.
- If the verdict says WARNING or CRITICAL, treat it as a prompt to back up the data and plan a replacement.

## Troubleshooting

**"cannot read EXT_CSD ... Permission denied"**
Run it with sudo. The register read goes through a block device ioctl, which is restricted to root.

**"no eMMC devices found"**
The machine may not have eMMC, or the device may be an SD card, which is skipped on purpose. You can point it at a device by hand with `-d`. In a container or minimal VM, `/sys/block` may not expose the device at all.

**Manufacturer shows as Unknown**
The chip reports an ID that is not in the built-in table yet. The raw ID is still displayed. If you need to request to add a vendor, open an issue with the ID and the vendor name.

**Everything says "not reported"**
That controller does not implement the lifetime registers. Changes in mmchealth's source code will not change that.

## License

mmchealth is free software. Copyright (C) 2026 SECRT Workshop Tech.

This program is free software; you can redistribute it and/or modify it under the terms of the GNU General Public License as published by the Free Software Foundation; either version 2 of the License, or (at your option) any later version.

This program is distributed in the hope that it will be useful, but WITHOUT ANY WARRANTY; without even the implied warranty of MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the GNU General Public License for more details.

You should have received a copy of the GNU General Public License along with this program; if not, see <https://www.gnu.org/licenses/>. 
