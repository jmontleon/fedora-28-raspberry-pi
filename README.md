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

### Set up the dkms
This will get you the driver for the touchpad.

```
dnf -y install dkms kernel-devel make bc bison bzip2 elfutils elfutils-devel flex libkcapi-hmaccalc m4 net-tools openssl-devel patch perl-devel perl-generators pesign
```

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
cat << EOF >> /usr/src/rpi-ft5406-1.0/rpi-ft5406.c
/*
 * Driver for memory based ft5406 touchscreen
 *
 * Copyright (C) 2015, 2017 Raspberry Pi
 *
 *
 * This program is free software; you can redistribute it and/or modify
 * it under the terms of the GNU General Public License version 2 as
 * published by the Free Software Foundation.
 */

#include <linux/module.h>
#include <linux/interrupt.h>
#include <linux/input.h>
#include <linux/irq.h>
#include <linux/delay.h>
#include <linux/slab.h>
#include <linux/bitops.h>
#include <linux/input/mt.h>
#include <linux/kthread.h>
#include <linux/platform_device.h>
#include <linux/stddef.h>
#include <linux/io.h>
#include <linux/dma-mapping.h>
#include <soc/bcm2835/raspberrypi-firmware.h>

#define MAXIMUM_SUPPORTED_POINTS 10
#define FTS_TOUCH_DOWN      0
#define FTS_TOUCH_UP        1
#define FTS_TOUCH_CONTACT   2

struct ft5406_regs {
	uint8_t device_mode;
	uint8_t gesture_id;
	uint8_t num_points;
	struct ft5406_touch {
		uint8_t xh;
		uint8_t xl;
		uint8_t yh;
		uint8_t yl;
		uint8_t pressure; /* Not supported */
		uint8_t area;     /* Not supported */
	} point[MAXIMUM_SUPPORTED_POINTS];
};

/* These are defaults if the DT entries are missing */
#define DEFAULT_SCREEN_WIDTH  800
#define DEFAULT_SCREEN_HEIGHT 480

struct ft5406 {
	struct platform_device	*pdev;
	struct input_dev	*input_dev;
	void __iomem		*ts_base;
	dma_addr_t		bus_addr;
	struct task_struct	*thread;

	uint16_t	max_x;
	uint16_t	max_y;
	uint8_t		hflip;
	uint8_t		vflip;
	uint8_t		xyswap;
};

/* Thread to poll for touchscreen events
 *
 * This thread polls the memory based register copy of the ft5406 registers
 * using the number of points register to know whether the copy has been
 * updated (we write 99 to the memory copy, the GPU will write between
 * 0 - 10 points)
 */
#define ID_TO_BIT(a) (1 << a)

static int ft5406_thread(void *arg)
{
	struct ft5406 *ts = (struct ft5406 *) arg;
	struct ft5406_regs regs;
	int known_ids = 0;

	while (!kthread_should_stop()) {
		/* 60fps polling */
		msleep_interruptible(17);
		memcpy_fromio(&regs, ts->ts_base, sizeof(struct ft5406_regs));
		iowrite8(99,
			 ts->ts_base +
			 offsetof(struct ft5406_regs, num_points));

		/*
		 * Do not output if theres no new information (num_points is 99)
		 * or we have no touch points and don't need to release any
		 */
		if (!(regs.num_points == 99 ||
		    (regs.num_points == 0 && known_ids == 0))) {
			int i;
			int modified_ids = 0, released_ids;

			for (i = 0; i < regs.num_points; i++) {
				int x = (((int) regs.point[i].xh & 0xf) << 8) +
					regs.point[i].xl;
				int y = (((int) regs.point[i].yh & 0xf) << 8) +
					regs.point[i].yl;
				int touchid = (regs.point[i].yh >> 4) & 0xf;
				int event_type = (regs.point[i].xh >> 6) & 0x03;

				modified_ids |= ID_TO_BIT(touchid);

				if (event_type == FTS_TOUCH_DOWN ||
				    event_type == FTS_TOUCH_CONTACT) {
					if (ts->hflip)
						x = ts->max_x - 1 - x;

					if (ts->vflip)
						y = ts->max_y - 1 - y;

					if (ts->xyswap)
						swap(x, y);

					if (!((ID_TO_BIT(touchid)) & known_ids))
						dev_dbg(&ts->pdev->dev,
							"x = %d, y = %d, press = %d, touchid = %d\n",
							x, y,
							regs.point[i].pressure,
							touchid);

					input_mt_slot(ts->input_dev, touchid);
					input_mt_report_slot_state(
							ts->input_dev,
							MT_TOOL_FINGER,
							1);

					input_report_abs(ts->input_dev,
							 ABS_MT_POSITION_X, x);
					input_report_abs(ts->input_dev,
							 ABS_MT_POSITION_Y, y);
				}
			}

			released_ids = known_ids & ~modified_ids;
			for (i = 0;
			     released_ids && i < MAXIMUM_SUPPORTED_POINTS;
			     i++) {
				if (released_ids & (1<<i)) {
					dev_dbg(&ts->pdev->dev,
						"Released %d, known = %x, modified = %x\n",
						i, known_ids, modified_ids);
					input_mt_slot(ts->input_dev, i);
					input_mt_report_slot_state(
							ts->input_dev,
							MT_TOOL_FINGER,
							0);
					modified_ids &= ~(ID_TO_BIT(i));
				}
			}
			known_ids = modified_ids;

			input_mt_report_pointer_emulation(ts->input_dev, true);
			input_sync(ts->input_dev);
		}
	}

	return 0;
}

static int ft5406_probe(struct platform_device *pdev)
{
	int err = 0;
	struct device *dev = &pdev->dev;
	struct device_node *np = dev->of_node;
	struct ft5406 *ts;
	struct device_node *fw_node;
	struct rpi_firmware *fw;
	u32 touchbuf;
	u32 val;

	dev_info(dev, "Probing device\n");

	fw_node = of_parse_phandle(np, "firmware", 0);
	if (!fw_node) {
		dev_err(dev, "Missing firmware node\n");
		return -ENOENT;
	}

	fw = rpi_firmware_get(fw_node);
	if (!fw)
		return -EPROBE_DEFER;

	ts = devm_kzalloc(dev, sizeof(struct ft5406), GFP_KERNEL);
	if (!ts) {
		dev_err(dev, "Failed to allocate memory\n");
		return -ENOMEM;
	}

	ts->input_dev = input_allocate_device();
	if (!ts->input_dev) {
		dev_err(dev, "Failed to allocate input device\n");
		return -ENOMEM;
	}

	ts->ts_base = dma_zalloc_coherent(dev, PAGE_SIZE, &ts->bus_addr,
					  GFP_KERNEL);
	if (!ts->ts_base) {
		pr_err("[%s]: failed to dma_alloc_coherent(%ld)\n",
				__func__, PAGE_SIZE);
		err = -ENOMEM;
		goto out;
	}

	touchbuf = (u32)ts->bus_addr;
	err = rpi_firmware_property(fw, RPI_FIRMWARE_FRAMEBUFFER_SET_TOUCHBUF,
				    &touchbuf, sizeof(touchbuf));

	if (err || touchbuf != 0) {
		dev_warn(dev, "Failed to set touchbuf, trying to get err:%x\n",
			 err);
		dma_free_coherent(dev, PAGE_SIZE, ts->ts_base, ts->bus_addr);
		ts->ts_base = 0;
		ts->bus_addr = 0;
	}

	if (!ts->ts_base) {
		dev_warn(dev,
		"set failed, trying get (err:%d touchbuf:%x virt:%p bus:%x)\n",
		err, touchbuf, ts->ts_base, ts->bus_addr);

		err = rpi_firmware_property(
				fw,
				RPI_FIRMWARE_FRAMEBUFFER_GET_TOUCHBUF,
				&touchbuf, sizeof(touchbuf));
		if (err) {
			dev_err(dev, "Failed to get touch buffer\n");
			goto out;
		}

		if (!touchbuf) {
			dev_err(dev, "Touchscreen not detected\n");
			err = -ENODEV;
			goto out;
		}

		dev_dbg(dev, "Got TS buffer 0x%x\n", touchbuf);

		/* mmap the physical memory */
		touchbuf &= ~0xc0000000;
		ts->ts_base = ioremap(touchbuf, sizeof(struct ft5406_regs));
		if (ts->ts_base == NULL) {
			dev_err(dev, "Failed to map physical address\n");
			err = -ENOMEM;
			goto out;
		}
	}
	platform_set_drvdata(pdev, ts);
	ts->pdev = pdev;

	ts->input_dev->name = "FT5406 memory based driver";

	if (of_property_read_u32(np, "touchscreen-size-x", &val) >= 0)
		ts->max_x = val;
	else
		ts->max_x = DEFAULT_SCREEN_WIDTH;

	if (of_property_read_u32(np, "touchscreen-size-y", &val) >= 0)
		ts->max_y = val;
	else
		ts->max_y = DEFAULT_SCREEN_HEIGHT;

	if (of_property_read_u32(np, "touchscreen-inverted-x", &val) >= 0)
		ts->hflip = val;

	if (of_property_read_u32(np, "touchscreen-inverted-y", &val) >= 0)
		ts->vflip = val;

	if (of_property_read_u32(np, "touchscreen-swapped-x-y", &val) >= 0)
		ts->xyswap = val;

	dev_dbg(dev,
		"Touchscreen parameters (%d,%d), hflip=%d, vflip=%d, xyswap=%d",
		ts->max_x, ts->max_y, ts->hflip, ts->vflip, ts->xyswap);

	__set_bit(EV_KEY, ts->input_dev->evbit);
	__set_bit(EV_SYN, ts->input_dev->evbit);
	__set_bit(EV_ABS, ts->input_dev->evbit);

	input_set_abs_params(ts->input_dev, ABS_MT_POSITION_X, 0,
			     ts->xyswap ? ts->max_y : ts->max_x, 0, 0);
	input_set_abs_params(ts->input_dev, ABS_MT_POSITION_Y, 0,
			     ts->xyswap ? ts->max_x : ts->max_y, 0, 0);

	input_mt_init_slots(ts->input_dev,
			    MAXIMUM_SUPPORTED_POINTS, INPUT_MT_DIRECT);

	input_set_drvdata(ts->input_dev, ts);

	err = input_register_device(ts->input_dev);
	if (err) {
		dev_err(dev, "could not register input device, %d\n",
			err);
		goto out;
	}

	/* create thread that polls the touch events */
	ts->thread = kthread_run(ft5406_thread, ts, "ft5406");
	if (ts->thread == NULL) {
		dev_err(dev, "Failed to create kernel thread");
		err = -ENOMEM;
		goto out;
	}

	return 0;

out:
	if (ts->bus_addr) {
		dma_free_coherent(dev, PAGE_SIZE, ts->ts_base, ts->bus_addr);
		ts->bus_addr = 0;
		ts->ts_base = NULL;
	} else if (ts->ts_base) {
		iounmap(ts->ts_base);
		ts->ts_base = NULL;
	}
	if (ts->input_dev) {
		input_unregister_device(ts->input_dev);
		ts->input_dev = NULL;
	}
	return err;
}

static int ft5406_remove(struct platform_device *pdev)
{
	struct device *dev = &pdev->dev;
	struct ft5406 *ts = (struct ft5406 *) platform_get_drvdata(pdev);

	dev_info(dev, "Removing rpi-ft5406\n");

	kthread_stop(ts->thread);

	if (ts->bus_addr)
		dma_free_coherent(dev, PAGE_SIZE, ts->ts_base, ts->bus_addr);
	else if (ts->ts_base)
		iounmap(ts->ts_base);
	if (ts->input_dev)
		input_unregister_device(ts->input_dev);

	return 0;
}

static const struct of_device_id ft5406_match[] = {
	{ .compatible = "rpi,rpi-ft5406", },
	{},
};
MODULE_DEVICE_TABLE(of, ft5406_match);

static struct platform_driver ft5406_driver = {
	.driver = {
		.name   = "rpi-ft5406",
		.owner  = THIS_MODULE,
		.of_match_table = ft5406_match,
	},
	.probe          = ft5406_probe,
	.remove         = ft5406_remove,
};

module_platform_driver(ft5406_driver);

MODULE_AUTHOR("Gordon Hollingworth");
MODULE_DESCRIPTION("Touchscreen driver for memory based FT5406");
MODULE_LICENSE("GPL");
EOF
```

```
dkms add -m rpi-ft5406 -v 1.0
dkms build -m rpi-ft5406 -v 1.0
dkms install -m rpi-ft5406 -v 1.0
```

### Update Device Tree
The devicetree file needs to be updated.

```
dnf -y install git
```

```
git clone https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux-stable.git
cd linux-stable
```

Add this block below wifi_pwrseq in arch/arm/boot/dts/bcm2837-rpi-3-b-plus.dts
```
  rpi_ft5406 {
		compatible = "rpi,rpi-ft5406";
		firmware = <&firmware>;
		status = "okay";
	};
```

```
make oldconfig
make dtbs
```

```
cp
```

In practice this file does not change often, so generally speaking you can copy the dtb file from /boot/dtb-$(previous-kernel-ver) /boot/dtb.
