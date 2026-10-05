# ThinkPad T530 ICC profile on Ubuntu

Version 1.0.0. The profiles and `TPLCD.INF` in `MONITOR_ICC/` are unmodified copies of Lenovo's ThinkPad Monitor INF/Color Profile, driver 5.03.0.0 (14 August 2013). They are kept because Lenovo no longer supports the ThinkPad T530. See `NOTICE` for that copyright. Lenovo does not supply or support this package. `t530-icc` and this README are covered by `LICENSE`.

Unpack the release and run it from that directory. No install step. The script reads `MONITOR_ICC/` beside itself.

```sh
tar -xzf t530-icc-1.0.0.tar.gz
cd t530-icc-1.0.0
./t530-icc --query --host rusty.local
./t530-icc --host rusty.local
```

`TPLCD.INF` is the table that picks one of the `.icm` profiles. The notes below record how that choice was made for the T530 named rusty.

## Host

rusty is a ThinkPad T530 (machine type `24298B8`) running Ubuntu 26.04.1. SSH as `plastic@rusty.local` uses the shared key already on this machine.

The internal panel is on the Intel GPU, connector `LVDS-1`. The NVIDIA connectors were disconnected.

## List the panel

On rusty:

```sh
cat /sys/class/dmi/id/product_name
cat /sys/class/dmi/id/product_version

for d in /sys/class/drm/card*-*; do
  printf '%s %s\n' "$(basename "$d")" "$(cat "$d/status")"
done

base64 -w0 /sys/class/drm/card1-LVDS-1/edid; echo
```

`colormgr get-devices` shows the same display as an xrandr device. Its current profile is the EDID profile colord generated (`~/.local/share/icc/edid-*.icc`), not a Lenovo ICM.

Reading taken 5 Oct 2026:

| Field | Value |
| --- | --- |
| Connector | `card1-LVDS-1`, connected |
| Modes | 1920x1080 |
| EDID vendor / product | AUO `0x11ED` (Windows id `AUO11ED`) |
| Manufacture | 2012 |
| Name strings | `AUO`, `B156HTN01.1` |
| Gamma byte | `0x78` → 2.2 |
| Chromaticity (x, y) | R 0.676, 0.314; G 0.215, 0.665; B 0.141, 0.069; W 0.313, 0.329 |
| Gamut from those primaries | about 95% of NTSC 1953 |

That matches the T530 Full HD option in Lenovo’s PSREF: 15.6 inch, 1920x1080, LED, 270 nits, 95% gamut. The AUO datasheet for B156HTN01.1 lists the same resolution, 270 cd/m² typical, and 95% NTSC. It is a TN panel (viewing angles well under IPS), with a phosphor LED backlight, not an RGB-LED backlight.

## Select the file

`AUO11ED` is not a hardware id in `TPLCD.INF`. Lenovo’s rows use ids such as `Monitor\LEN40B2`. Match the panel the INF describes: size, resolution, and gamut class. Each install section’s `AddReg` line names an `[ICM…]` block, and that block sets `ICMProfile`.

15.6 inch rows in `TPLCD.INF`:

| Hardware id | Panel the INF describes | Profile |
| --- | --- | --- |
| `LEN40B0` | Wide HD 1366x768, Gamut60 | `TPLCD60.icm` |
| `LEN40B1` | Wide HD+ 1600x900, Gamut60 | `TPLCD60.icm` |
| `LEN40B2` | Wide FHD 1920x1080, LED, Gamut95 | `TPLCD95.icm` |
| `LEN40B3` | Wide FHD 1920x1080, RGB LED | `TPLCD.icm` |
| `LEN40B4` | Wide FHD 1920x1080, standard Gamut60 | `TPLCD60.icm` |
| `LEN40B5` | Wide FHD++ 2880x1620, IPS, Gamut72 | `TPLCD95.icm` |

The rusty panel is the `LEN40B2` case. Its install section is `TPLCDWFHDG95.Install`, which adds `ICMG95`:

```
[ICMG95]
HKR,,ICMProfile,0,"TPLCD95.icm"
```

Selected file: `MONITOR_ICC/TPLCD95.icm`.

`./t530-icc` does that lookup and installs the file with colord. It installs only when one 15.6 inch row matches the resolution and the EDID gamut class, or the EDID hardware id itself is in the INF. A gamut that falls between those rows, or two rows that name different files, aborts with no profile installed. `--profile` is the explicit override. `--query` prints the choice and does not install.

The other FHD files are for a different backlight or a 60% gamut panel. `TPFLX*.icm` is the FlexView / IPS family. `TPMB*.icm` is MaxBright. `TPLCD100.icm` belongs to the 17 inch RGB-LED Gamut100 row (`LEN406A`).

`TPLCD100.icm` happens to sit closer to this EDID’s primaries than `TPLCD95.icm` does. The preserved INF still assigns `TPLCD95.icm` to the 15.6 inch FHD 95% configuration, so that is the file this procedure selects.
