# Fedora 30 Raspberry Pi 3B+ Setup Instructions

These instructions relate primarily to getting the official Raspberry Pi Touchscreen working and could probably be followed for most recent Pi versions.

## Install Instructions
Most instructions are available at https://fedoraproject.org/wiki/Architectures/ARM/Raspberry_Pi

## armv7hl vs aarch64
- With Fedora 30 I no longer have inverted colors when using fbdev like I did with Fedora 28.
- How you enable overlays is a bit different
  - For armvh7hl remove the ftddir line in /boot/extlinux/extlinux.conf
  - For aarch64 unlink /boot/dtb
- Regardless of architecture the correct touchscreen overlay is not included
- I chose to go with aarch64.
  - If you go with armv7hl the instructions will be similar, but dtb locations may vary a bit.

## Official Raspberry Pi Touch Screen

### Blacklist VC4
The vc4 driver will currently blank the official 7" touch screen until we get the dtb updated. For the time being it needs to be blacklisted if you're using the touch display.

You will probably need to do this with an HDMI monitor and then attach the LCD. While at it. Be aware this blanking issue affects some HDMI monitors as well. If you find mid-boot the screen blanks out you can use this same workaround but the fix will likely not be relevant.

Below we'll patch the dts(i) files to fix this and unblacklist it.

```
cat << EOF >> /etc/modprobe.d/blacklist-vc4.conf
blacklist vc4
EOF
```

Instead of running `dracut -f` to rebuild the initramfs, update the kernel to save some waiting later. Once done shutdown, and attach the LCD. You should be able to get to a login prompt without losing video.

```
dnf -y update kernel
```

### Install and download everything we'll need to get touch and backlight control working.

```
dnf -y install dkms kernel-devel make bc bison bzip2 elfutils elfutils-devel flex libkcapi-hmaccalc m4 net-tools openssl-devel patch perl-devel perl-generators pesign python3-xlib python3-evdev git rpm-build

git clone https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux-stable.git
```

### Set up dkms for the touchpad backlight driver

Run these commands as root to create the directory, dkms.conf, Makefile, driver source file.
```
mkdir -p /usr/src/rpi_backlight-1.0
```

```
cat << EOF >> /usr/src/rpi_backlight-1.0/dkms.conf
dkms driver
PACKAGE_NAME="rpi_backlight"
PACKAGE_VERSION="1.0"
BUILT_MODULE_NAME[0]="rpi_backlight"
DEST_MODULE_LOCATION[0]="/kernel/drivers/video/backlight"
AUTOINSTALL="yes"
EOF
```

```
cat << EOF >> /usr/src/rpi_backlight-1.0/Makefile
obj-m += rpi_backlight.o
EOF
```

```
curl https://raw.githubusercontent.com/raspberrypi/linux/rpi-4.16.y/drivers/video/backlight/rpi_backlight.c -o /usr/src/rpi_backlight-1.0/rpi_backlight.c
```

Add, build, and install the modules.
```
dkms add -m rpi_backlight -v 1.0
dkms build -m rpi_backlight -v 1.0
dkms install -m rpi_backlight -v 1.0
```

### Update Device Tree
The devicetree file needs to be updated to enable the touchpad.

```
cat << EOF >> rpi-ts.patch
diff --git a/arch/arm/boot/dts/bcm2837-rpi-3-b-plus.dts b/arch/arm/boot/dts/bcm2837-rpi-3-b-plus.dts
index abb2bc7..486df37 100644
--- a/arch/arm/boot/dts/bcm2837-rpi-3-b-plus.dts
+++ b/arch/arm/boot/dts/bcm2837-rpi-3-b-plus.dts
@@ -33,6 +33,12 @@
                compatible = "mmc-pwrseq-simple";
                reset-gpios = <&expgpio 1 GPIO_ACTIVE_LOW>;
        };
+
+       rpi_backlight {
+               compatible = "raspberrypi,rpi-backlight";
+               firmware = <&firmware>;
+               status = "okay";
+       };
 };

 &firmware {
@@ -50,6 +56,11 @@
                                  "";
                status = "okay";
        };
+
+       ts: touchscreen {
+               compatible = "raspberrypi,firmware-ts";
+       };
+
 };

 &gpio {
@@ -157,6 +168,11 @@
        bus-width = <4>;
 };

+&i2c_dsi {
+       gpios = <&gpio 44 0
+                &gpio 45 0>;
+};
+
 /* uart0 communicates with the BT module */
 &uart0 {
        pinctrl-names = "default";
diff --git a/arch/arm/boot/dts/bcm283x.dtsi b/arch/arm/boot/dts/bcm283x.dtsi
index 2def068..cc38287 100644
--- a/arch/arm/boot/dts/bcm283x.dtsi
+++ b/arch/arm/boot/dts/bcm283x.dtsi
@@ -562,7 +562,11 @@
                                             "dsi1_ddr2",
                                             "dsi1_ddr";

-                       status = "disabled";
+                       port {
+                               dsi_out_port: endpoint {
+                                       remote-endpoint = <&panel_dsi_port>;
+                               };
+                       };
                };

                i2c1: i2c@7e804000 {
@@ -634,6 +638,30 @@
                vc4: gpu {
                        compatible = "brcm,bcm2835-vc4";
                };
+
+               i2c_dsi: i2c {
+                       /* We have to use i2c-gpio because the
+                        * firmware is also polling another device
+                        * using the only hardware I2C bus that could
+                        * connect to these pins.
+                        */
+
+                       compatible = "i2c-gpio";
+                       #address-cells = <1>;
+                       #size-cells = <0>;
+                       gpios = <&gpio 28 0
+                                &gpio 29 0>;
+
+                       lcd@45 {
+                               compatible = "raspberrypi,panel_raspberrypi_touchscreen";
+                               reg = <0x45>;
+                               port {
+                                       panel_dsi_port: endpoint {
+                                               remote-endpoint = <&dsi_out_port>;
+                                       };
+                               };
+                       };
+               };
        };

        clocks {
EOF
patch --ignore-whitespace -p1 < rpi-ts.patch
```

Build the dtb
```
yes "" | make oldconfig
make dtbs
cp arch/arm64/boot/dts/broadcom/bcm2837-rpi-3-b-plus.dtb /boot
```

In practice this file does not change often, so generally speaking you can copy it to /boot/dtb for each new kernel installed.

This can be handled automatically with a shutdown service file

```
cat << EOF > /etc/systemd/system/copy-dtb.service
[Unit]
Description=Copy custom DTB into place, necessary when the kernel is updated

[Service]
Type=oneshot
RemainAfterExit=true
ExecStop=/bin/cp -f /boot/bcm2837-rpi-3-b-plus.dtb /boot/dtb/broadcom/bcm2837-rpi-3-b-plus.dtb

[Install]
WantedBy=multi-user.target
EOF
```

```
systemctl daemon-reload
systemctl enable copy-dtb
systemctl start copy-dtb
```

These changes also fix the vc4 driver when using DSI (e.g. 7" Raspberry Pi Touchscreen) so you can enable the driver again if you disabled it.

```
rm /etc/modprobe.d/vc4.conf
dracut -f
```

### Click Fix Script
While the touchscreen works I have been unable to left or right click without a script like this.

Add your user to the input group so it can connect to the input dev
```
usermod -aG input $your-non-root-username
```

This script requires one unpackaged python module. Install it as your non-root user.

```
pip3 install --user PyUserInput
```

```
mkdir -p ~/bin
cat << EOF > ~/bin/clickfix.py
#!/bin/python3

from evdev import InputDevice
import time
from pymouse import PyMouse
from threading import Timer
import subprocess

dev = InputDevice('/dev/input/by-path/platform-soc:firmware:touchscreen-event')
m = PyMouse()
lasttime = time.time()
rlasttime = time.time()
originaltime = lasttime
oldclickx = 0
oldclicky = 0


for event in dev.read_loop():

    if event.type == 3 and event.code == 47 and event.value == 1:
        rclicktime = time.time()
        if (rclicktime - rlasttime) < .5:
            rlasttime = rclicktime
        else:
            print("Two Finger tap.")
            subprocess.check_call(['xinput', '--disable', 'raspberrypi-ts'])
            x2, y2 = m.position()  # Get the pointer coordinates
            m.click(x2, y2, 2)
            subprocess.check_call(['xinput', '--enable', 'raspberrypi-ts'])
            rlasttime = rclicktime

    elif event.type == 1 and event.code == 330 and event.value == 1:
        clicktime = time.time()
        clickx, clicky = m.position()
        if (clicktime - lasttime) < .5 and (abs(clickx - oldclickx) < 20) and (abs(clicky - oldclicky) < 20):
            print("Double click.")
            subprocess.check_call(['xinput', '--disable', 'raspberrypi-ts'])
            x2, y2 = m.position()
            m.click(x2, y2, 1)
            m.click(x2, y2, 1)
            subprocess.check_call(['xinput', '--enable', 'raspberrypi-ts'])
            lasttime = originaltime
        else:
            lasttime = clicktime
        oldclickx = clickx
        oldclicky = clicky
EOF
chmod +x ~/bin/clickfix.py
```

I added the script to my Xfce startup so whenever I log in it runs.

I did the same with a script to detect when the screensaver kicks in to turn the backlight off. Be aware that if the screen locks this will not unblank for you to enter your password, which may make it difficult to unlock and therefore undesirable. If you're just blanking the screen and not locking it it should work fine.

```
#!/usr/bin/perl

my $blanked = 0;
open (IN, "xscreensaver-command -watch |");
while (<IN>) {
    if (m/^(BLANK|LOCK)/) {
        if (!$blanked) {
            system "bash -c 'echo 0 | sudo tee /sys/class/backlight/rpi_backlight/brightness'";
            $blanked = 1;
        }
    } elsif (m/^UNBLANK/) {
        system "bash -c 'echo 255 | sudo tee /sys/class/backlight/rpi_backlight/brightness'";
        $blanked = 0;
    }
}
```
