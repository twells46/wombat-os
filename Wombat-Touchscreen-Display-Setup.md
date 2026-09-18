# Wombat Touchscreen and Display Setup on Raspberry Pi OS

This document explains how to enable and calibrate the Wombat's TSC2007 resistive touchscreen and configure its HDMI display on a current Raspberry Pi OS installation using the KMS graphics stack and the labwc Wayland compositor.

It also explains the actual purpose of the kernel-related steps in the original `Wombat-OS-Build.md` instructions and identifies which of those steps are no longer necessary.

## Final working configuration

The working system has:

- The stock Raspberry Pi OS kernel and stock `tsc2007` kernel module.
- A separately compiled custom `tsc2007.dtbo` device-tree overlay.
- The TSC2007 at I²C address `0x48` with its pen interrupt on GPIO 25.
- An 800×480 HDMI mode selected by labwc through `wlr-randr`.
- The touchscreen mapped to `HDMI-A-1` by labwc.
- A six-value affine calibration matrix derived from measured raw corner coordinates.
- Touch-to-mouse emulation enabled so touches move the visible cursor.

The final labwc calibration matrix is:

```text
1.089352 -0.003025 -0.047122 0.006822 1.138903 -0.085055
```

## What the original kernel instructions were trying to accomplish

The kernel section in `wombat-os.wiki/Wombat-OS-Build.md` performed four logically separate jobs:

1. Enable the `CONFIG_TOUCHSCREEN_TSC2007` kernel option as a module.
2. Add KIPR's custom TSC2007 device-tree overlay to the Raspberry Pi kernel overlay build.
3. Install the resulting kernel, modules, board device trees, and overlays onto the SD card.
4. Replace `config.txt` with settings that enabled I²C, loaded the overlay, and selected a custom HDMI mode.

The only Wombat-specific kernel changes were enabling the TSC2007 module and building the custom overlay. The rest was generic kernel replacement machinery.

Current Raspberry Pi OS kernels already provide the TSC2007 driver as a module. This can be confirmed with:

```bash
sudo modprobe tsc2007
lsmod | grep tsc2007
```

Loading the module is not sufficient by itself. It only makes the driver code available. Linux still needs a device-tree node that describes the physical controller, including its I²C address and interrupt GPIO. Without that node, the driver has no device to probe and no touchscreen input device is created.

Consequently, rebuilding and replacing the whole kernel is unnecessary. Only the custom overlay needs to be compiled and installed.

## Hardware described by the overlay

The source overlay is located at:

```text
configFiles/tsc2007-overlay.dts
```

It describes:

- A Texas Instruments TSC2007-compatible controller.
- I²C bus 1.
- I²C address `0x48`.
- GPIO 25 as the active-low pen interrupt.
- An X-plate resistance of 300 ohms.
- Pressure, polling, and input-fuzz parameters.

The relevant device-tree node is conceptually:

```dts
tsc2007@48 {
    compatible = "ti,tsc2007";
    reg = <0x48>;
    interrupt-parent = <&gpio>;
    interrupts = <25 IRQ_TYPE_LEVEL_LOW>;
    gpios = <&gpio 25 GPIO_ACTIVE_LOW>;
    ti,x-plate-ohms = <300>;
    ti,max_rt = <4096>;
    ti,poll_period = <5>;
    ti,fuzzx = <64>;
    ti,fuzzy = <64>;
    ti,fuzzz = <64>;
};
```

## Building only the device-tree overlay

A DTBO is architecture-independent. It can be built on the Raspberry Pi or on an x86-64 host; a cross-compiler is not required.

The source uses C-preprocessor `#include` directives and symbolic constants, so it must be passed through `cpp` before `dtc`. Calling `dtc` directly on the unprocessed source is insufficient.

### Install the build tools on Raspberry Pi OS

```bash
sudo apt update
sudo apt install device-tree-compiler raspberrypi-kernel-headers
```

If `raspberrypi-kernel-headers` is not available, use the matching Debian-style kernel headers:

```bash
sudo apt install device-tree-compiler linux-headers-$(uname -r)
```

Verify that the device-tree binding headers exist:

```bash
test -r /lib/modules/$(uname -r)/build/include/dt-bindings/gpio/gpio.h \
    && echo "headers found"
```

### Compile the overlay

Run this from the root of the `wombat-os` repository:

```bash
cpp \
  -nostdinc \
  -I /lib/modules/$(uname -r)/build/include \
  -undef \
  -D__DTS__ \
  -x assembler-with-cpp \
  configFiles/tsc2007-overlay.dts |
dtc \
  -@ \
  -I dts \
  -O dtb \
  -o tsc2007.dtbo \
  -
```

The stages have distinct purposes:

- `cpp` resolves `IRQ_TYPE_LEVEL_LOW`, `GPIO_ACTIVE_LOW`, and `BCM2835_FSEL_GPIO_IN`.
- `dtc -@` produces the binary overlay and includes the symbols and fixups needed for dynamic device-tree overlays.

Warnings about overlay fragments having unit addresses without `reg` properties are normal for this style of Raspberry Pi overlay.

If a Raspberry Pi Linux source checkout is already available, its `include` directory can be used instead of installed kernel headers:

```bash
cpp \
  -nostdinc \
  -I /path/to/raspberrypi-linux/include \
  -undef \
  -D__DTS__ \
  -x assembler-with-cpp \
  configFiles/tsc2007-overlay.dts |
dtc -@ -I dts -O dtb -o tsc2007.dtbo -
```

### Inspect the compiled overlay

```bash
file tsc2007.dtbo
dtc -I dtb -O dts -o - tsc2007.dtbo | less
```

The decompiled output should include values corresponding to:

```dts
compatible = "ti,tsc2007";
reg = <0x48>;
interrupts = <0x19 0x08>;
```

`0x19` is GPIO 25 and `0x08` is the level-low interrupt flag.

## Installing and enabling the overlay

On current Raspberry Pi OS releases, install the overlay into `/boot/firmware/overlays`:

```bash
sudo install -m 0644 \
  tsc2007.dtbo \
  /boot/firmware/overlays/tsc2007.dtbo
```

On older releases whose boot partition is mounted directly at `/boot`, use:

```bash
sudo install -m 0644 \
  tsc2007.dtbo \
  /boot/overlays/tsc2007.dtbo
```

Edit the active `config.txt` and add these lines in an applicable section, such as `[all]`:

```ini
dtparam=i2c_arm=on
dtoverlay=tsc2007
```

On a current system, edit:

```bash
sudoedit /boot/firmware/config.txt
```

On an older system, edit `/boot/config.txt` instead.

Do not replace the complete stock `config.txt` with `configFiles/config.txt` from the repository. That file was written for an older Raspberry Pi OS release and contains unrelated and stale platform configuration.

It is also unnecessary to add `tsc2007` to `/etc/modules`. Once the overlay creates the I²C device, the kernel can load the driver automatically from the device's modalias.

Reboot after installing and enabling the overlay:

```bash
sudo reboot
```

## Verifying the kernel and device-tree path

The important result is a bound I²C device and a registered input device, not merely a loaded module.

Check the I²C device and driver binding:

```bash
ls -l /sys/bus/i2c/devices/1-0048
readlink -f /sys/bus/i2c/devices/1-0048/driver
```

A successful binding points to the TSC2007 driver, similar to:

```text
/sys/bus/i2c/devices/1-0048/driver -> .../drivers/tsc2007
```

Check the registered input device and kernel messages:

```bash
grep -A10 -B2 'TSC2007' /proc/bus/input/devices
sudo dmesg | grep -iE 'tsc2007|1-0048|touchscreen'
```

The input device should be named:

```text
TSC2007 Touchscreen
```

Test raw events independently of the desktop:

```bash
sudo apt install evtest
sudo evtest
```

Select the `TSC2007 Touchscreen` event device. Coordinate and touch events in `evtest` confirm that the kernel, I²C, interrupt, overlay, and input-driver portions are working.

## Correct display resolution

The physical HDMI panel resolution is **800×480**.

The repository's tracked `configFiles/config.txt` also specifies 800×480:

```ini
hdmi_group=2
hdmi_mode=87
hdmi_cvt 800 480 60 6 0 0 0
framebuffer_width=800
framebuffer_height=480
```

Aspect code `6` means 15:9, which matches 800:480. An attempted 800×400 mode caused the bottom approximately 80 lines of the display to repeat the top of the image. That repeated region was an LCD timing artifact rather than part of the logical desktop, so neither the mouse nor touchscreen could interact with it.

The root-level `displayconf.txt` present during this investigation specified 800×400, but it was an untracked local file rather than part of the repository history. Its value was not authoritative and proved incorrect for the physical panel.

The panel does not provide useful EDID identification. `wlr-randr` initially reported null make/model information and offered fallback modes such as 1024×768 and 800×600, but not 800×480.

With the full KMS graphics stack, the legacy `hdmi_group`, `hdmi_mode`, and `hdmi_cvt` firmware settings should not be relied upon for Wayland output selection. Labwc can request the correct custom mode through `wlr-randr`:

```bash
wlr-randr \
  --output HDMI-A-1 \
  --custom-mode 800x480@60Hz
```

The mode can first be tested without applying it:

```bash
wlr-randr --dryrun \
  --output HDMI-A-1 \
  --custom-mode 800x480@60Hz
```

If a test mode ever makes the local display unusable, restore a known working mode over SSH:

```bash
wlr-randr --output HDMI-A-1 --mode 800x600
```

### Persist the display mode in labwc

Create a user labwc autostart file without discarding the system defaults:

```bash
mkdir -p ~/.config/labwc

if [ ! -f ~/.config/labwc/autostart ]; then
    cp /etc/xdg/labwc/autostart ~/.config/labwc/autostart
fi
```

Append:

```bash
wlr-randr --output HDMI-A-1 --custom-mode 800x480@60Hz
```

to:

```text
~/.config/labwc/autostart
```

## Wayland versus the old Xorg calibration

The repository contains Xorg configuration such as:

```text
configFiles/99-calibration.conf
configFiles/40-tsc2007.conf
configFiles/screen_settings/Default/99-calibration.conf
```

Those files configure the Xorg `evdev` driver. They do not configure a native Wayland session using labwc and libinput. Installing `xinput-calibrator` or copying files into `/etc/X11/xorg.conf.d` therefore does not calibrate this Wayland setup.

Labwc supports:

- Mapping a touch device to a named output.
- Translating touch events into mouse events.
- Applying a six-value libinput affine calibration matrix.

## Measuring and calculating the touchscreen calibration

Raw corner coordinates were measured with `evtest`. The refined measurements were:

| Physical point | Raw X | Raw Y |
|---|---:|---:|
| Top-left | 171 | 307 |
| Top-right | 3944 | 280 |
| Bottom-left | 195 | 3898 |
| Bottom-right | 3940 | 3880 |

The bottom Y samples varied approximately from 3860 to 3898. This is expected to impose a several-pixel precision limit on a resistive touchscreen.

The axes were neither swapped nor inverted. They did, however, have slightly different usable minima and maxima, plus a small amount of cross-axis skew. Fitting all four points to an affine transformation produced:

```text
1.089352 -0.003025 -0.047122 0.006822 1.138903 -0.085055
```

For normalized raw coordinates `x` and `y`, libinput applies the matrix as:

```text
x' = 1.089352*x - 0.003025*y - 0.047122
y' = 0.006822*x + 1.138903*y - 0.085055
```

The small off-diagonal terms correct the observed skew across the panel.

## Final labwc touchscreen configuration

The final `~/.config/labwc/rc.xml` is:

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

Reload labwc after changing the file:

```bash
labwc --reconfigure
```

If that command does not trigger a visible reload, send the compositor a hangup signal:

```bash
kill -HUP "$(pgrep -xo labwc)"
```

### Mouse-emulation behavior

With:

```xml
mouseEmulation="yes"
```

touches move the visible cursor and are translated into mouse events.

With:

```xml
mouseEmulation="no"
```

labwc uses native touchscreen semantics. In that mode, the mouse cursor disappearing or remaining elsewhere during a touch is normal; the touch should still activate the location under the finger.

## Verifying the Wayland configuration

Confirm the compositor, output, and input-device classification with:

```bash
pgrep -a -x labwc
wlr-randr
sudo libinput list-devices |
  sed -n '/TSC2007 Touchscreen/,/^$/p'
```

Expected properties include:

```text
Capabilities: touch
```

`libinput list-devices` may continue to display `Calibration: identity matrix`. That command creates its own libinput context and reports static/default device configuration; it does not inspect the calibration matrix held privately by the running labwc compositor. Verify the labwc matrix through actual cursor or touch placement instead.

## Expected accuracy and noise

The overlay configures `ti,fuzzx`, `ti,fuzzy`, and `ti,fuzzz` as 64 raw units. The observed bottom-edge Y variation was about 38 raw units. At 800×480, that variation corresponds to roughly five display pixels after calibration.

Perfect pixel-level tracking is therefore not a realistic goal for this resistive touchscreen. The final configuration should provide consistent targeting within several pixels across the display.

## Original instructions that should not be repeated

For this touchscreen/display task, avoid the following portions of the old build procedure:

- Do not rebuild and replace the complete kernel when the stock kernel already supplies `tsc2007.ko`.
- Do not replace the kernel overlay Makefile merely to compile one overlay.
- Do not copy all kernel modules and board DTBs to the SD card.
- Do not overwrite the complete stock `config.txt` with the repository's older copy.
- Do not add `tsc2007` manually to `/etc/modules`; device-tree-based module autoloading is sufficient.
- Do not install Xorg calibration files expecting them to affect labwc/Wayland.
- Do not use 800×400 as the HDMI mode. The physical panel mode is 800×480.
- Do not interpret a successful `modprobe` or `lsmod` result as proof that the hardware has bound to the driver.

The excessive `sudo`, `chmod 777`, `scp` for local copies, and whole-kernel replacement in the original wiki are implementation artifacts, not requirements of the TSC2007 hardware.

## Troubleshooting checklist

### Module loads, but there is no touchscreen input device

Check for the overlay and its boot configuration:

```bash
ls -l /boot/firmware/overlays/tsc2007.dtbo
grep -E '^(dtparam=i2c_arm|dtoverlay=tsc2007)' \
  /boot/firmware/config.txt
```

Then check whether `1-0048` exists and has a driver symlink. If it does not, the overlay was not applied, the wrong boot partition was edited, or the I²C device did not respond.

### `evtest` works, but the desktop does not respond correctly

The kernel path is working. Check:

- That labwc identifies the device as having the `touch` capability.
- That `mapToOutput` names the actual output shown by `wlr-randr`.
- That the matrix is inside labwc's active `rc.xml`.
- That labwc was reconfigured or restarted after editing the file.
- That no Xorg calibration file is being mistaken for Wayland configuration.

### Cursor disappears when touching

That is normal for native Wayland touch semantics. Set `mouseEmulation="yes"` in the labwc `<touch>` element if a visible mouse cursor is desired.

### Display defaults to 1024×768

The panel does not advertise a useful 800×480 EDID mode. Apply and persist:

```bash
wlr-randr --output HDMI-A-1 --custom-mode 800x480@60Hz
```

### Bottom of the display repeats the top

The active timing is wrong. This occurred with 800×400 because the physical display expects 800×480. Restore 800×480.

## References

- [Raspberry Pi `config.txt` documentation](https://www.raspberrypi.com/documentation/computers/config_txt.html)
- [Raspberry Pi legacy video-option documentation](https://www.raspberrypi.com/documentation/computers/legacy_config_txt.html)
- [Raspberry Pi kernel TSC2007 device-tree binding](https://github.com/raspberrypi/linux/blob/rpi-6.18.y/Documentation/devicetree/bindings/input/touchscreen/ti%2Ctsc2007.yaml)
- [Raspberry Pi kernel `bcm2711_defconfig`](https://github.com/raspberrypi/linux/blob/rpi-6.18.y/arch/arm64/configs/bcm2711_defconfig)
- [Linux TSC2007 driver](https://github.com/torvalds/linux/blob/master/drivers/input/touchscreen/tsc2007_core.c)
- [labwc configuration documentation](https://github.com/raspberrypi-ui/labwc-latest/blob/master/docs/labwc-config.5.scd)
- [libinput static device configuration](https://wayland.freedesktop.org/libinput/doc/latest/device-configuration-via-udev.html)
- [`wlr-randr` manual](https://manpages.debian.org/testing/wlr-randr/wlr-randr.1.en.html)
