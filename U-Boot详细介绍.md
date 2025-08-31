# U-Boot 详细介绍：嵌入式系统的通用引导加载程序

## 目录
1. [U-Boot 概述](#1-u-boot-概述)
2. [U-Boot 架构与核心组件](#2-u-boot-架构与核心组件)
3. [启动流程详解](#3-启动流程详解)
4. [配置系统](#4-配置系统)
5. [设备树支持](#5-设备树支持)
6. [启动命令详解](#6-启动命令详解)
7. [内存管理](#7-内存管理)
8. [网络功能](#8-网络功能)
9. [文件系统支持](#9-文件系统支持)
10. [安全启动](#10-安全启动)
11. [移植和开发](#11-移植和开发)
12. [调试和优化](#12-调试和优化)

---

## 1. U-Boot 概述

### 1.1 什么是 U-Boot

U-Boot（Universal Boot Loader）是一个开源的通用引导加载程序，广泛应用于嵌入式系统中。它的主要职责是初始化硬件、加载操作系统内核，并将控制权交给内核。

从项目的 README 文件可以看到：

```text
This directory contains the source code for U-Boot, a boot loader for
Embedded boards based on PowerPC, ARM, MIPS and several other
processors, which can be installed in a boot ROM and used to
initialize and test the hardware or to download and run application
code.
```

### 1.2 支持的架构

U-Boot 支持多种处理器架构：

```text
/arch			Architecture specific files
  /arc			Files generic to ARC architecture
  /arm			Files generic to ARM architecture
  /m68k			Files generic to m68k architecture
  /microblaze		Files generic to microblaze architecture
  /mips			Files generic to MIPS architecture
  /nds32		Files generic to NDS32 architecture
  /nios2		Files generic to Altera NIOS2 architecture
  /openrisc		Files generic to OpenRISC architecture
  /powerpc		Files generic to PowerPC architecture
  /sandbox		Files generic to HW-independent "sandbox"
  /sh			Files generic to SH architecture
  /x86			Files generic to x86 architecture
```

### 1.3 主要特性

- **多架构支持**：支持 ARM、PowerPC、MIPS、x86 等多种处理器架构
- **灵活配置**：基于 Kconfig 的配置系统
- **设备树支持**：完整的设备树（Device Tree）支持
- **网络功能**：支持 TFTP、DHCP、NFS 等网络协议
- **多种启动方式**：支持从 Flash、SD 卡、网络等多种介质启动
- **命令行界面**：提供强大的命令行工具用于调试和配置

## 2. U-Boot 架构与核心组件

### 2.1 目录结构

```text
/api			Machine/arch independent API for external apps
/board			Board dependent files
/cmd			U-Boot commands functions
/common			Misc architecture independent functions
/configs		Board default configuration files
/disk			Code for disk drive partition handling
/doc			Documentation (don't expect too much)
/drivers		Commonly used device drivers
/dts			Contains Makefile for building internal U-Boot fdt.
/examples		Example code for standalone applications, etc.
/fs			Filesystem code (cramfs, ext2, jffs2, etc.)
/include		Header Files
/lib			Library routines generic to all architectures
/net			Networking code
/post			Power On Self Test
/scripts		Various build scripts and Makefiles
/test			Various unit test files
/tools			Tools to build S-Record or U-Boot images, etc.
```

### 2.2 启动状态机

U-Boot 的启动过程可以分为多个状态，这在 `common/bootm.c` 中有详细体现：

```c
/**
 * Execute selected states of the bootm command.
 *
 * Note the arguments to this state must be the first argument, Any 'bootm'
 * or sub-command arguments must have already been taken.
 *
 * Note that if states contains more than one flag it MUST contain
 * BOOTM_STATE_START, since this handles and consumes the
 * 'bootm' command line args.
 *
 * Also note that aside from boot_os_fn functions and bootm_load_os no
 * other functions we store the return value of in 'ret' may use a
 * negative return value, without special handling.
 *
 * @param cmdtp		Pointer to command table entry for bootm
 * @param flag		Command flags (CMD_FLAG_...)
 * @param argc		Number of subcommand arguments (0 = no arguments)
 * @param argv		Arguments
 * @param states	Mask containing states to run (BOOTM_STATE_...)
 * @param images	Image header information
 * @param boot_progress 1 to show boot progress, 0 to not do this
 * @return 0 if ok, something else on error
 */
int do_bootm_states(cmd_tbl_t *cmdtp, int flag, int argc, char * const argv[],
		    int states, bootm_headers_t *images, int boot_progress)
{
	boot_os_fn *boot_fn;
	ulong iflag = 0;
	int ret = 0, need_boot_fn;
	u32 unmask;

	unmask = env_get_ulong("bootm_states_unmask", 16, 0);
	if (unmask)
		states &= ~unmask;

	images->state |= states;

	/*
	 * Work through the states and see how far we get. We stop on
	 * any error.
	 */
	if (states & BOOTM_STATE_START)
		ret = bootm_start(cmdtp, flag, argc, argv);

	if (!ret && (states & BOOTM_STATE_FINDOS))
		ret = bootm_find_os(cmdtp, flag, argc, argv);

	if (!ret && (states & BOOTM_STATE_FINDOTHER))
		ret = bootm_find_other(cmdtp, flag, argc, argv);

	/* Load the OS */
	if (!ret && (states & BOOTM_STATE_LOADOS)) {
		iflag = bootm_disable_interrupts();
		ret = bootm_load_os(images, &load_end, 0);
		if (ret && ret != BOOTM_ERR_OVERLAP)
			goto err;
		else if (ret == BOOTM_ERR_OVERLAP)
			ret = 0;
#if CONFIG_IS_ENABLED(LEGACY_IMAGE_FORMAT)
		if (images->legacy_hdr_valid) {
			image_start = images->legacy_hdr_os_data_start;
			image_len = images->legacy_hdr_os_data_size;
		} else {
			image_start = (ulong)images->os.image_start;
			image_len = images->os.image_len;
		}
#endif
	}

	/* Relocate the ramdisk */
#ifdef CONFIG_SYS_BOOT_RAMDISK_HIGH
	if (!ret && (states & BOOTM_STATE_RAMDISK)) {
		ulong rd_len = images->rd_end - images->rd_start;

		ret = boot_ramdisk_high(&images->lmb, images->rd_start,
			rd_len, &images->initrd_start, &images->initrd_end);
		if (!ret) {
			env_set_hex("initrd_start", images->initrd_start);
			env_set_hex("initrd_end", images->initrd_end);
		}
	}
#endif
```

### 2.3 ARM 架构启动实现

在 `arch/arm/lib/bootm.c` 中，我们可以看到 ARM 架构特定的启动实现：

```c
/* Main Entry point for arm bootm implementation
 *
 * Modeled after the powerpc implementation
 * DIFFERENCE: Instead of calling prep and go at the end
 * they are called if subcommand is equal 0.
 */
int do_bootm_linux(int flag, int argc, char * const argv[],
		   bootm_headers_t *images)
{
	/* No need for those on ARM */
	if (flag & BOOTM_STATE_OS_BD_T || flag & BOOTM_STATE_OS_CMDLINE)
		return -1;

	if (flag & BOOTM_STATE_OS_PREP) {
		boot_prep_linux(images);
		return 0;
	}

	if (flag & (BOOTM_STATE_OS_GO | BOOTM_STATE_OS_FAKE_GO)) {
		boot_jump_linux(images, flag);
		return 0;
	}

	boot_prep_linux(images);
	boot_jump_linux(images, flag);
	return 0;
}
```

### 2.4 跳转到 Linux 内核

```c
/* Subcommand: GO */
static void boot_jump_linux(bootm_headers_t *images, int flag)
{
#ifdef CONFIG_ARM64
	void (*kernel_entry)(void *fdt_addr, void *res0, void *res1,
			void *res2);
	int fake = (flag & BOOTM_STATE_OS_FAKE_GO);
	int es_flag = 0;

#if defined(CONFIG_AMP)
	es_flag = arm64_switch_amp_pe(images);
#elif defined(CONFIG_ARM64_SWITCH_TO_AARCH32)
	es_flag = arm64_switch_aarch32(images);
#endif
	kernel_entry = (void (*)(void *fdt_addr, void *res0, void *res1,
			void *res2))images->ep;

	debug("## Transferring control to Linux (at address %lx)...\n",
		(ulong) kernel_entry);
	bootstage_mark(BOOTSTAGE_ID_RUN_OS);

	announce_and_cleanup(images, fake);

	if (!fake) {
#ifdef CONFIG_ARMV8_PSCI
		armv8_setup_psci();
#endif
		do_nonsec_virt_switch();

		if (es_flag == ARM64_SWITCH_ES_FAIL) {
			if (!armv7_init_nonsec())
				armv7_init_nonsec();
			secure_ram_addr(_do_nonsec_entry)(kernel_entry,
							  0, machid, r2);
		} else
#endif
			kernel_entry(0, machid, r2);
	}
#endif
}
```

## 3. 启动流程详解

### 3.1 初始化阶段

U-Boot 的启动流程在 README 中有详细说明：

```text
Board Initialisation Flow:
--------------------------

This is the intended start-up flow for boards. This should apply for both
SPL and U-Boot proper (i.e. they both follow the same rules).

Note: "SPL" stands for "Secondary Program Loader," which is explained in
more detail later in this file.

At present, SPL mostly uses a separate code path, but the function names
and roles of each function are the same. Some boards or architectures
may not conform to this.  At least most ARM boards which use
CONFIG_SPL_FRAMEWORK conform to this.

Execution typically starts with an architecture-specific (and possibly
CPU-specific) start.S file, such as:

	- arch/arm/cpu/armv7/start.S
	- arch/powerpc/cpu/mpc83xx/start.S
	- arch/mips/cpu/start.S

and so on. From there, three functions are called; the purpose and
limitations of each of these functions are described below.

lowlevel_init():
	- purpose: essential init to permit execution to reach board_init_f()
	- no global_data or BSS
	- there is no stack (ARMv7 may have one but it will soon be removed)
	- must not set up SDRAM or use console
	- must only do the bare minimum to allow execution to continue to
		board_init_f()
	- this is almost never needed
	- return normally from this function

board_init_f():
	- purpose: set up the machine ready for running board_init_r():
		i.e. SDRAM and serial UART
	- global_data is available
	- stack is in SRAM
	- BSS is not available, so you cannot use global/static variables,
		only stack variables and global_data

	Non-SPL-specific notes:
	- dram_init() is called to set up DRAM. If already done in SPL this
		can do nothing

	SPL-specific notes:
	- you can override the entire board_init_f() function with your own
		version as needed.
	- preloader_console_init() can be called here in extremis
	- should set up SDRAM, and anything needed to make the UART work
	- these is no need to clear BSS, it will be done by crt0.S
	- must return normally from this function (don't call board_init_r()
		directly)

Here the BSS is cleared. For SPL, if CONFIG_SPL_STACK_R is defined, then at
this point the stack and global_data are relocated to below
CONFIG_SPL_STACK_R_ADDR. For non-SPL, U-Boot is relocated to run at the top of
memory.

board_init_r():
	- purpose: main execution, common code
	- global_data is available
	- SDRAM is available
	- BSS is available, all static/global variables can be used
	- execution eventually continues to main_loop()

	Non-SPL-specific notes:
	- U-Boot is relocated to the top of memory and is now running from
		there.

	SPL-specific notes:
	- stack is optionally in SDRAM, if CONFIG_SPL_STACK_R is defined and
		CONFIG_SPL_STACK_R_ADDR points into SDRAM
	- preloader_console_init() can be called here - typically this is
		done by selecting CONFIG_SPL_BOARD_INIT and then supplying a
		spl_board_init() function containing this call
	- loads U-Boot or (in falcon mode) Linux
```

### 3.2 具体的启动示例

在板级配置文件中，我们可以看到启动命令的具体配置，比如在 `include/configs/ib62x0.h` 中：

```c
/*
 * Default environment variables
 */
#define CONFIG_BOOTCOMMAND \
	"setenv bootargs ${console} ${mtdparts} ${bootargs_root}; "	\
	"ubi part root; "						\
	"ubifsmount ubi:rootfs; "					\
	"ubifsload 0x800000 ${kernel}; "				\
	"ubifsload 0x700000 ${fdt}; "					\
	"ubifsumount; "							\
	"fdt addr 0x700000; fdt resize; fdt chosen; "			\
	"bootz 0x800000 - 0x700000"
```

这个例子展示了一个完整的启动序列：
1. 设置启动参数（`setenv bootargs`）
2. 挂载 UBI 文件系统（`ubi part root; ubifsmount ubi:rootfs`）
3. 加载内核到内存（`ubifsload 0x800000 ${kernel}`）
4. 加载设备树到内存（`ubifsload 0x700000 ${fdt}`）
5. 卸载文件系统（`ubifsumount`）
6. 处理设备树（`fdt addr 0x700000; fdt resize; fdt chosen`）
7. 启动内核（`bootz 0x800000 - 0x700000`）

## 4. 配置系统

### 4.1 Kconfig 配置

U-Boot 使用 Linux 内核相同的 Kconfig 配置系统：

```text
Previously, all configuration was done by hand, which involved creating
symbolic links and editing configuration files manually. More recently,
U-Boot has added the Kbuild infrastructure used by the Linux kernel,
allowing you to use the "make menuconfig" command to configure your
build.


Selection of Processor Architecture and Board Type:
---------------------------------------------------

For all supported boards there are ready-to-use default
configurations available; just type "make <board_name>_defconfig".

Example: For a TQM823L module type:

	cd u-boot
	make TQM823L_defconfig
```

### 4.2 配置选项类型

配置选项分为两类：

```text
There are two classes of configuration variables:

* Configuration _OPTIONS_:
  These are selectable by the user and have names beginning with
  "CONFIG_".

* Configuration _SETTINGS_:
  These depend on the hardware etc. and should not be meddled with if
  you don't know what you're doing; they have names beginning with
  "CONFIG_SYS_".
```

### 4.3 板级配置示例

在 `include/configs/UCP1020.h` 中可以看到详细的板级配置：

```c
"flashuboot=tftp $ubootaddr $ubootfile; "				\
	"protect off $nor_ubootaddr +$filesize; "			\
	"erase $nor_ubootaddr +$filesize; "				\
	"cp.b $ubootaddr $nor_ubootaddr $filesize; "			\
	"protect on $nor_ubootaddr +$filesize\0 "			\
"flashworking=tftp $workingaddr $cramfsfile; "				\
	"protect off $nor_workingaddr +$filesize; "			\
	"erase $nor_workingaddr +$filesize; "				\
	"cp.b $workingaddr $nor_workingaddr $filesize; "		\
	"protect on $nor_workingaddr +$filesize\0 "			\
"hwconfig=usb1:dr_mode=host,phy_type=ulpi\0 "				\
"kerneladdr=0x01100000\0"						\
"kernelfile=uImage\0"							\
"loadaddr=0x01000000\0"							\
```

这个配置定义了：
- Flash 更新命令（`flashuboot`, `flashworking`）
- 硬件配置（`hwconfig`）
- 内存地址设置（`kerneladdr`, `loadaddr`）
- 文件名配置（`kernelfile`）

## 5. 设备树支持

### 5.1 设备树配置

U-Boot 支持三种设备树配置方式：

```text
- Device tree:
		CONFIG_OF_CONTROL
		If this variable is defined, U-Boot will use a device tree
		to configure its devices, instead of relying on statically
		compiled #defines in the board file. This option is
		experimental and only available on a few boards. The device
		tree is available in the global data as gd->fdt_blob.

		U-Boot needs to get its device tree from somewhere. This can
		be done using one of the three options below:

		CONFIG_OF_EMBED
		If this variable is defined, U-Boot will embed a device tree
		binary in its image. This device tree file should be in the
		board directory and called <soc>-<board>.dts. The binary file
		is then picked up in board_init_f() and made available through
		the global data structure as gd->fdt_blob.

		CONFIG_OF_SEPARATE
		If this variable is defined, U-Boot will build a device tree
		binary. It will be called u-boot.dtb. Architecture-specific
		code will locate it at run-time. Generally this works by:

			cat u-boot.bin u-boot.dtb >image.bin

		and in fact, U-Boot does this for you, creating a file called
		u-boot-dtb.bin which is useful in the common case. You can
		still use the individual files if you need something more
		exotic.

		CONFIG_OF_BOARD
		If this variable is defined, U-Boot will use the device tree
		provided by the board at runtime instead of embedding one with
		the image. Only boards defining board_fdt_blob_setup() support
		this option (see include/fdtdec.h file).
```

### 5.2 设备树启动示例

在 README 中提供了设备树启动的具体示例：

```text
Boot Linux and pass a flat device tree:
-----------

First, U-Boot must be compiled with the appropriate defines. See the section
titled "Linux Kernel Interface" above for a more in depth explanation. The
following is an example of how to start a kernel and pass an updated
flat device tree:

=> print oftaddr
oftaddr=0x300000
=> print oft
oft=oftrees/mpc8540ads.dtb
=> tftp $oftaddr $oft
Speed: 1000, full duplex
Using TSEC0 device
TFTP from server 192.168.1.1; our IP address is 192.168.1.101
Filename 'oftrees/mpc8540ads.dtb'.
Load address: 0x300000
Loading: #
done
Bytes transferred = 4106 (100a hex)
=> tftp $loadaddr $bootfile
Speed: 1000, full duplex
Using TSEC0 device
TFTP from server 192.168.1.1; our IP address is 192.168.1.2
Filename 'uImage'.
Load address: 0x200000
Loading:############
done
Bytes transferred = 1029407 (fb51f hex)
=> print loadaddr
loadaddr=200000
=> print oftaddr
oftaddr=0x300000
=> bootm $loadaddr - $oftaddr
## Booting image at 00200000 ...
   Image Name:	 Linux-2.6.17-dirty
   Image Type:	 PowerPC Linux Kernel Image (gzip compressed)
   Data Size:	 1029343 Bytes = 1005.2 kB
   Load Address: 00000000
   Entry Point:	 00000000
   Verifying Checksum ... OK
   Uncompressing Kernel Image ... OK
Booting using flat device tree at 0x300000
Using MPC85xx ADS machine description
Memory CAM mapping: CAM0=256Mb, CAM1=256Mb, CAM2=0Mb residual: 0Mb
[snip]
```

这个例子展示了：
1. 设置设备树地址（`oftaddr=0x300000`）
2. 通过 TFTP 下载设备树文件（`tftp $oftaddr $oft`）
3. 下载内核镜像（`tftp $loadaddr $bootfile`）
4. 使用设备树启动内核（`bootm $loadaddr - $oftaddr`）

## 6. 启动命令详解

### 6.1 bootm 命令

`bootm` 是 U-Boot 中最重要的启动命令，在 `cmd/bootm.c` 中实现：

```c
static int do_bootm_subcommand(cmd_tbl_t *cmdtp, int flag, int argc,
			char * const argv[])
{
	int ret = 0;
	long state;
	cmd_tbl_t *c;

	c = find_cmd_tbl(argv[0], &cmd_bootm_sub[0], ARRAY_SIZE(cmd_bootm_sub));
	argc--; argv++;

	if (c) {
		state = (long)c->cmd;
		if (state == BOOTM_STATE_START)
			state |= BOOTM_STATE_FINDOS | BOOTM_STATE_FINDOTHER;
	} else {
		/* Unrecognized command */
		return CMD_RET_USAGE;
	}

	if (((state & BOOTM_STATE_START) != BOOTM_STATE_START) &&
	    images.state >= state) {
		printf("Trying to execute a command out of order\n");
		return CMD_RET_USAGE;
	}

	ret = do_bootm_states(cmdtp, flag, argc, argv, state, &images, 1);

	return ret;
}
```

### 6.2 bootm 子命令

bootm 支持多个子命令：

```c
static cmd_tbl_t cmd_bootm_sub[] = {
	U_BOOT_CMD_MKENT(start, 0, 1, (void *)BOOTM_STATE_START, "", ""),
	U_BOOT_CMD_MKENT(loados, 0, 1, (void *)BOOTM_STATE_LOADOS, "", ""),
#ifdef CONFIG_SYS_BOOT_RAMDISK_HIGH
	U_BOOT_CMD_MKENT(ramdisk, 0, 1, (void *)BOOTM_STATE_RAMDISK, "", ""),
#endif
#ifdef CONFIG_OF_LIBFDT
	U_BOOT_CMD_MKENT(fdt, 0, 1, (void *)BOOTM_STATE_OS_CMDLINE, "", ""),
#endif
	U_BOOT_CMD_MKENT(cmdline, 0, 1, (void *)BOOTM_STATE_OS_CMDLINE, "", ""),
	U_BOOT_CMD_MKENT(bdt, 0, 1, (void *)BOOTM_STATE_OS_BD_T, "", ""),
	U_BOOT_CMD_MKENT(prep, 0, 1, (void *)BOOTM_STATE_OS_PREP, "", ""),
	U_BOOT_CMD_MKENT(fake, 0, 1, (void *)BOOTM_STATE_OS_FAKE_GO, "", ""),
	U_BOOT_CMD_MKENT(go, 0, 1, (void *)BOOTM_STATE_OS_GO, "", ""),
};
```

### 6.3 命令语法扩展

对于新的 uImage 格式（FIT），U-Boot 支持扩展的命令语法，如 `doc/uImage.FIT/command_syntax_extensions.txt` 中所述：

```text
With the introduction of the new uImage format, bootm command (and other
commands as well) have to understand new syntax of the arguments. This is
necessary in order to specify objects contained in the new uImage, on which
bootm has to operate.

bootm usage scenarios
---------------------

Below is a summary of bootm usage scenarios, focused on booting a PowerPC
Linux kernel. The purpose of the following list is to document a complete list
of supported bootm usages.

11. bootm [<addr1>]:<subimg1> [<addr2>]:<subimg2>
12. bootm [<addr1>]:<subimg1> [<addr2>]:<subimg2> [<addr3>]:<subimg3>
13. bootm [<addr1>]:<subimg1> [<addr2>]:<subimg2> <addr3>
14. bootm [<addr1>]:<subimg1> -			  [<addr3>]:<subimg3>
15. bootm [<addr1>]:<subimg1> -			  <addr3>
```

例如：
```bash
bootm 200000:kernel@1 400000:ramdisk@1 400000:fdt@1
```

这个命令启动：
- 地址 200000 处的内核镜像（kernel@1）
- 地址 400000 处的 ramdisk 镜像（ramdisk@1）
- 地址 400000 处的设备树（fdt@1）

## 7. 内存管理

### 7.1 内存映射

U-Boot 的内存管理在 README 中有详细说明：

```text
Memory Management:
------------------

U-Boot runs in system state and uses physical addresses, i.e. the
MMU is not used either for address mapping nor for memory protection.

The available memory is mapped to fixed addresses using the memory
controller. In this process, a contiguous block is formed for each
memory type (Flash, SDRAM, SRAM), even when it consists of several
physical memory banks.

U-Boot is installed in the first 128 kB of the first Flash bank (on
TQM8xxL modules this is the range 0x40000000 ... 0x4001FFFF). After
booting and sizing and initializing DRAM, the code relocates itself
to the upper end of DRAM. Immediately below the U-Boot code some
memory is reserved for use by malloc() [see CONFIG_SYS_MALLOC_LEN
configuration setting]. Below that, a structure with global Board
Info data is placed, followed by the stack (growing downward).

Additionally, some exception handler code is copied to the low 8 kB
of DRAM (0x00000000 ... 0x00001FFF).

So a typical memory configuration with 16 MB of DRAM could look like
this:

	0x0000 0000	Exception Vector code
	      :
	0x0000 1FFF
	0x0000 2000	Free for Application Use
	      :
	      :

	      :
	      :
	0x00FB FF20	Monitor Stack (Growing downward)
	0x00FB FFAC	Board Info Data and permanent copy of global data
	0x00FC 0000	Malloc Arena
	      :
	0x00FD FFFF
	0x00FE 0000	RAM Copy of Monitor Code
	...		eventually: LCD or video framebuffer
	...		eventually: pRAM (Protected RAM - unchanged by reset)
	0x00FF FFFF	[End of RAM]
```

### 7.2 环境变量内存配置

在代码中可以看到多个与内存相关的环境变量：

```c
"kerneladdr=0x01100000\0"						\
"kernelfile=uImage\0"							\
"loadaddr=0x01000000\0"							\
```

还有高级内存配置：

```text
  initrd_high	- restrict positioning of initrd images:
		  If this variable is not set, initrd images will be
		  copied to the highest possible address in RAM; this
		  is usually what you want since it allows for
		  maximum initrd size. If for some reason you want to
		  make sure that the initrd image is loaded below the
		  CONFIG_SYS_BOOTMAPSZ limit, you can set this environment
		  variable to a value of "no" or "off" or "0".
		  Alternatively, you can set it to a maximum upper
		  address to use (U-Boot will still check that it
		  does not overwrite the U-Boot stack and data).

		  For instance, when you have a system with 16 MB
		  RAM, and want to reserve 4 MB from use by Linux,
		  you can do this by adding "mem=12M" to the value of
		  the "bootargs" variable. However, now you must make
		  sure that the initrd image is placed in the first
		  12 MB as well - this can be done with

		  setenv initrd_high 00c00000

		  If you set initrd_high to 0xFFFFFFFF, this is an
		  indication to U-Boot that all addresses are legal
		  for the Linux kernel, including addresses in flash
		  memory. In this case U-Boot will NOT COPY the
		  ramdisk at all. This may be useful to reduce the
		  boot time on your system, but requires that this
		  feature is supported by your Linux kernel.
```

## 8. 网络功能

### 8.1 网络环境变量

U-Boot 支持丰富的网络功能，相关环境变量包括：

```text
  ipaddr	- IP address; needed for tftpboot command

  loadaddr	- Default load address for commands like "bootp",
		  "rarpboot", "tftpboot", "loadb" or "diskboot"

  loads_echo	- see CONFIG_LOADS_ECHO

  serverip	- TFTP server IP address; needed for tftpboot command

  bootretry	- see CONFIG_BOOT_RETRY_TIME

  bootdelaykey	- see CONFIG_AUTOBOOT_DELAY_STR

  bootstopkey	- see CONFIG_AUTOBOOT_STOP_STR

  ethprime	- controls which interface is used first.

  ethact	- controls which interface is currently active.
		  For example you can do the following

		  => setenv ethact FEC
		  => ping 192.168.0.1 # traffic sent on FEC
		  => setenv ethact SCC
		  => ping 10.0.0.1 # traffic sent on SCC

  ethrotate	- When set to "no" U-Boot does not go through all
		  available network interfaces.
		  It just stays at the currently selected interface.

  netretry	- When set to "no" each network operation will
		  either succeed or fail without retrying.
		  When set to "once" the network operation will
		  fail when all the available network interfaces
		  are tried once without success.
		  Useful on scripts which control the retry operation
		  themselves.
```

### 8.2 TFTP 配置

TFTP 是 U-Boot 中常用的网络传输协议：

```text
  tftpsrcp	- If this is set, the value is used for TFTP's
		  UDP source port.

  tftpdstp	- If this is set, the value is used for TFTP's UDP
		  destination port instead of the Well Know Port 69.

  tftpblocksize - Block size to use for TFTP transfers; if not set,
		  we use the TFTP server's default block size

  tftptimeout	- Retransmission timeout for TFTP packets (in milli-
		  seconds, minimum value is 1000 = 1 second). Defines
		  when a packet is considered to be lost so it has to
		  be retransmitted. The default is 5000 = 5 seconds.
		  Lowering this value may make downloads succeed
		  faster in networks with high packet loss rates or
		  with unreliable TFTP servers.

  tftptimeoutcountmax	- maximum count of TFTP timeouts (no
		  unit, minimum value = 0). Defines how many timeouts
		  can happen during a single file transfer before that
		  transfer is aborted. The default is 10, and 0 means
		  'no timeouts allowed'. Increasing this value may help
		  downloads succeed with high packet loss rates, or with
		  unreliable TFTP servers or client hardware.
```

### 8.3 网络启动示例

在配置文件中可以看到网络启动的例子：

```c
"boot_usb_fat = "							\
	"setenv bootargs root=/dev/ram rw "				\
	"console=$consoledev,$baudrate $othbootargs "			\
	"ramdisk_size=$ramdisk_size;"					\
	"usb start;"							\
	"fatload usb 0:2 $loadaddr $bootfile;"				\
	"fatload usb 0:2 $fdtaddr $fdtfile;"				\
	"fatload usb 0:2 $ramdiskaddr $ramdiskfile;"			\
	"bootm $loadaddr $ramdiskaddr $fdtaddr\0 "			\
```

这个例子展示了从 USB 设备启动的完整流程。

## 9. 文件系统支持

### 9.1 支持的文件系统

U-Boot 支持多种文件系统：

```text
/fs			Filesystem code (cramfs, ext2, jffs2, etc.)
```

### 9.2 分区支持

```text
- Partition Labels (disklabels) Supported:
		Zero or more of the following:
		CONFIG_MAC_PARTITION   Apple's MacOS partition table.
		CONFIG_ISO_PARTITION   ISO partition table, used on CDROM etc.
		CONFIG_EFI_PARTITION   GPT partition table, common when EFI is the
				       bootloader.  Note 2TB partition limit; see
				       disk/part_efi.c
		CONFIG_SCSI) you must configure support for at
		least one non-MTD partition type as well.
```

### 9.3 UBIFS 示例

在之前的配置示例中，我们看到了 UBIFS 的使用：

```c
#define CONFIG_BOOTCOMMAND \
	"setenv bootargs ${console} ${mtdparts} ${bootargs_root}; "	\
	"ubi part root; "						\
	"ubifsmount ubi:rootfs; "					\
	"ubifsload 0x800000 ${kernel}; "				\
	"ubifsload 0x700000 ${fdt}; "					\
	"ubifsumount; "							\
	"fdt addr 0x700000; fdt resize; fdt chosen; "			\
	"bootz 0x800000 - 0x700000"
```

## 10. 安全启动

### 10.1 验证启动概述

U-Boot 支持验证启动（Verified Boot），在 `doc/uImage.FIT/verified-boot.txt` 中有详细说明：

```text
U-Boot Verified Boot
====================

Introduction
------------
Verified boot here means the verification of all software loaded into a
machine during the boot process to ensure that it is authorised and correct
for that machine.

Verified boot extends from the moment of system reset to as far as you wish
into the boot process. An example might be loading U-Boot from read-only
memory, then loading a signed kernel, then using the kernel's dm-verity
driver to mount a signed root filesystem.

A key point is that it is possible to field-upgrade the software on machines
which use verified boot. Since the machine will only run software that has
been correctly signed, it is safe to read software from an updatable medium.
It is also possible to add a secondary signed firmware image, in read-write
memory, so that firmware can easily be upgraded in a secure manner.


Signing
-------
Verified boot uses cryptographic algorithms to 'sign' software images.
Images are signed using a private key known only to the signer, but can
be verified using a public key. As its name suggests the public key can be
made available without risk to the verification process. The private and
public keys are mathematically related. For more information on how this
works look up "public key cryptography" and "RSA" (a particular algorithm).
```

### 10.2 签名验证流程

```text
The signing and verification process looks something like this:


      Signing                                      Verification
      =======                                      ============

 +--------------+                   *
 | RSA key pair |                   *             +---------------+
 | .key  .crt   |                   *             | Public key in |
 +--------------+       +------> public key ----->| trusted place |
       |                |           *             +---------------+
       |                |           *                    |
       v                |           *                    v
   +---------+          |           *              +--------------+
   |         |----------+           *              |              |
   | signer  |                      *              |    U-Boot    |
   |         |----------+           *              |  signature   |--> yes/no
   +---------+          |           *              | verification |
      ^                 |           *              |              |
      |                 |           *              +--------------+
      |                 |           *                    ^
      |                 |           *                    |
 +----------+           |           *                    |
 | Software |           +----> signed image -------------+
 |  image   |                       *
 +----------+                       *


The signature algorithm relies only on the public key to do its work. Using
this key it checks the signature that it finds in the image. If it verifies
then we know that the image is OK.

The public key from the signer allows us to verify and therefore trust
software from updatable memory.

It is critical that the public key be secure and cannot be tampered with.
It can be stored in read-only memory, or perhaps protected by other on-chip
crypto provided by some modern SOCs. If the public key can be changed, then
the verification is worthless.
```

### 10.3 BeagleBone 验证启动示例

在 `doc/uImage.FIT/beaglebone_vboot.txt` 中提供了 BeagleBone 的验证启动示例：

```text
Overview
--------

The steps are roughly as follows:

1. Build U-Boot for the board, with the verified boot options enabled.

2. Obtain a suitable Linux kernel

3. Create a Image Tree Source file (ITS) file describing how you want the
kernel to be packaged, compressed and signed.

4. Create a key pair

5. Sign the kernel

6. Put the public key into U-Boot's image

7. Put U-Boot and the kernel onto the board

8. Try it
```

然后是具体的验证过程：

```text
You can also run fit_check_sign to check it:

   $UOUT/tools/fit_check_sign -f image.fit -k am335x-boneblack-pubkey.dtb

which results in:

Verifying Hash Integrity ... sha1,rsa2048:dev+
## Loading kernel from FIT Image at 12345678 ...
   Using 'conf@1' configuration
   Trying 'kernel@1' kernel subimage
     Description:  unavailable
     Created:      Sun Jun  1 12:50:30 2014
     Type:         Kernel Image
     Compression:  gzip compressed
     Data Start:   0x12345678
     Data Size:    4040128 Bytes = 3.9 MB
     Architecture: ARM
     OS:           Linux
     Load Address: 0x80008000
     Entry Point:  0x80008000
     Hash algo:    sha1
     Hash value:   c94364646427e10f423fbd39d531f8e0d5546e90
   Verifying Hash Integrity ... sha1+ OK
   Loading Kernel Image ... OK
## Loading fdt from FIT Image at 12345678 ...
   Using 'conf@1' configuration
   Trying 'fdt@1' fdt subimage
     Description:  unavailable
     Created:      Sun Jun  1 12:50:30 2014
     Type:         Flat Device Tree
     Compression:  uncompressed
     Data Start:   0x12345678
     Data Size:    31547 Bytes = 30.81 kB = 0.03 MB
     Architecture: ARM
     Hash algo:    sha1
     Hash value:   cb09202f889d824f23b8e4404b781be5ad38a68d
   Verifying Hash Integrity ... sha1+ OK
   Loading Device Tree ... OK
```

## 11. 移植和开发

### 11.1 U-Boot 移植指南

README 中提供了一个幽默但实用的移植指南：

```c
int main(int argc, char *argv[])
{
	sighandler_t no_more_time;

	signal(SIGALRM, no_more_time);
	alarm(PROJECT_DEADLINE - toSec (3 * WEEK));

	if (available_money > available_manpower) {
		Pay consultant to port U-Boot;
		return 0;
	}

	Download latest U-Boot source;

	Subscribe to u-boot mailing list;

	if (clueless)
		email("Hi, I am new to U-Boot, how do I get started?");

	while (learning) {
		Read the README file in the top level directory;
		Read http://www.denx.de/twiki/bin/view/DULG/Manual;
		Read applicable doc/*.README;
		Read the source, Luke;
		/* find . -name "*.[chS]" | xargs grep -i <keyword> */
	}

	if (available_money > toLocalCurrency ($2500))
		Buy a BDI3000;
	else
		Add a lot of aggravation and time;

	if (a similar board exists) {	/* hopefully... */
		cp -a board/<similar> board/<myboard>
		cp include/configs/<similar>.h include/configs/<myboard>.h
	} else {
		Create your own board support subdirectory;
		Create your own board include/configs/<myboard>.h file;
	}
	Edit new board/<myboard> files
	Edit new include/configs/<myboard>.h

	while (!accepted) {
		while (!running) {
			do {
				Add / modify source code;
			} until (compiles);
			Debug;
			if (clueless)
				email("Hi, I am having problems...");
		}
		Send patch file to the U-Boot email list;
		if (reasonable critiques)
			Incorporate improvements from email list code review;
		else
			Defend code as written;
	}

	return 0;
}

void no_more_time (int sig)
{
      hire_a_guru();
}
```

### 11.2 编码标准

```text
Coding Standards:
-----------------

All contributions to U-Boot should conform to the Linux kernel
coding style; see the kernel coding style guide at
https://www.kernel.org/doc/html/latest/process/coding-style.html, and the
script "scripts/Lindent" in your Linux kernel source directory.

Source files originating from a different project (for example the
MTD subsystem) are generally exempt from these guidelines and are not
reformatted to ease subsequent migration to newer versions of those
sources.

Please note that U-Boot is implemented in C (and to some small parts in
Assembler); no C++ is used, so please do not use C++ style comments (//)
in your code.

Please also stick to the following formatting rules:
- remove any trailing white space
- use TAB characters for indentation and vertical alignment, not spaces
- make sure NOT to use DOS '\r\n' line feeds
- do not add more than 2 consecutive empty lines to source files
- do not add trailing empty lines to source files
```

### 11.3 板级初始化示例

在 `board/ti/am335x/board.c` 中可以看到具体的板级初始化代码：

```c
#ifdef CONFIG_BOARD_LATE_INIT
int board_late_init(void)
{
#if !defined(CONFIG_SPL_BUILD)
	uint8_t mac_addr[6];
	uint32_t mac_hi, mac_lo;
#endif

#ifdef CONFIG_ENV_VARS_UBOOT_RUNTIME_CONFIG
	char *name = NULL;

	if (board_is_bone_lt()) {
		/* BeagleBoard.org BeagleBone Black Wireless: */
		if (!strncmp(board_ti_get_rev(), "BWA", 3)) {
			name = "BBBW";
		}
		/* SeeedStudio BeagleBone Green Wireless */
		if (!strncmp(board_ti_get_rev(), "GW1", 3)) {
			name = "BBGW";
		}
		/* BeagleBoard.org BeagleBone Blue */
		if (!strncmp(board_ti_get_rev(), "BLA", 3)) {
			name = "BBBL";
		}
	}

	if (board_is_bbg1())
		name = "BBG1";
	set_board_info_env(name);

	/*
	 * Default FIT boot on HS devices. Non FIT images are not allowed
	 * on HS devices.
	 */
	if (get_device_type() == HS_DEVICE)
		env_set("boot_fit", "1");
#endif

	return 0;
}
#endif
```

这个例子展示了如何在后期初始化中：
1. 检测不同的板型
2. 设置相应的环境变量
3. 根据设备类型配置安全启动

## 12. 调试和优化

### 12.1 调试支持

U-Boot 提供了多种调试方法：

```text
- I/O tracing:
		When CONFIG_IO_TRACE is selected, U-Boot intercepts all I/O
		accesses and can checksum them or write a list of them out
		to memory. See the 'iotrace' command for details. This is
		useful for testing device drivers since it can confirm that
		the driver behaves the same way before and after a code
		change. Currently this is supported on sandbox and arm. To
		add support for your architecture, add '#include <iotrace.h>'
		to the bottom of arch/<arch>/include/asm/io.h and test.

		Example output from the 'iotrace stats' command is below.
		Note that if the trace buffer is exhausted, the checksum will
		still continue to operate.

			iotrace is enabled
			Start:  10000000	(buffer start address)
			Size:   00010000	(buffer size)
			Offset: 00000120	(current buffer offset)
			Output: 10000120	(start + offset)
			Count:  00000018	(number of trace records)
			CRC32:  9526fb66	(CRC32 of all trace records)
```

### 12.2 时间戳支持

```text
- Timestamp Support:

		When CONFIG_TIMESTAMP is selected, the timestamp
		(date and time) of an image is printed by image
		commands like bootm or iminfo. This option is
		automatically enabled when you select CONFIG_CMD_DATE .
```

### 12.3 静默控制台

```text
  silent_linux  - If set then Linux will be told to boot silently, by
		  changing the console to be empty. If "yes" it will be
		  made silent. If "no" it will not be made silent. If
		  unset, then it will be made silent if the U-Boot console
		  is silent.
```

在 `common/bootm.c` 中可以看到静默控制台的实现：

```c
#if defined(CONFIG_SILENT_CONSOLE) && !defined(CONFIG_SILENT_U_BOOT_ONLY)
static void fixup_silent_linux(void)
{
	char *buf;
	const char *env_val;
	char *cmdline = env_get("bootargs");
	int want_silent;

	/*
	 * Only fix cmdline when requested. The environment variable
	 * "silent_linux" must not be set to "no".
	 */
	want_silent = env_get_yesno("silent_linux");
	if (want_silent == 0)
		return;
	else if (want_silent == -1 && !(gd->flags & GD_FLG_SILENT))
		return;

	debug("before silent fix-up: %s\n", cmdline);
	if (cmdline && (cmdline[0] != '\0')) {
		char *start = strstr(cmdline, CONSOLE_ARG);

		/* Allocate space for maximum possible new command line */
		buf = malloc(strlen(cmdline) + 1 + strlen(CONSOLE_ARG) + 1);
		if (!buf) {
			debug("%s: out of memory\n", __func__);
			return;
		}

		if (start) {
			char *end = strchr(start, ' ');
			int num_start_bytes = start - cmdline + CONSOLE_ARG_LEN;

			strncpy(buf, cmdline, num_start_bytes);
			if (end)
				strcpy(buf + num_start_bytes, end);
			else
				buf[num_start_bytes] = '\0';
		} else {
			sprintf(buf, "%s %s", cmdline, CONSOLE_ARG);
		}
		env_val = buf;
	} else {
		buf = NULL;
		env_val = CONSOLE_ARG;
	}

	env_set("bootargs", env_val);
	debug("after silent fix-up: %s\n", env_val);
	free(buf);
}
#endif /* CONFIG_SILENT_CONSOLE */
```

## 总结

U-Boot 是一个功能强大且灵活的引导加载程序，支持多种架构和启动方式。通过本文的详细介绍，我们了解了：

1. **架构设计**：模块化的设计使得 U-Boot 能够支持众多硬件平台
2. **启动流程**：从硬件初始化到内核启动的完整过程
3. **配置系统**：基于 Kconfig 的灵活配置机制
4. **设备树支持**：现代 Linux 系统必需的设备树支持
5. **网络功能**：丰富的网络协议支持，便于远程启动和调试
6. **文件系统**：支持多种文件系统，适应不同存储介质
7. **安全启动**：密码学验证确保系统安全
8. **开发支持**：完整的开发和调试工具链

U-Boot 的这些特性使其成为嵌入式系统开发中不可或缺的基础组件。无论是产品开发还是原型验证，理解 U-Boot 的工作原理都对嵌入式系统工程师具有重要意义。

随着嵌入式系统复杂性的增加和安全要求的提高，U-Boot 也在不断演进，加入了更多的现代特性如安全启动、设备树支持、验证启动等。掌握这些技术对于开发可靠、安全的嵌入式产品至关重要。