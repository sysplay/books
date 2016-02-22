# Kernel C Extras in a Linux Driver

> This third article, in the series on Linux device drivers deals with the kernel’s message logging,
and kernel-specific GCC extensions.

Enthused by how Pugs impressed professor Gopi, in the last class, Shweta decided to do something similar. And there was already an opportunity – finding out where has the output of printk gone. So, as soon as she entered the lab, she got hold of the best located system, logged into it, and took charge. Knowing her professor pretty well, she knew that there would be a hint for the finding, from the class itself. So, she flashed back what all the professor taught, and suddenly remembered the error output demonstration from `insmod vfat.ko` – `dmesg | tail`. She immediately tried that and for sure found out the printk output, there. But how did it come here? A tap on her shoulder brought her out of the thought. "Shall we go for a coffee?", proposed Pugs. "But I need to …". "I know what you are thinking about.", interrupted Pugs. "Let's go, yaar. I’ll explain you all about `dmesg`".

## Kernel Message Logging

On the coffee table, Pugs began:
As far as parameters are concerned, printf & printk are same, except that when programming for the kernel we don’t bother about the float formats of %f, %lf & their likes. However unlike printf, printk is not destined to dump its output on some console. In fact, it cannot do so, as it is something which is in the background, and executes like a library, only when triggered either from the hardware space or the user space. So, then where does `printk` print? All the `printk` calls, just put their contents into the (log) ring buffer of the kernel. Then, the syslog daemon running in the user space picks them for final processing & redirection to various devices, as configured in its configuration file `/etc/syslog.conf`.

You must have observed the out of place macro KERN_INFO, in the printk calls, in the previous article. That actually is a constant string, which gets concatenated with the format string after it, making it a single string. Note that there is no comma (,) between them – they are no two separate arguments. There are eight such macros defined in `<linux/kernel.h>` under the kernel source, namely:

```C
#define KERN_EMERG	"<0>" /* system is unusable			*/
#define KERN_ALERT	"<1>" /* action must be taken immediately	*/
#define KERN_CRIT	"<2>" /* critical conditions			*/
#define KERN_ERR	"<3>" /* error conditions			*/
#define KERN_WARNING	"<4>" /* warning conditions			*/
#define KERN_NOTICE	"<5>" /* normal but significant condition	*/
#define KERN_INFO	"<6>" /* informational				*/
#define KERN_DEBUG	"<7>" /* debug-level messages			*/
```

Depending on these log levels (i.e. the first 3 characters in the format string), the syslog daemon in the user space redirects the corresponding messages to their configured locations – a typical one being the log file `/var/log/messages` for all the log levels. Hence, all the `printk` outputs are by default in that file. Though, they can be configured differently to say serial port `(/dev/ttyS0)` or say all consoles, like what happens typically for `KERN_EMERG`. Now, `/var/log/messages` is buffered & contain messages not only from the kernel but also from various daemons running in the user space. Moreover, the `/var/log/messages` most often is not readable by a normal user, and hence a user-space utility `dmesg` is provided to directly parse the kernel ring buffer and dump it on the standard output. Figure 6 shows the snippets from the two.

![Figure 6](/Images/Part3/figure_6_kernels_message_logging.png)

## Kernel-specific GCC extensions

With all these Shweta got frustrated, as she wanted to find all these by her own, and then do a impression in the next class – but all flop. Pissed off, she said, “So as you have explained all about printing in kernel, why don’t you tell about the weird C in the driver as well – the special keywords `__init`, `__exit`, etc.”

These are not any special keywords. Kernel C is not any weird C but just the standard C with some additional extensions from the C compiler `gcc`. Macros `__init` and `__exit` are just two of these extensions. However, these do not have any relevance in case we are using them for dynamically loadable driver, but only when the same code gets built into the kernel. All the functions marked with `__init` get placed inside the init section of the kernel image and all functions marked with `__exit` are placed inside the exit section of the kernel image, automatically by gcc, during kernel compilation. What is the benefit? All functions with `__init` are supposed to be executed only once during boot-up, till the next boot-up. So, once they are executed during boot-up, kernel frees up RAM by removing them by freeing up the init section. Similarly, all functions in exit section are supposed to be called during system shutdown. Now, if system is shutting down anyway, why do you need to do any cleanups. Hence, the exit section is not even built into the kernel – another cool optimization.

This is a beautiful example of how kernel & gcc goes hand-in-hand to achieve lot of optimizations and many other tricks – we could see others, as we go along. And that is why Linux kernel can be compiled only using gcc-based compilers – a close knit bond.

## Kernel function’s return guidelines

While returning from coffee, Pugs started all praises for the OSS & its community. Do you know why different individuals are able to come together and contribute excellently without any conflicts – moreover in a project as huge as Linux? There are many reasons. But definitely, one of the strong reasons is, following & abiding by the inherent coding guidelines. Take for example the guideline for returning values from a function in kernel programming.

Any kernel function needing error handling, typically returns an integer-like type and the return value again follows a guideline. For an error, we return a negative number – a minus sign appended with a macro included through the kernel header `<linux/errno.h>`, that includes the various error number headers under the kernel sources, namely `<asm/errno.h>`, `<asm-generic/errno.h>`, `<asm-generic/errno-base.h>`. For success, zero is the most common return value, unless there is some additional information to be provided. In that case, a positive value is returned, the value indicating the information like number of bytes transferred.

## Kernel C = Pure C

Once back into the lab, Shweta remembered their professor mentioning that no /usr/include headers can be used for kernel programming. But Pugs said that kernel C is just standard C with some gcc extensions. Why this conflict? Actually this is not a conflict. Standard C is just pure C – just the language. The headers are not part of it. Those are part of the standard libraries built in C for C programmers, based on the concept of re-using code. Does that mean, all standard libraries and hence all ANSI standard functions are not part of ‘pure’ C? Yes. Then, hadn’t it been really tough coding the kernel. Not for this reason. In reality, kernel developers have developed their own needed set of functions, and they are all part of the kernel code. printk is just one of them. Similarly, many string functions, memory functions, … are all part of the kernel source under various directories like kernel, ipc, lib, … and the corresponding headers under include/linux directory.

“O ya! That is why we need to have kernel source for building a driver”, affirmed Shweta. “If not the complete source, at least the headers are a must. And that is why we have separate packages to install complete kernel source or just the kernel headers”, added Pugs. “In the lab, all the sources are setup. But if I want to try out drivers on my Linux system at my hostel room, how do I go about it?” asked Shweta. “Our lab have Fedora, where the kernel sources typically get installed under `/usr/src/kernels/<kernel_version>` unlike the standard place `/usr/src/linux`. Lab administrators must have installed it using command line `yum install kernel-devel`. I use Mandriva and installed the kernel sources using `urpmi kernel-source`, replied Pugs. `but, I have Ubuntu`. "Okay!! For that just use `apt-get install` – possibly, `apt-get install linux-source`".

## Summing up

Lab timings were just getting over. Suddenly, Shweta put out her curiosity – "Hey Pugs! What is the next topic we are going to learn in our Linux device drivers class?". "Hmmm!! Most probably character drivers". With this information, Shweta hurriedly packed up her bag & headed towards her room to setup the kernel sources and try out the next driver on her own. "In case you are stuck up, just give me a call. I’ll be there", called up Pugs from the behind with a smile.

