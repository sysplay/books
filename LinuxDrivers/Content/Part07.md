# Generic Hardware Access in Linux

> This seventh article, which is part of the series on Linux device drivers, talks about accessing hardware in Linux.

Shweta was all jubilant about her character driver achievements, as she entered the Linux device drivers laboratory on the second floor of her college. Why not? Many of her classmates had already read her blog & commented on her expertise. And today was a chance for show-off at an another level. Till now, it was all software. Today’s lab was on accessing hardware in Linux. Students are expected to “learn by experimentation” to access various kinds of hardware in Linux on various architectures over multiple lab sessions here.

As usual, the lab staff are a bit skeptical to let the students directly get onto the hardware, without any background. So to build their background, they have prepared some slide presentations, which can be accessed from [SysPlay’s website](https://sysplay.in/index.php?pagefile=linux_drivers).

## Generic hardware interfacing

As every one settled in the laboratory, lab expert Priti started with the introduction to hardware interfacing in Linux. Skipping the theoretical details, the first interesting slide was about the generic architecture-transparent hardware interfacing. See Figure 11.

![Figure 11](/Images/Part7/figure_11_hardware_mapping.png)

The basic assumption being that the architecture is 32-bit. For others, the memory map would change accordingly. For 32-bit address bus, the address/memory map ranges from 0 `(0x00000000)` to ‘232 – 1′ `(0xFFFFFFFF)`. And an architecture independent layout of this memory map would be as shown in the Figure 11 – memory (RAM) and device regions (registers & memories of devices) mapped in an interleaved fashion. The architecture dependent thing would be what these addresses are actually there. For example, in an x86 architecture, the initial 3GB `(0x00000000 to 0xBFFFFFFF)` is typically for RAM and the later 1GB `(0xC0000000 to 0xFFFFFFFF)`for device maps. However, if the RAM is less, say 2GB, device maps could start from 2GB `(0x80000000)`.

Type in `cat /proc/iomem` to list the memory map on your system. `cat /proc/meminfo` would give you an approximate RAM size on your system. Refer to Figure 12 for a snapshot.

![Figure 12](/Images/Part7/figure_12_phys_n_bus_addresses.png)

Irrespective of the actual values, the addresses referring to RAM are termed as physical addresses. And the addresses referring to device maps are termed as bus addresses, as these devices are always mapped through some architecture-specific bus. For example, PCI bus in x86 architecture, AMBA bus in ARM architectures, SuperHyway bus in SuperH (or SH) architectures, GX bus on PowerPC (or PPC), etc.

All the architecture dependent values of these physical and bus addresses are either dynamically configurable or are to be obtained from the datasheets (i.e. hardware manuals) of the corresponding architecture processors/controllers. But the interesting part is that, in Linux none of these are directly accessible but are to be mapped to virtual addresses and then accessed through that. Thus, making the RAM and device accesses generic enough, except just mapping them to virtual addresses. And the corresponding APIs for mapping & unmapping the device bus addresses to virtual addresses are:

```C
#include <asm/io.h>

void *ioremap(unsigned long device_bus_address, unsigned long device_region_size);
void iounmap(void *virt_addr);
```

These are prototyped in `<asm/io.h>`. Once mapped to virtual addresses, it boils down to the device datasheet, as to which set of device registers and/or device memory to read from or write into, by adding their offsets to the virtual address returned by `ioremap()`. For that, the following are the APIs (prototyped in the same header file `<asm/io.h>`):

```C
#include <asm/io.h>

unsigned int ioread8(void *virt_addr);
unsigned int ioread16(void *virt_addr);
unsigned int ioread32(void *virt_addr);
unsigned int iowrite8(u8 value, void *virt_addr);
unsigned int iowrite16(u16 value, void *virt_addr);
unsigned int iowrite32(u32 value, void *virt_addr);
```

## Accessing the video RAM of “DOS” days

After this first set of information, students were directed for the live experiments. They were suggested to do an initial experiment with the video RAM of “DOS” days to understand the usage of the above APIs. Shweta got onto the system – displayed the /proc/iomem window – one very similar to as shown in Figure 12. From there, she got the video RAM address ranging from `0x000A0000` to `0x000BFFFF`. And with that she added the above APIs with appropriate parameters into the constructor and destructor of her already written null driver to convert it into a vram driver. Then, she added the user access to the video RAM through read & write calls of the `vram driver`. Here’s what she coded in the new file `video_ram.c`:

```C
#include <linux/module.h>
#include <linux/version.h>
#include <linux/kernel.h>
#include <linux/types.h>
#include <linux/kdev_t.h>
#include <linux/fs.h>
#include <linux/device.h>
#include <linux/cdev.h>
#include <linux/uaccess.h>
#include <asm/io.h>

#define VRAM_BASE 0x000A0000
#define VRAM_SIZE 0x00020000

static void __iomem *vram;
static dev_t first;
static struct cdev c_dev;
static struct class *cl;

static int my_open(struct inode *i, struct file *f)
{
	return 0;
}
static int my_close(struct inode *i, struct file *f)
{
	return 0;
}
static ssize_t my_read(struct file *f, char __user *buf, size_t len, loff_t *off)
{
	int i;
	u8 byte;

	if (*off >= VRAM_SIZE)
	{
		return 0;
	}
	if (*off + len > VRAM_SIZE)
	{
		len = VRAM_SIZE - *off;
	}
	for (i = 0; i < len; i++)
	{
		byte = ioread8((u8 *)vram + *off + i);
		if (copy_to_user(buf + i, &byte, 1))
		{
			return -EFAULT;
		}
	}
	*off += len;

	return len;
}
static ssize_t my_write(
		struct file *f, const char __user *buf, size_t len, loff_t *off)
{
	int i;
	u8 byte;

	if (*off >= VRAM_SIZE)
	{
		return 0;
	}
	if (*off + len > VRAM_SIZE)
	{
		len = VRAM_SIZE - *off;
	}
	for (i = 0; i < len; i++)
	{
		if (copy_from_user(&byte, buf + i, 1))
		{
			return -EFAULT;
		}
		iowrite8(byte, (u8 *)vram + *off + i);
	}
	*off += len;

	return len;
}

static struct file_operations vram_fops =
{
	.owner = THIS_MODULE,
	.open = my_open,
	.release = my_close,
	.read = my_read,
	.write = my_write
};

static int __init vram_init(void) /* Constructor */
{
	int ret;
	struct device *dev_ret;

	if ((vram = ioremap(VRAM_BASE, VRAM_SIZE)) == NULL)
	{
		printk(KERN_ERR "Mapping video RAM failed\n");
		return -ENOMEM;
	}
	if ((ret = alloc_chrdev_region(&first, 0, 1, "vram")) < 0)
	{
		return ret;
	}
	if (IS_ERR(cl = class_create(THIS_MODULE, "chardrv")))
	{
		unregister_chrdev_region(first, 1);
		return PTR_ERR(cl);
	}
	if (IS_ERR(dev_ret = device_create(cl, NULL, first, NULL, "vram")))
	{
		class_destroy(cl);
		unregister_chrdev_region(first, 1);
		return PTR_ERR(dev_ret);
	}

	cdev_init(&c_dev, &vram_fops);
	if ((ret = cdev_add(&c_dev, first, 1)) < 0)
	{
		device_destroy(cl, first);
		class_destroy(cl);
		unregister_chrdev_region(first, 1);
		return ret;
	}
	return 0;
}

static void __exit vram_exit(void) /* Destructor */
{
	cdev_del(&c_dev);
	device_destroy(cl, first);
	class_destroy(cl);
	unregister_chrdev_region(first, 1);
	iounmap(vram);
}

module_init(vram_init);
module_exit(vram_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Anil Kumar Pugalia <email@sarika-pugs.com>");
MODULE_DESCRIPTION("Video RAM Driver");
```

## Summing up

Then, Shweta repeated the following steps:

- Build the vram driver (video_ram.ko file) by running make with the same Makefile changed to build this driver.
- Usual load of the driver using insmod video_ram.ko.
- Usual write into `/dev/vram`, say using `echo -n “0123456789” > /dev/vram`.
- Read the `/dev/vram` contents using `xxd /dev/vram | less`. The usual `cat /dev/vram` also can be used but that would give all binary content. `xxd` shows them up as hexadecimal in centre with the corresponding ASCII along the right side.
- Usual unload the driver using rmmod video_ram.

*Note 1:* Today’s systems typically use separate video cards having their own video RAM. So, the video RAM used in the “DOS days”, i.e. the one mentioned in this article is unused and many a times not even present. Hence, playing around with it, is safe, without any effect on the system, or the display.

*Note 2:* Moreover, if the video RAM is absent, the read/write may not be actually reading/writing, but just sending/receiving signals in the air. In such a case, writes would not do any change, and reads would keep on reading the same value – thus `xxd` showing the same values.

It was yet half an hour left for the practical class to be over and a lunch break. So, Shweta decided to walk around and possibly help somebody in their experiments.
