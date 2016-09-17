# Kernel Space Debuggers in Linux

> This tenth article, which is part of the series on Linux device drivers, talks about kernel space debugging in Linux.

Shweta was back from hospital and relaxing in the library by reading up various books. Since the time she has known the ioctl way of debugging, she has been impatient to know more about debugging in kernel space. The basic curiosity coming from the fact that how and where would one run the kernel space debugger, if there is any. Contrast this with application/user space debugging, where we have the OS running underneath, and a shell or a GUI over it to run the debugger, say for example, the GNU debugger (`gdb`), the data display debugger (`ddd`). And viola, she came across this interesting kernel space debugging mechanism using `kgdb`, provided as part of the kernel itself, since kernel 2.6.26

## Debugger challenge in kernel space

As we need some interface to be up, to run a debugger to debug anything, a debugger for debugging the kernel, could be visualized in 2 possible ways:

1. Put the debugger into the kernel itself. And then the debugger runs from within, accessible through the usual monitor or console. An example of it is kdb. Until kernel 2.6.35, it was not official, meaning its source code was needed to be downloaded as 2 set of patches (one architecture dependent and one architecture independent) from ftp://oss.sgi.com/projects/kdb/download/ and then to be patched with the kernel source – though since kernel 2.6.35, majority of it is part of the kernel source’s official release. In either case, the kdb support needs to be enabled in the kernel source and then the kernel is to be compiled, installed and booted with. The boot screen itself would give the `kdb debugging` interface.

2. Put a minimal debugging server into the kernel; a debugger client would then connect to it from a remote host system’s user space over some interface, say serial or ethernet. An example for that is kgdb – the kernel’s gdb server, to be used with gdb (client) from a remote system over either serial or ethernet. Since kernel 2.6.26, its serial interface part has been merged with kernel source’s official release. Though, if interested in its network interface part, it would still need to be patched with one of the releases from http://sourceforge.net/projects/kgdb/. In either case, the next step would be to enable kgdb support in the kernel source and then the kernel is to be compiled, installed and booted with – though this time to be connected with a remote debugger client.

Please note that in both the above cases, the complete kernel source for the kernel to be debugged is needed, unlike in case of building modules, where just headers are sufficient. Here is how to play around with `kgdb` over serial interface.

## Setting up the Linux kernel with kgdb

*Pre-requisite:* Either kernel source package for the running kernel is installed on your system, or a corresponding kernel source release has been downloaded from http://kernel.org.

First of all, the kernel to be debugged need to have kgdb enabled and built into it. To achieve that, the kernel source has to be configured with `CONFIG_KGDB=y`. Additionally, for `kgdb` over serial, `CONFIG_KGDB_SERIAL_CONSOLE=y` needs to be configured. And `CONFIG_DEBUG_INFO` is preferred for symbolic data to be built into kernel for making debugging with gdb more meaningful. `CONFIG_FRAME_POINTER=y` enables frame pointers in the kernel allowing gdb to construct more accurate stack back traces. All these options are available under “Kernel hacking” in the menu obtained by issuing the following commands in the kernel source directory (preferably as root or using sudo):

```C
$ make mrproper # To clean up properly
$ make oldconfig # Configure the kernel same as the current running one
$ make menuconfig # Start the ncurses based menu for further configuration
```

See the highlighted selections in Figure 15, for how and where would these options be:

- “KGDB: kernel debugging with remote gdb” → CONFIG_KGDB
	- “KGDB: use kgdb over the serial console” → CONFIG_KGDB_SERIAL_CONSOLE
- “Compile the kernel with debug info” → CONFIG_DEBUG_INFO
- “Compile the kernel with frame pointers” → CONFIG_FRAME_POINTER

![Figure 15](/Images/Part10/figure_15_configuring_kernel_with_kgdb-1024x640.png)

Once configured and saved, the kernel can be built by typing make in the kernel source directory. And then a make install is expected to install it, along with adding an entry for the installed kernel in the grub configuration file. Depending on the distribution, the grub configuration file may be `/boot/grub/menu.lst`, `/etc/grub.cfg`, or something similar. Once installed, the kgdb related kernel boot parameters, need to be added to this newly added entry.

![Figure 16](/Images/Part10/figure_16_grub_config_for_kernel_with_kgdb-1024x549.png)

Figure 16 highlights the kernel boot parameters added to the newly installed kernel, in the grub‘s configuration file.

`kgdboc` is for gdb connecting over console and the basic format is: `kgdboc=<serial_device>,<baud_rate>`
where:
`<serial_device>` is the serial device file on the system, which would run the kernel to be debugged, for the serial port to be used for debugging
`<baud_rate>` is the baud rate of the serial port to be used for debugging

`kgdbwait` enables the booting kernel to wait till a remote gdb (i.e. gdb on another system) connects to it, and this parameter should be passed only after kgdboc.

With this, the system is ready to reboot into this newly built and installed kernel. On reboot, at the grub‘s menu, select to boot from this new kernel, and then it will wait for gdb to connect with it from an another system over the serial port.

All the above snapshots have been with kernel source 2.6.33.14, downloaded from http://kernel.org. And the same should work for at the least any 2.6.3x release of kernel source. Also, the snapshots are captured for kgdb over serial device file `/dev/ttyS0`, i.e. the first serial port.

## Setting up gdb on another system

*Pre-requisite:* Serial ports of the system to be debugged and the another system to run gdb from, should be connected using a null modem (i.e. a cross over serial) cable.

Here are the gdb commands to get the gdb from the other system connect to the waiting kernel. All these commands have to be given on the gdb prompt, after typing gdb on the shell.

Connecting over serial port /dev/ttyS0 (of the system running gdb) with baud rate 115200 bps:

```
$ gdb
...
(gdb) file vmlinux
(gdb) set remote interrupt-sequence Ctrl-C
(gdb) set remotebaud 115200
(gdb) target remote /dev/ttyS0
(gdb) continue
```

In the above commands, vmlinux is the kernel image built with kgdb enabled and needs to be copied into the directory on the system, from where gdb is being executed. Also, the serial device file and its baud rate has to be correctly given as per one’s system setup.

## Debugging using gdb with kgdb

After this, it is all like debugging an application from gdb. One may stop execution using `Ctrl+C`, add break points using b[reak], step execution using s[tep] or n[ext], … – the usual gdb way. For details on how to use gdb, there are enough tutorials available online. In fact, if not comfortable with text-based gdb, the debugging could be made GUI-based, using any of the standard GUI tools over gdb, for example, `ddd`, Eclipse, etc.

## Summing up

By now, Shweta was all excited to try out the kernel space debugging experiment using kgdb. And as she needed two systems to try it out, she decided to go to the Linux device drivers lab. There, she set up the system to be debugged, with the kgdb enabled kernel, and connected it with an another system using a null modem cable. Then on the second system, she executed gdb to remotely connect with and step through the kernel on the first system.

## Notes

Parameter for using kgdb over ethernet is kgdboe. It has the following format: `kgdboe=[<this_udp_port>]@<this_ip>/[this_dev],[<remote_udp_port>]@<remote_ip>/[<remote_mac_addr>]`
where:
`<this_udp_port>` is optional and defaults to 6443,
`<this_ip>` is IP address of the system, which would run the kernel to be debugged,
`<this_dev>` is optional and defaults to eth0,
`<remote_udp_port>` is optional and defaults to 6442,
`<remote_ip>` is IP address of the system, from which gdb would be connecting from,
`<remote_mac_addr>` is optional and defaults to broadcast.

Here are the gdb commands for connecting to kgdb over network port 6443 to IP address `192.168.1.2`:

```C
$ gdb
...
(gdb) file vmlinux
(gdb) set remotebreak 0
(gdb) target remote udp:192.168.1.2:6443
(gdb) continue
```

