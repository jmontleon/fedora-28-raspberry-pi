# Fedora 28 Raspberry Pi 3B+ Setup Instructions

These instructions relate primarily to getting the official Raspberry Pi Touchscreen working and could probably be followed for most recent Pi versions. The biggest change would be to which device tree file you'd modify.

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

### Set up dkms for the touchpad driver

Run these commands as root to create the directory, dkms.conf, Makefile, driver source file.
```
mkdir -p /usr/src/rpi-ft5406-1.0
```

```
cat << EOF >> /usr/src/rpi-ft5406-1.0/dkms.conf
dkms driver
PACKAGE_NAME="rpi-ft5406"
PACKAGE_VERSION="1.0"
BUILT_MODULE_NAME[0]="rpi-ft5406"
DEST_MODULE_LOCATION[0]="/kernel/drivers/input/touchscreen/"
AUTOINSTALL="yes"
EOF
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
cat << EOF >> /usr/src/rpi-ft5406-1.0/Makefile
obj-m	+= rpi-ft5406.o
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

```
curl https://raw.githubusercontent.com/raspberrypi/linux/rpi-4.16.y/drivers/input/touchscreen/rpi-ft5406.c -o /usr/src/rpi-ft5406-1.0/rpi-ft5406.c
```

Add, build, and install the modules.
```
dkms add -m rpi-ft5406 -v 1.0
dkms build -m rpi-ft5406 -v 1.0
dkms install -m rpi-ft5406 -v 1.0
dkms add -m rpi_backlight -v 1.0
dkms build -m rpi_backlight -v 1.0
dkms install -m rpi_backlight -v 1.0
```

### Update Device Tree
The devicetree file needs to be updated to enable the touchpad.

```
cd linux-stable
wget 'https://src.fedoraproject.org/cgit/rpms/kernel.git/plain/bcm2837-rpi-initial-3plus-support.patch?h=f28&id=930c3373a22804fbf2764b78bc89d8ccf8e47961' -O bcm2837-rpi-initial-3plus-support.patch
patch -p1 < bcm2837-rpi-initial-3plus-support.patch
```

```
cat << EOF >> rpi-ft5406-backlight.patch
--- arch/arm/boot/dts/bcm2837-rpi-3-b-plus.dts.orig     2018-06-08 22:09:50.109792061 -0400
+++ arch/arm/boot/dts/bcm2837-rpi-3-b-plus.dts  2018-06-08 22:10:38.139576894 -0400
@@ -33,6 +33,18 @@
                compatible = "mmc-pwrseq-simple";
                reset-gpios = <&expgpio 1 GPIO_ACTIVE_HIGH>;
        };
+
+        rpi_ft5406 {
+                compatible = "rpi,rpi-ft5406";
+                firmware = <&firmware>;
+                status = "okay";
+        };
+
+        rpi_backlight {
+			compatible = "raspberrypi,rpi-backlight";
+			firmware = <&firmware>;
+			status = "okay";
+		};
 };

 &firmware {
EOF
patch --ignore-whitespace -p0 < rpi-ft5406-backlight.patch
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
systemctl daemon-reload
systemctl enable copy-dtb
systemctl start copy-dtb
```

### Right Click
I used a script I found on Stack Exchange to get right click working.

It required one unpackaged python module. Install it as your non-root user.

```
pip install --user PyMouse
```

The script is at https://codereview.stackexchange.com/questions/148696/touchscreen-right-click.
I had to change the event10 to event0 in my case. And 'ELAN Touchscreen' to 'FT5406 memory based driver'. The event handler will depend on how many input devices you have and in what order they are enumerated. When I had a USB keyboard plugged in it would appear as event2. You can look at /proc/bus/input/devices on the Handlers line to see which corresponds to your FT5406 memory based driver. It would probably be pretty easy to modify the script to lookup the value with something like:

```
cat /proc/bus/input/devices | grep "FT5406 memory based driver" -A4 | grep -oE event.?[0-9]\{1,1}
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

### Firefox Bug
Firefox has a bug where long pressing to right click disables left click. This is fixed in Firefox 61.0. As of this time builds are available on https://koji.fedoraproject.org to download. For example https://koji.fedoraproject.org/koji/buildinfo?buildID=1097146.

https://bugzilla.mozilla.org/show_bug.cgi?id=1321069
