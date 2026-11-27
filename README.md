# IMX462 camera setup on Raspberry Pi

[![Driver ships with OS](https://img.shields.io/badge/driver-ships_with_OS-blue)](https://github.com/raspberrypi/linux/blob/rpi-6.18.y/drivers/media/i2c/imx290.c)
[![RPi OS Bookworm | Trixie](https://img.shields.io/badge/RPi_OS-Bookworm_%7C_Trixie-blue?logo=raspberrypi)](https://www.raspberrypi.com/software/operating-systems/)
[![Kernel 6.6+](https://img.shields.io/badge/kernel-6.6%2B-blue?logo=raspberrypi)](https://github.com/raspberrypi/linux/tree/rpi-6.6.y)

Raspberry Pi setup for Sony IMX462, a 2 MP 1/2.8" STARVIS low-light CMOS sensor with 2.9 µm pixels. Raspberry Pi OS supports it out of the box with `imx290` family driver.

- 2-lane and 4-lane MIPI CSI-2 (up to 891 Mbps/lane)
- 10-bit and 12-bit RAW output
- 1920×1080 @ 60 fps (full resolution)
- 1280×720 @ 60 fps (sensor crop)
- Color and mono variants
- HCG mode via module parameter

![Kurokesu camera modules connected to a Raspberry Pi 5.](docs/kurokesu-on-pi.jpg)

*IMX462 camera modules are available at [kurokesu.com](https://www.kurokesu.com/item/462C-CSI)*

## Setup

Connect camera to CSI port with Pi powered off.

For mono modules, enable Kurokesu apt archive first. Stock Raspberry Pi OS does not ship `imx462_mono.json` tuning file yet, Kurokesu `libcamera` fork does:

```bash
curl -fsSLO https://apt.kurokesu.com/setup.sh
sudo sh setup.sh
```

*With archive enabled, next step upgrades stock `rpicam-apps` and `libcamera` to Kurokesu forks.*

Update OS and reboot:

```bash
sudo apt update && sudo apt full-upgrade -y
sudo reboot
```

Edit boot configuration:

```bash
sudo nano /boot/firmware/config.txt
```

Make two changes:

1. Find `camera_auto_detect` near the top and set it to `0`:

```ini
camera_auto_detect=0
```

2. Add `dtoverlay=imx462` under the `[all]` section at the bottom of the file:

```ini
[all]
dtoverlay=imx462
```

*If camera is connected to cam0 port, use `dtoverlay=imx462,cam0` instead. See [cam0](#cam0).*

Save and exit.

`config.txt` changes take effect after reboot:

```bash
sudo reboot
```

Verify camera is detected:

```bash
rpicam-hello --list-cameras
```

Expected output:

```
Available cameras
-----------------
0 : imx462 [1945x1097 12-bit RGGB] (/base/axi/pcie@1000120000/rp1/i2c@70000/imx290@1a)
    Modes: 'SRGGB10_CSI2P' : 1280x720 [60.00 fps - (332, 188)/1280x720 crop]
                             1920x1080 [60.00 fps - (12, 8)/1920x1080 crop]
           'SRGGB12_CSI2P' : 1280x720 [60.00 fps - (332, 188)/1280x720 crop]
                             1920x1080 [60.00 fps - (12, 8)/1920x1080 crop]
```

Start live preview:

```bash
rpicam-hello -t 0
```

On headless systems, capture a still image instead:

```bash
rpicam-still -o test.jpg
```

## Driver

IMX462 runs on `imx290` family driver, which also covers IMX290 and IMX327. Camera tools list sensor as `imx462`, while kernel log and module parameters use driver name `imx290`.

## dtoverlay options

`imx462` overlay supports comma-separated options to override defaults:

| option | description | default |
|--------|-------------|---------|
| [`cam0`](#cam0) | Use cam0 port instead of cam1 | cam1 |
| [`4lane`](#4lane) | Use 4-lane MIPI CSI-2 (if wired) | 2 lanes |
| [`clock-frequency=<Hz>`](#clock-frequency) | Sensor input clock (Hz) | 37125000 |
| [`mono`](#mono) | Mono sensor variant | color |
| `rotation=<deg>` | Sensor mounting rotation | 0 |
| `orientation=<n>` | Camera orientation (0 front, 1 back, 2 external) | 2 |
| `media-controller=0` | Use legacy unicam driver | on |

### cam0

If camera is connected to cam0 port, append `,cam0`:

```ini
dtoverlay=imx462,cam0
```

### 4lane

To enable 4-lane MIPI CSI-2, append `,4lane`:

```ini
dtoverlay=imx462,4lane
```

> [!WARNING]
> Before using `4lane`, confirm your camera port actually supports 4-lane MIPI CSI. Not all Raspberry Pi models and carrier boards provide 4-lane MIPI CSI on both ports.

4 lanes halve link frequency. Modes and framerates stay the same, see [Output formats](#output-formats).

### clock-frequency

Kurokesu IMX462 modules carry a 37.125 MHz oscillator, which matches overlay default. Driver also supports 74.25 MHz.

### mono

For mono modules, append `,mono`:

```ini
dtoverlay=imx462,mono
```

Mono modules output `Y10` and `Y12` grayscale formats instead of Bayer.

Mono modules need Kurokesu `libcamera` fork, see [Setup](#setup).

> [!TIP]
> Options can be combined. Example (cam0, 4-lane, mono):
>
> ```ini
> dtoverlay=imx462,cam0,4lane,mono
> ```

## Output formats

| Link frequency | Data rate / lane | Lanes | Bit depth | Width | Height | Max FPS |
|---|---|---|---|---|---|---|
| **1280×720 (sensor crop)** | | | | | | |
| 148.5 MHz | 297 Mbps | 4 | 10-bit | 1280 | 720 | 60 fps |
| 148.5 MHz | 297 Mbps | 4 | 12-bit | 1280 | 720 | 60 fps |
| 297 MHz | 594 Mbps | 2 | 10-bit | 1280 | 720 | 60 fps |
| 297 MHz | 594 Mbps | 2 | 12-bit | 1280 | 720 | 60 fps |
| **1920×1080 (full resolution)** | | | | | | |
| 222.75 MHz | 445.5 Mbps | 4 | 10-bit | 1920 | 1080 | 60 fps |
| 222.75 MHz | 445.5 Mbps | 4 | 12-bit | 1920 | 1080 | 60 fps |
| 445.5 MHz | 891 Mbps | 2 | 10-bit | 1920 | 1080 | 60 fps |
| 445.5 MHz | 891 Mbps | 2 | 12-bit | 1920 | 1080 | 60 fps |

## HCG mode

IMX462 supports high conversion gain (HCG) mode for improved signal-to-noise ratio in low light, at cost of lower full-well capacity. Default is low conversion gain (LCG).

HCG is set via module parameter. New value applies at next stream start:

```bash
# 0=off, 1=on
echo 1 | sudo tee /sys/module/imx290/parameters/hcg_mode
```

> [!NOTE]
> Module parameter is global: it applies to all sensors on `imx290` driver, including IMX290 and IMX327. libcamera is not aware of HCG, so reported analog gain does not include it.

## FAQ

### Why does rpicam-hello say "No cameras available!"?

Check [Setup](#setup) steps first: `camera_auto_detect=0`, overlay line under `[all]` and `cam0` option matching the port in use. Then check kernel log:

```bash
dmesg | grep imx290
```

`Could not initialize device` means sensor does not answer on I2C:

```
imx290 0-001a: Error writing reg 0x3000: -121
imx290 0-001a: error -EREMOTEIO: Could not initialize device
imx290 0-001a: probe with driver imx290 failed with error -121
```

Check camera port and cable seating, with Pi powered off.

`Could not create camera helper for imx462` means libcamera predates IMX462 support. Update OS as shown in [Setup](#setup).

### Why does mono module image have blue tint?

Without `mono` option, driver reports Bayer format and libcamera processes grayscale image as color. Append `,mono` to overlay line, see [mono](#mono).

### Why does mono module fail with "imx462_mono.json not found"?

With `mono` option, libcamera looks for `imx462_mono.json` tuning file, which stock Raspberry Pi OS does not ship yet:

```
ERROR IPAProxy ipa_proxy.cpp:189 Configuration file 'imx462_mono.json' not found for IPA module 'rpi/pisp'
```

Kurokesu `libcamera` fork ships it. Enable Kurokesu apt archive as shown in [Setup](#setup), then update OS and reboot.

To stay on stock packages, use IMX296 mono tuning file instead, which has same black level:

```bash
rpicam-hello -t 0 --tuning-file /usr/share/libcamera/ipa/rpi/pisp/imx296_mono.json
```

On Pi 4 and older, replace `pisp` with `vc4`.

## Credits

- FRAMOS and [Manivannan Sadhasivam](https://github.com/Mani-Sadhasivam) (Linaro) for original [imx290 driver](https://github.com/raspberrypi/linux/blob/rpi-6.18.y/drivers/media/i2c/imx290.c)
- [Dave Stevenson](https://github.com/6by9) of Raspberry Pi for IMX462 support in driver and [imx462 overlay](https://github.com/raspberrypi/linux/blob/rpi-6.18.y/arch/arm/boot/dts/overlays/imx462-overlay.dts)
- [geoffrey-vl](https://github.com/geoffrey-vl) for libcamera IMX462 helper and tuning ([raspberrypi/libcamera#207](https://github.com/raspberrypi/libcamera/pull/207))
