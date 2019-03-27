# Fedora 29 Raspberry Pi 3B+ Setup Instructions

These instructions relate primarily to getting the official Raspberry Pi Touchscreen working and could probably be followed for most recent Pi versions. The biggest change would be to which device tree file you'd modify. It is important to update to at least a 5.0 kernel to follow these instructions. This is when the raspberrypi-ts driver became available

## Install Instructions
Most instructions are available at https://fedoraproject.org/wiki/Architectures/ARM/Raspberry_Pi

## armv7hl vs aarch64
Right now the biggest factor for me is that that red and blue are reversed on aarch64 when not using the vc4 driver, which makes for a bad desktop experience.

## Official Raspberry Pi Touch Screen

### Blacklist VC4
The vc4 driver will currently blank the official 7" touch screen so it needs to be blacklisted. You will probably need to do this with an HDMI monitor and then attach the LCD. While at it. Be aware this blanking issue affects some HDMI monitors as well.

```
cat << EOF >> /etc/modprobe.d/blacklist-vc4.conf
blacklist vc4
EOF
```

Update the kernel. Instead of running dracut to rebuild the initramfs and then updating the kernel later and sitting through the wait again just do so now. Once done shutdown, and attach the LCD. You should be able to get to a login prompt without losing video.

```
dnf -y update kernel
```

### Install and download everything we'll need to get touch and backlight control working.

```
dnf -y install dkms kernel-devel make bc bison bzip2 elfutils elfutils-devel flex libkcapi-hmaccalc m4 net-tools openssl-devel patch perl-devel perl-generators pesign python-xlib python-evdev git rpm-build

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
index 42bb09044cc7..4f7d553110f9 100644
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

 &hdmi {
EOF
patch --ignore-whitespace -p1 < rpi-ts.patch
```

Build the dtb
```
yes "" | make oldconfig
make dtbs
```

Replace the current dtb
```
cp arch/arm/boot/dts/bcm2837-rpi-3-b-plus.dtb /boot/dtb
```

In practice this file does not change often, so generally speaking you can copy the dtb file from /boot/dtb-$(previous-kernel-ver) /boot/dtb.

This can be handled automatically with a shutdown service file

```
cat << EOF > /etc/systemd/system/copy-dtb.service
[Unit]
Description=Copy custom DTB into place, necessary when the kernel is updated

[Service]
Type=oneshot
RemainAfterExit=true
ExecStop=/bin/cp -f /boot/bcm2837-rpi-3-b-plus.dtb /boot/dtb/bcm2837-rpi-3-b-plus.dtb

[Install]
WantedBy=multi-user.target
EOF
```

```
cp /boot/dtb/bcm2837-rpi-3-b-plus.dtb /boot
systemctl daemon-reload
systemctl enable copy-dtb
systemctl start copy-dtb
```

### Click Fix Script
Since Fedora 28 I have also lost left click. I've modified the old script I used pretty heavily to provide both left and right click.

It required one unpackaged python module. Install it as your non-root user.

```
pip install --user PyMouse
```


```
mkdir -p ~/bin
cat << EOF > ~/bin/clickfix.py
#!/bin/python

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
            print "Two Finger tap."
            subprocess.check_call(['xinput', '--disable', 'raspberrypi-ts'])
            x2, y2 = m.position()  # Get the pointer coordinates
            m.click(x2, y2, 2)
            subprocess.check_call(['xinput', '--enable', 'raspberrypi-ts'])
            rlasttime = rclicktime

    elif event.type == 1 and event.code == 330 and event.value == 1:
        clicktime = time.time()
        clickx, clicky = m.position()
        if (clicktime - lasttime) < .5 and (abs(clickx - oldclickx) < 20) and (abs(clicky - oldclicky) < 20):
            print "Double click."
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
