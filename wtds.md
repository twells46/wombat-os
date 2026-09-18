# Wombat Touchscreen and Display Setup

These instructions configure the Wombat's TSC2007 touchscreen and 800x480 HDMI panel on current Raspberry Pi OS with labwc.

Run the overlay build commands from the root of this repository.

## 1. Build and enable the touchscreen overlay

Install the required tools and headers:

```bash
sudo apt update
sudo apt install device-tree-compiler raspberrypi-kernel-headers
```

If `raspberrypi-kernel-headers` is unavailable, install the matching kernel headers instead:

```bash
sudo apt install device-tree-compiler linux-headers-$(uname -r)
```

Build the overlay:

```bash
cpp \
  -nostdinc \
  -I /lib/modules/$(uname -r)/build/include \
  -undef \
  -D__DTS__ \
  -x assembler-with-cpp \
  configFiles/tsc2007-overlay.dts |
dtc -@ -I dts -O dtb -o tsc2007.dtbo -
```

Install it and enable I2C plus the overlay:

```bash
sudo install -m 0644 tsc2007.dtbo /boot/firmware/overlays/tsc2007.dtbo
sudoedit /boot/firmware/config.txt
```

Add these lines under `[all]` (or another applicable section) in `config.txt`:

```ini
dtparam=i2c_arm=on
dtoverlay=tsc2007
```

On older Raspberry Pi OS releases, replace `/boot/firmware` with `/boot` in the paths above. Reboot:

```bash
sudo reboot
```

After reboot, confirm the driver bound and the input device exists:

```bash
ls -l /sys/bus/i2c/devices/1-0048
readlink -f /sys/bus/i2c/devices/1-0048/driver
grep -A10 -B2 'TSC2007' /proc/bus/input/devices
```

The input device should be named `TSC2007 Touchscreen`. For a raw touch test:

```bash
sudo apt install evtest
sudo evtest
```

## 2. Set the display mode

The panel resolution is **800x480**. In a running labwc session, test the mode:

```bash
wlr-randr --dryrun --output HDMI-A-1 --custom-mode 800x480@60Hz
```

Apply it:

```bash
wlr-randr --output HDMI-A-1 --custom-mode 800x480@60Hz
```

Persist it in labwc:

```bash
mkdir -p ~/.config/labwc
if [ ! -f ~/.config/labwc/autostart ]; then
    cp /etc/xdg/labwc/autostart ~/.config/labwc/autostart
fi
```

Append this line to `~/.config/labwc/autostart`:

```bash
wlr-randr --output HDMI-A-1 --custom-mode 800x480@60Hz
```

## 3. Configure touch mapping and calibration

Create or replace `~/.config/labwc/rc.xml` with:

```xml
<?xml version="1.0"?>
<openbox_config xmlns="http://openbox.org/3.4/rc">
	<touch
		deviceName="TSC2007 Touchscreen"
		mapToOutput="HDMI-A-1"
		mouseEmulation="yes"
	/>

	<libinput>
		<device category="TSC2007 Touchscreen">
			<calibrationMatrix>1.089352 -0.003025 -0.047122 0.006822 1.138903 -0.085055</calibrationMatrix>
		</device>
	</libinput>
</openbox_config>
```

Reload labwc:

```bash
labwc --reconfigure
```

If it does not reload, use:

```bash
kill -HUP "$(pgrep -xo labwc)"
```

## Quick checks

```bash
wlr-randr
sudo libinput list-devices | sed -n '/TSC2007 Touchscreen/,/^$/p'
```

`wlr-randr` should show `HDMI-A-1`; change `mapToOutput` if it reports a different output name. Touches should move the cursor and target accurately to within several pixels.

Do not rebuild the full kernel, overwrite the stock `config.txt`, add `tsc2007` to `/etc/modules`, or use the repository's Xorg calibration files: none is needed for this Wayland/labwc setup.
