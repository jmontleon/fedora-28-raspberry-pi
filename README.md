# Fedora 28 Raspberry Pi Setup Instructions

## Install Instrctions
Most instructions are available at https://fedoraproject.org/wiki/Architectures/ARM/Raspberry_Pi

## armv7hl vs aarch64
Right now the biggest factor for me is that that red and blue are reversed on aarch64, which makes for a bad desktop experience.

## Official Touch Screen Support

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

### Set up dkms for the touchpad driver

Install dkms and other packages required for kernel development
```
dnf -y install dkms kernel-devel make bc bison bzip2 elfutils elfutils-devel flex libkcapi-hmaccalc m4 net-tools openssl-devel patch perl-devel perl-generators pesign
```

Create the directory for the driver module, create the dkms.conf, Makefile, and download the driver source.
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
cat << EOF >> /usr/src/rpi-ft5406-1.0/Makefile
obj-m	+= rpi-ft5406.o
EOF
```

```
curl https://raw.githubusercontent.com/raspberrypi/linux/rpi-4.16.y/drivers/input/touchscreen/rpi-ft5406.c -o /usr/src/rpi-ft5406-1.0/rpi-ft5406.c
```

Add, build, and install the module.
```
dkms add -m rpi-ft5406 -v 1.0
dkms build -m rpi-ft5406 -v 1.0
dkms install -m rpi-ft5406 -v 1.0
```

### Update Device Tree
The devicetree file needs to be updated to enable the touchpad.

```
dnf -y install git
```

```
git clone https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux-stable.git
cd linux-stable
wget 'https://src.fedoraproject.org/cgit/rpms/kernel.git/plain/bcm2837-rpi-initial-3plus-support.patch?h=f28&id=930c3373a22804fbf2764b78bc89d8ccf8e47961' -O bcm2837-rpi-initial-3plus-support.patch
patch -p1 < bcm2837-rpi-initial-3plus-support.patch
```

```
cat << EOF >> rpi-ft5406.patch
--- arch/arm/boot/dts/bcm2837-rpi-3-b-plus.dts.orig     2018-06-08 22:03:39.576476545 -0400
+++ arch/arm/boot/dts/bcm2837-rpi-3-b-plus.dts  2018-06-08 22:06:32.715681114 -0400
@@ -33,6 +33,12 @@
                compatible = "mmc-pwrseq-simple";
                reset-gpios = <&expgpio 1 GPIO_ACTIVE_HIGH>;
        };
+
+       rpi_ft5406 {
+               compatible = "rpi,rpi-ft5406";
+               firmware = <&firmware>;
+               status = "okay";
+       };
 };

 &firmware {
EOF

patch -p1 < rpi-ft5406.patch
```

Build the dtb
```
make oldconfig
make dtbs
```

Replace the current dtb 
```
cp
```

In practice this file does not change often, so generally speaking you can copy the dtb file from /boot/dtb-$(previous-kernel-ver) /boot/dtb.
