# Hacking platform drivers into Intel Edison

Unlike most of the development boards, Intel Edison uses binary, proprietary blob for delivering tables of devices connected to I2C and SPI buses. They're loaded via Simple Firmware Interface (SFI) during kernel startup  in Intel MID SFI initialization code (located in intel_mid_sfi.c) and compared to board data structure (board.c). 

"To summarise, in the need of add new devices the SFI Tables is not an option because there isn't any available tool so far to edit or modify such tables and seems like IFWI source is not going to be available to user.  So the best option is to do all the hacks in kernel and add the devices there and remove possible conflicting devices from board.c file." , quoting [icpda](https://communities.intel.com/message/278282#278282), user of an official Intel support forum.

So, we need to hack a kernel? It's much easier than hacking ourselves back in time, so let's do it! I'll add TSC2007, a 4-wire resistive touch panel driver connected over I2C, based on [jbayalas](https://communities.intel.com/message/312395#312395) post in the same thread.

## Adding platform data into device libs

To begin with, we'll create appropriate entries in `linux/arch/x86/platform/intel-mid/device_libs`.

```c
/*
 * platform_tsc2007.h: tsc2007 platform data header file
 *
 * (C) Copyright 2015 Virtus Lab Ltd.
 * Author: Jakub Kramarz <jkramarz@virtuslab.com>
 *
 * This program is free software; you can redistribute it and/or
 * modify it under the terms of the GNU General Public License
 * as published by the Free Software Foundation; version 2
 * of the License.
 */
#ifndef _PLATFORM_TSC2007_H_
#define _PLATFORM_TSC2007_H_

#include <linux/sfi.h>
#include <asm/intel-mid.h>
#include <linux/i2c/tsc2007.h>

extern void __init *tsc2007dl_platform_data(void *info) __attribute__((weak)); // <-- This function will be called later from board.c to get platform data for the device when initializing device

static struct tsc2007_platform_data tsc2007dl_pdata = {
	// here goes platform data you want to provide for your device, see patch at the bottom
};

static struct sfi_device_table_entry tsc2007dl_table_entry = { // <-- This structure is an entry of our very own SFI device table, which will be loaded later
	.type = SFI_DEV_TYPE_I2C,
	.host_num = 6, // <-- The Most Popular I2C Bus on Intel Edison
	.addr = 0x48, // <-- Address of TSC2007 with A0, A1 tied to ground
	.irq = 165, // <-- this value won't be actually used by this driver, it's u8 so won't fit INTEL_MID_IRQ_OFFSET
	.max_freq = 100000,
	.name = "tsc2007", // <-- name of kernel module for handling this device
};

static struct devs_id tsc2007dl_devs_id = { // <-- Just a SFI boilerplate, keep the values consistent over all these places.
	.type = SFI_DEV_TYPE_I2C,
	.delay = 0,
	.get_platform_data = &tsc2007dl_platform_data,
	.device_handler = NULL,
};

#endif
```
The `platform_tsc2007.c` will be included in the patch at the bottom of this post.

OK, now we'll need to add it into `struct devs_id device_ids[]` in `board.c`:
```c
#if defined(CONFIG_TOUCHSCREEN_TSC2007)
        {"tsc2007", SFI_DEV_TYPE_I2C, 1, &tsc2007dl_platform_data, NULL}, // but don't ask me what is that magic number in the middle argument
#endif
```

##Loading our very own SFI entries

Let's take a look at `static int sfi_parse_devs(struct sfi_table_header *table)` in `intel_mid_sfi.c`:
```c
(...)
                if (dev->device_handler) {
                        dev->device_handler(pentry, dev);
                } else {
                        switch (pentry->type) {
                        case SFI_DEV_TYPE_IPC:
                                sfi_handle_ipc_dev(pentry, dev);
                                break;
                        case SFI_DEV_TYPE_SPI:
                                sfi_handle_spi_dev(pentry, dev);
                                break;
                        case SFI_DEV_TYPE_I2C:
                                sfi_handle_i2c_dev(pentry, dev);
                                break;
                        case SFI_DEV_TYPE_SD:
                                sfi_handle_sd_dev(pentry, dev);
                                break;
                        case SFI_DEV_TYPE_HSI:
                        case SFI_DEV_TYPE_UART:
                        default:
                                break;
                        }
                }

(...)
// where dev is devs_id entry with corresponding name
```
We can't use `sfi_parse_devs` without generating own SFI OEMB table, so let's just "handle" our entry separately by calling `sfi_handle_i2c_dev` directly in `int intel_mid_platform_init(void)`:
```c
static int __init intel_mid_platform_init(void)
{
        /* Get SFI OEMB Layout */
        sfi_table_parse(SFI_SIG_OEMB, NULL, NULL, sfi_parse_oemb);
        sfi_table_parse(SFI_SIG_GPIO, NULL, NULL, sfi_parse_gpio);
        sfi_table_parse(SFI_SIG_DEVS, NULL, NULL, sfi_parse_devs);

        #if defined(CONFIG_TOUCHSCREEN_TSC2007)
        /* Hack the platform! */
        sfi_handle_i2c_dev(&tsc2007dl_table_entry, &tsc2007dl_devs_id);
        #endif
        
        return 0;
}
```

## Final touches
Now add newly created files to Makefiles, add missing includes (see provided patch) and create a patch to include into Yocto build.
```sh
git add .
git commit
git format-patch -1 
mv *.patch ~/ww05-15/edison-src/device-software/meta-edison/recipes-kernel/linux/files/
#then, list the patch in linux-yocto_3.10.bbappend
```

## Final patch for TSC2007, adding required modifications in driver itself
```patch
From 9ed714be2c5a55f9f949e7dcc0f954e2f54aa838 Mon Sep 17 00:00:00 2001
From: Jakub Kramarz <jkramarz@virtuslab.com>
Date: Mon, 28 Sep 2015 18:38:34 +0200
Subject: [PATCH] Added TSC2007 support

---
 arch/x86/platform/intel-mid/board.c                |  8 ++-
 arch/x86/platform/intel-mid/device_libs/Makefile   |  1 +
 .../intel-mid/device_libs/platform_tsc2007.c       | 59 ++++++++++++++++++++++
 .../intel-mid/device_libs/platform_tsc2007.h       | 50 ++++++++++++++++++
 arch/x86/platform/intel-mid/intel_mid_sfi.c        |  9 ++++
 drivers/input/touchscreen/tsc2007.c                | 15 ++++--
 include/linux/i2c/tsc2007.h                        |  6 ++-
 7 files changed, 139 insertions(+), 8 deletions(-)
 create mode 100644 arch/x86/platform/intel-mid/device_libs/platform_tsc2007.c
 create mode 100644 arch/x86/platform/intel-mid/device_libs/platform_tsc2007.h

diff --git a/arch/x86/platform/intel-mid/board.c b/arch/x86/platform/intel-mid/board.c
index eb9d098..85e7824 100644
--- a/arch/x86/platform/intel-mid/board.c
+++ b/arch/x86/platform/intel-mid/board.c
@@ -75,6 +75,10 @@
 #include "device_libs/platform_bq24261.h"
 #include "device_libs/platform_pcal9555a.h"
 
+#if defined(CONFIG_TOUCHSCREEN_TSC2007)
+	#include "device_libs/platform_tsc2007.h"
+#endif
+
 /*
  * SPI devices
  */
@@ -105,7 +109,9 @@ struct devs_id __initconst device_ids[] = {
 	{"pcal9555a-2", SFI_DEV_TYPE_I2C, 1, &pcal9555a_platform_data, NULL},
 	{"pcal9555a-3", SFI_DEV_TYPE_I2C, 1, &pcal9555a_platform_data, NULL},
 	{"pcal9555a-4", SFI_DEV_TYPE_I2C, 1, &pcal9555a_platform_data, NULL},
-
+	#if defined(CONFIG_TOUCHSCREEN_TSC2007)
+	{"tsc2007", SFI_DEV_TYPE_I2C, 1, &tsc2007dl_platform_data, NULL},
+	#endif
 	/* SPI devices */
 	{"spidev", SFI_DEV_TYPE_SPI, 0, &spidev_platform_data, NULL},
 	{"ads7955", SFI_DEV_TYPE_SPI, 0, &ads7955_platform_data, NULL},
diff --git a/arch/x86/platform/intel-mid/device_libs/Makefile b/arch/x86/platform/intel-mid/device_libs/Makefile
index 750ac8f..0afed25 100644
--- a/arch/x86/platform/intel-mid/device_libs/Makefile
+++ b/arch/x86/platform/intel-mid/device_libs/Makefile
@@ -10,6 +10,7 @@ obj-y += platform_msic_audio.o
 obj-y += platform_msic_gpio.o
 obj-y += platform_msic_ocd.o
 obj-y += platform_tc35876x.o
+obj-$(subst m,y,$(CONFIG_TOUCHSCREEN_TSC2007)) += platform_tsc2007.o
 obj-y += pci/
 obj-$(subst m,y,$(CONFIG_BATTERY_INTEL_MDF)) += platform_msic_battery.o
 obj-$(subst m,y,$(CONFIG_INTEL_MID_POWER_BUTTON)) += platform_msic_power_btn.o
diff --git a/arch/x86/platform/intel-mid/device_libs/platform_tsc2007.c b/arch/x86/platform/intel-mid/device_libs/platform_tsc2007.c
new file mode 100644
index 0000000..ef01dd5
--- /dev/null
+++ b/arch/x86/platform/intel-mid/device_libs/platform_tsc2007.c
@@ -0,0 +1,59 @@
+/*
+ * platform_tsc2007.c:  tsc2007 platform data initilization file
+ *
+ * (C) Copyright 2015 Virtus Lab Ltd.
+ * Author: Jakub Kramarz <jkramarz@virtuslab.com>
+ *
+ * This program is free software; you can redistribute it and/or
+ * modify it under the terms of the GNU General Public License
+ * as published by the Free Software Foundation; version 2
+ * of the License.
+ */
+
+#include <linux/i2c.h>
+#include <linux/gpio.h>
+#include <asm/intel-mid.h>
+#include "platform_tsc2007.h"
+
+int init_platform_hw(struct i2c_client *client)
+{
+	int err;
+	struct tsc2007_platform_data *pdata = (struct tsc2007_platform_data *) client->dev.platform_data;
+        err = devm_gpio_request_one(&client->dev, pdata->gpio, GPIOF_DIR_IN, "tsc2007 irq");
+	if (err) {
+                dev_err(&client->dev, "failed to request GPIO IRQ pin: %d\n", err);
+        }
+	return err;
+}
+
+void exit_platform_hw(struct i2c_client *client)
+{
+	int err;
+	struct tsc2007_platform_data *pdata = (struct tsc2007_platform_data *) client->dev.platform_data;
+        devm_gpio_free(&client->dev, pdata->gpio);
+}
+
+int tsc2007dl_get_pendown_state(void)
+{
+	int val;
+	val = gpio_get_value(tsc2007dl_pdata.gpio);
+	//printk(KERN_DEBUG "tsc2007dl_get_pendown_state gpio: %d, val: %d", tsc2007dl_pdata.gpio, val);
+	return (val == 0) ? 1 : 0;
+}
+
+void tsc2007dl_clear_penirq(void)
+{
+	return;
+}
+
+void __init* tsc2007dl_platform_data(void *info)
+{
+	struct i2c_board_info *i2c_info = info;
+		
+	i2c_info->irq = tsc2007dl_pdata.gpio + INTEL_MID_IRQ_OFFSET;
+        
+	tsc2007dl_pdata.get_pendown_state = &tsc2007dl_get_pendown_state;
+	tsc2007dl_pdata.clear_penirq = &tsc2007dl_clear_penirq;
+
+	return &tsc2007dl_pdata;
+}
diff --git a/arch/x86/platform/intel-mid/device_libs/platform_tsc2007.h b/arch/x86/platform/intel-mid/device_libs/platform_tsc2007.h
new file mode 100644
index 0000000..4ce3ea3
--- /dev/null
+++ b/arch/x86/platform/intel-mid/device_libs/platform_tsc2007.h
@@ -0,0 +1,50 @@
+/*
+ * platform_tsc2007.h: tsc2007 platform data header file
+ *
+ * (C) Copyright 2015 Virtus Lab Ltd.
+ * Author: Jakub Kramarz <jkramarz@virtuslab.com>
+ *
+ * This program is free software; you can redistribute it and/or
+ * modify it under the terms of the GNU General Public License
+ * as published by the Free Software Foundation; version 2
+ * of the License.
+ */
+#ifndef _PLATFORM_TSC2007_H_
+#define _PLATFORM_TSC2007_H_
+
+#include <linux/sfi.h>
+#include <asm/intel-mid.h>
+#include <linux/i2c/tsc2007.h>
+
+extern void __init *tsc2007dl_platform_data(void *info) __attribute__((weak));
+
+static struct tsc2007_platform_data tsc2007dl_pdata = {
+	.model = 2007,
+	.x_plate_ohms = 308,
+	.max_rt = 0,
+	.poll_period = 10,
+	.get_pendown_state = NULL,
+	.clear_penirq = NULL,
+	.fuzzx = 10,
+	.fuzzy = 10,
+	.fuzzz = 10,
+	.gpio = 165,  //GP165, J18 - pin 2
+};
+
+static struct sfi_device_table_entry tsc2007dl_table_entry = {
+	.type = SFI_DEV_TYPE_I2C,
+	.host_num = 6,
+	.addr = 0x48,
+	.irq = 165,
+	.max_freq = 100000,
+	.name = "tsc2007",
+};
+
+static struct devs_id tsc2007dl_devs_id = {
+	.type = SFI_DEV_TYPE_I2C,
+	.delay = 0,
+	.get_platform_data = &tsc2007dl_platform_data,
+	.device_handler = NULL,
+};
+
+#endif
diff --git a/arch/x86/platform/intel-mid/intel_mid_sfi.c b/arch/x86/platform/intel-mid/intel_mid_sfi.c
index 41f98c4..26e079b 100644
--- a/arch/x86/platform/intel-mid/intel_mid_sfi.c
+++ b/arch/x86/platform/intel-mid/intel_mid_sfi.c
@@ -44,6 +44,10 @@
 #include <asm/reboot.h>
 #include "intel_mid_weak_decls.h"
 
+#if defined(CONFIG_TOUCHSCREEN_TSC2007)
+#include "device_libs/platform_tsc2007.h"
+#endif
+
 #define	SFI_SIG_OEM0	"OEM0"
 #define MAX_IPCDEVS	24
 #define MAX_SCU_SPI	24
@@ -583,6 +587,11 @@ static int __init intel_mid_platform_init(void)
 	sfi_table_parse(SFI_SIG_GPIO, NULL, NULL, sfi_parse_gpio);
 	sfi_table_parse(SFI_SIG_DEVS, NULL, NULL, sfi_parse_devs);
 
+	#if defined(CONFIG_TOUCHSCREEN_TSC2007)
+	/* Hack the platform data! */
+	sfi_handle_i2c_dev(&tsc2007dl_table_entry, &tsc2007dl_devs_id);
+	#endif
+	
 	return 0;
 }
 arch_initcall(intel_mid_platform_init);
diff --git a/drivers/input/touchscreen/tsc2007.c b/drivers/input/touchscreen/tsc2007.c
index 0b67ba4..4ac0c68 100644
--- a/drivers/input/touchscreen/tsc2007.c
+++ b/drivers/input/touchscreen/tsc2007.c
@@ -26,6 +26,7 @@
 #include <linux/interrupt.h>
 #include <linux/i2c.h>
 #include <linux/i2c/tsc2007.h>
+#include <linux/gpio.h>
 
 #define TSC2007_MEASURE_TEMP0		(0x0 << 4)
 #define TSC2007_MEASURE_AUX		(0x2 << 4)
@@ -298,7 +299,7 @@ static int tsc2007_probe(struct i2c_client *client,
 	}
 
 	ts->client = client;
-	ts->irq = client->irq;
+	ts->irq = gpio_to_irq(pdata->gpio);
 	ts->input = input_dev;
 	init_waitqueue_head(&ts->wait);
 
@@ -337,10 +338,14 @@ static int tsc2007_probe(struct i2c_client *client,
 			pdata->fuzzz, 0);
 
 	if (pdata->init_platform_hw)
-		pdata->init_platform_hw();
+	{
+		err = pdata->init_platform_hw(client);
+		if (err)
+        	        goto err_free_irq;
+	}
 
 	err = request_threaded_irq(ts->irq, tsc2007_hard_irq, tsc2007_soft_irq,
-				   IRQF_ONESHOT, client->dev.driver->name, ts);
+				   IRQF_ONESHOT | IRQF_TRIGGER_RISING | IRQF_TRIGGER_FALLING, client->dev.driver->name, ts);
 	if (err < 0) {
 		dev_err(&client->dev, "irq %d busy?\n", ts->irq);
 		goto err_free_mem;
@@ -359,7 +364,7 @@ static int tsc2007_probe(struct i2c_client *client,
  err_free_irq:
 	free_irq(ts->irq, ts);
 	if (pdata->exit_platform_hw)
-		pdata->exit_platform_hw();
+		pdata->exit_platform_hw(client);
  err_free_mem:
 	input_free_device(input_dev);
 	kfree(ts);
@@ -374,7 +379,7 @@ static int tsc2007_remove(struct i2c_client *client)
 	free_irq(ts->irq, ts);
 
 	if (pdata->exit_platform_hw)
-		pdata->exit_platform_hw();
+		pdata->exit_platform_hw(client);
 
 	input_unregister_device(ts->input);
 	kfree(ts);
diff --git a/include/linux/i2c/tsc2007.h b/include/linux/i2c/tsc2007.h
index 506a9f7..82fd7db 100644
--- a/include/linux/i2c/tsc2007.h
+++ b/include/linux/i2c/tsc2007.h
@@ -14,11 +14,13 @@ struct tsc2007_platform_data {
 	int	fuzzy;
 	int	fuzzz;
 
+	int	gpio;
+
 	int	(*get_pendown_state)(void);
 	void	(*clear_penirq)(void);		/* If needed, clear 2nd level
 						   interrupt source */
-	int	(*init_platform_hw)(void);
-	void	(*exit_platform_hw)(void);
+	int	(*init_platform_hw)(struct i2c_client *client);
+	void	(*exit_platform_hw)(struct i2c_client *client);
 };
 
 #endif
-- 
1.9.1
```
To enable these changes, add `CONFIG_TOUCHSCREEN_TSC2007=y` at the bottom of `defconfig`.