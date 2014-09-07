---

layout: post
title: "uboot启动linux内核"
date: 2014-2-19 13:34
category: "学习"
comments: false
tags : "uboot"

---
## nand flash的内存规划 ##

nand flash 没有分区表,所以我们必须先弄清楚uboot,linux kernel.yeffs2-file-system存放的地址.
下面是webee210开发板的nand flash 的规划:
![nand-flash](/picture/uboot5.png "nand-flash")

根据上面的内存表,我们可以在uboot的命令行模式下,通过命令测试一下(下面是在webee210进操作):

		/*将uboot烧入nand flash*/                                     
		webee# dnw 0x20000000;                                  
		webee# Now, Downloading [ADDRESS:0x20000000,TOTAL:0x4184c]                                              
			   writing at 0x40000 -- 100% is complete. 268364 bytes written: ok                 
		webee# nand write 0x20000000 0x0 0x4184c                     
		/*将uimage烧入nand flash*/                 
		webee# dnw 0x20000000;                       
		webee# Now, Downloading [ADDRESS:0x20000000,TOTAL:0x2132f0]                    
		webee# nand write 0x20000000 0x100000 0x2132f0                 
		/*将yaffs2烧入nand flash*/                    
		webee# dnw 0x20000000;                       
		webee# Now, Downloading [ADDRESS:0x20000000,TOTAL:0x77c100]                         
		webee# nand write.yaffs 0x20000000 0x77c100                  
		/*设置环境变量*/                                    
		webee# setenv bootargs noinitrd root=/dev/mtdblock2 init=/linuxrc console=ttySAC0,115200 rootfstype=yaffs2 mem=512M                                    
		webee# save                                   
		/*将内核copy到0x20007fc0,因为0x20008000是内核的链接地址,uImage有64bit的头文件*/                         
		webee# nand read 0x20007fc0 0x100000 0x500000                             
		/*启动系统*/                                  
		webee# bootm 0x20007fc0                                                

uboot的菜单烧写与启动,莫过于是上面的命令集封装起来而已.


## bootm的执行过程 ##

{% highlight c %}
/*******************************************************************/
/* bootm - boot application image from image in memory */
/*******************************************************************/

int do_bootm (cmd_tbl_t *cmdtp, int flag, int argc, char * const argv[])
{
	ulong		iflag;
	ulong		load_end = 0;
	int		ret;
	boot_os_fn	*boot_fn;
#ifdef CONFIG_NEEDS_MANUAL_RELOC
	static int relocated = 0;

	/* 重置boot函数列表 */
	if (!relocated) {
		int i;
		for (i = 0; i < ARRAY_SIZE(boot_os); i++)
			if (boot_os[i] != NULL)
				boot_os[i] += gd->reloc_off;
		relocated = 1;
	}
#endif

	/* 提取bootm的参数 */
	if (argc > 1) {
		char *endp;

		simple_strtoul(argv[1], &endp, 16);
		/* endp pointing to NULL means that argv[1] was just a
		 * valid number, pass it along to the normal bootm processing
		 *
		 * If endp is ':' or '#' assume a FIT identifier so pass
		 * along for normal processing.
		 *
		 * Right now we assume the first arg should never be '-'
		 */
		if ((*endp != 0) && (*endp != ':') && (*endp != '#'))
			return do_bootm_subcommand(cmdtp, flag, argc, argv);
	}
	/*验证uImage的格式*/
	if (bootm_start(cmdtp, flag, argc, argv))
		return 1;

	/*
	 * We have reached the point of no return: we are going to
	 * overwrite all exception vector code, so we cannot easily
	 * recover from any failures any more...
	 */
	iflag = disable_interrupts();

#if defined(CONFIG_CMD_USB)
	/*
	 * turn off USB to prevent the host controller from writing to the
	 * SDRAM while Linux is booting. This could happen (at least for OHCI
	 * controller), because the HCCA (Host Controller Communication Area)
	 * lies within the SDRAM and the host controller writes continously to
	 * this area (as busmaster!). The HccaFrameNumber is for example
	 * updated every 1 ms within the HCCA structure in SDRAM! For more
	 * details see the OpenHCI specification.
	 */
	usb_stop();
#endif
	/*加载uImage.img,对其解压*/
	ret = bootm_load_os(images.os, &load_end, 1);

{% endhighlight c %}

images 为下面的结构体.而os为image的信息.
{% highlight c %}
/*
 * Legacy and FIT format headers used by do_bootm() and do_bootm_<os>()
 * routines.
 */
typedef struct bootm_headers {
	/*
	 * Legacy os image header, if it is a multi component image
	 * then boot_get_ramdisk() and get_fdt() will attempt to get
	 * data from second and third component accordingly.
	 */
	image_header_t	*legacy_hdr_os;		/* image header pointer */
	image_header_t	legacy_hdr_os_copy;	/* header copy */
	ulong		legacy_hdr_valid;

#if defined(CONFIG_FIT)
	const char	*fit_uname_cfg;	/* configuration node unit name */

	void		*fit_hdr_os;	/* os FIT image header */
	const char	*fit_uname_os;	/* os subimage node unit name */
	int		fit_noffset_os;	/* os subimage node offset */

	void		*fit_hdr_rd;	/* init ramdisk FIT image header */
	const char	*fit_uname_rd;	/* init ramdisk subimage node unit name */
	int		fit_noffset_rd;	/* init ramdisk subimage node offset */

	void		*fit_hdr_fdt;	/* FDT blob FIT image header */
	const char	*fit_uname_fdt;	/* FDT blob subimage node unit name */
	int		fit_noffset_fdt;/* FDT blob subimage node offset */
#endif

#ifndef USE_HOSTCC
	image_info_t	os;		/* os image info */
	ulong		ep;		/* entry point of OS */

	ulong		rd_start, rd_end;/* ramdisk start/end */

#ifdef CONFIG_OF_LIBFDT
	char		*ft_addr;	/* flat dev tree address */
#endif
	ulong		ft_len;		/* length of flat device tree */

	ulong		initrd_start;
	ulong		initrd_end;
	ulong		cmdline_start;
	ulong		cmdline_end;
	bd_t		*kbd;
#endif

	int		verify;		/* getenv("verify")[0] != 'n' */

#define	BOOTM_STATE_START	(0x00000001)
#define	BOOTM_STATE_LOADOS	(0x00000002)
#define	BOOTM_STATE_RAMDISK	(0x00000004)
#define	BOOTM_STATE_FDT		(0x00000008)
#define	BOOTM_STATE_OS_CMDLINE	(0x00000010)
#define	BOOTM_STATE_OS_BD_T	(0x00000020)
#define	BOOTM_STATE_OS_PREP	(0x00000040)
#define	BOOTM_STATE_OS_GO	(0x00000080)
	int		state;

#ifdef CONFIG_LMB
	struct lmb	lmb;		/* for memory mgmt */
#endif
} bootm_headers_t;
{% endhighlight c %}

image_info_t 的定义如下:
{% highlight c %}
typedef struct image_info {
	ulong		start, end;		/* start/end of blob */
	ulong		image_start, image_len; /* start of image within blob, len of image */
	ulong		load;			/* load addr for the image */
	uint8_t		comp, type, os;		/* compression, type of image, os type */
} image_info_t;
{% endhighlight c %}

回头看看bootm_load_os函数的作用:

{% highlight c %}
static int bootm_load_os(image_info_t os, ulong *load_end, int boot_progress)
{
	uint8_t comp = os.comp;       //压缩类型
	ulong load = os.load;         //image的加载地址
	ulong blob_start = os.start;  //image的起始地址
	ulong blob_end = os.end;
	ulong image_start = os.image_start; 
	ulong image_len = os.image_len;
	uint unc_len = CONFIG_SYS_BOOTM_LEN;
#if defined(CONFIG_LZMA) || defined(CONFIG_LZO)
	int ret;
#endif /* defined(CONFIG_LZMA) || defined(CONFIG_LZO) */

	const char *type_name = genimg_get_type_name (os.type);

	/*查找image的压缩类型,并进行解压*/
	switch (comp) {
	case IH_COMP_NONE:
		if (load == blob_start || load == image_start) {
			/*如果load address=真正的内核地址*/
			printf ("   XIP %s ... ", type_name);
		} else {
			printf ("   Loading %s ... ", type_name);
			memmove_wd ((void *)load, (void *)image_start,
					image_len, CHUNKSZ);
		}
		*load_end = load + image_len;
		puts("OK\n");
		break;
#ifdef CONFIG_GZIP
	case IH_COMP_GZIP:
		printf ("   Uncompressing %s ... ", type_name);
		if (gunzip ((void *)load, unc_len,
					(uchar *)image_start, &image_len) != 0) {
			puts ("GUNZIP: uncompress, out-of-mem or overwrite error "
				"- must RESET board to recover\n");
			if (boot_progress)
				show_boot_progress (-6);
			return BOOTM_ERR_RESET;
		}

		*load_end = load + image_len;
		break;
#endif /* CONFIG_GZIP */
#ifdef CONFIG_BZIP2
	case IH_COMP_BZIP2:
		printf ("   Uncompressing %s ... ", type_name);
		/*
		 * If we've got less than 4 MB of malloc() space,
		 * use slower decompression algorithm which requires
		 * at most 2300 KB of memory.
		 */
		int i = BZ2_bzBuffToBuffDecompress ((char*)load,
					&unc_len, (char *)image_start, image_len,
					CONFIG_SYS_MALLOC_LEN < (4096 * 1024), 0);
		if (i != BZ_OK) {
			printf ("BUNZIP2: uncompress or overwrite error %d "
				"- must RESET board to recover\n", i);
			if (boot_progress)
				show_boot_progress (-6);
			return BOOTM_ERR_RESET;
		}

		*load_end = load + unc_len;
		break;
#endif /* CONFIG_BZIP2 */
#ifdef CONFIG_LZMA
	case IH_COMP_LZMA: {
		SizeT lzma_len = unc_len;
		printf ("   Uncompressing %s ... ", type_name);

		ret = lzmaBuffToBuffDecompress(
			(unsigned char *)load, &lzma_len,
			(unsigned char *)image_start, image_len);
		unc_len = lzma_len;
		if (ret != SZ_OK) {
			printf ("LZMA: uncompress or overwrite error %d "
				"- must RESET board to recover\n", ret);
			show_boot_progress (-6);
			return BOOTM_ERR_RESET;
		}
		*load_end = load + unc_len;
		break;
	}
#endif /* CONFIG_LZMA */
#ifdef CONFIG_LZO
	case IH_COMP_LZO:
		printf ("   Uncompressing %s ... ", type_name);

		ret = lzop_decompress((const unsigned char *)image_start,
					  image_len, (unsigned char *)load,
					  &unc_len);
		if (ret != LZO_E_OK) {
			printf ("LZO: uncompress or overwrite error %d "
			      "- must RESET board to recover\n", ret);
			if (boot_progress)
				show_boot_progress (-6);
			return BOOTM_ERR_RESET;
		}

		*load_end = load + unc_len;
		break;
#endif /* CONFIG_LZO */
	default:
		printf ("Unimplemented compression type %d\n", comp);
		return BOOTM_ERR_UNIMPLEMENTED;
	}

	flush_cache(load, (*load_end - load) * sizeof(ulong));

	puts ("OK\n");
	debug ("   kernel loaded at 0x%08lx, end = 0x%08lx\n", load, *load_end);
	if (boot_progress)
		show_boot_progress (7);

	if ((load < blob_end) && (*load_end > blob_start)) {
		debug ("images.os.start = 0x%lX, images.os.end = 0x%lx\n", blob_start, blob_end);
		debug ("images.os.load = 0x%lx, load_end = 0x%lx\n", load, *load_end);

		return BOOTM_ERR_OVERLAP;
	}

	return 0;
}
{% endhighlight c %}

bootm_load_os()镜像进行解压.接着回到cmd_bootm.c中的do_bootm():

{% highlight c %}
show_boot_progress (8);
#ifdef CONFIG_SILENT_CONSOLE
	if (images.os.os == IH_OS_LINUX)
		fixup_silent_linux();
#endif
	/*启动系统*/
	boot_fn = boot_os[images.os.os];

{% endhighlight c %}

{% highlight c %}
boot_os的定义如下,他是一个列表,

static boot_os_fn *boot_os[] = {
#ifdef CONFIG_BOOTM_LINUX
	[IH_OS_LINUX] = do_bootm_linux,
#endif
#ifdef CONFIG_BOOTM_NETBSD
	[IH_OS_NETBSD] = do_bootm_netbsd,
#endif
#ifdef CONFIG_LYNXKDI
	[IH_OS_LYNXOS] = do_bootm_lynxkdi,
#endif
#ifdef CONFIG_BOOTM_RTEMS
	[IH_OS_RTEMS] = do_bootm_rtems,
#endif
#if defined(CONFIG_BOOTM_OSE)
	[IH_OS_OSE] = do_bootm_ose,
#endif
#if defined(CONFIG_CMD_ELF)
	[IH_OS_VXWORKS] = do_bootm_vxworks,
	[IH_OS_QNX] = do_bootm_qnxelf,
#endif
#ifdef CONFIG_INTEGRITY
	[IH_OS_INTEGRITY] = do_bootm_integrity,
#endif
};
{% endhighlight c %}

来看do_bootm_linux()函数,我们留意传递给内核的参数:

{% highlight c %}
DECLARE_GLOBAL_DATA_PTR;

int do_bootm_linux(int flag, int argc, char * const argv[], bootm_headers_t *images)
{
	/* First parameter is mapped to $r5 for kernel boot args */
	void	(*theKernel) (char *, ulong, ulong);
	char	*commandline = getenv ("bootargs");     //传递给内核的启动参数
	ulong	rd_data_start, rd_data_end;

	if ((flag != 0) && (flag != BOOTM_STATE_OS_GO))
		return 1;

	int	ret;

	char	*of_flat_tree = NULL;
#if defined(CONFIG_OF_LIBFDT)
	/* did generic code already find a device tree? */
	if (images->ft_len)
		of_flat_tree = images->ft_addr;
#endif

	/*内核的入口指针*/
	theKernel = (void (*)(char *, ulong, ulong))images->ep;

	/* find ramdisk */
	ret = boot_get_ramdisk (argc, argv, images, IH_ARCH_MICROBLAZE,
			&rd_data_start, &rd_data_end);
	if (ret)
		return 1;

	show_boot_progress (15);

	if (!of_flat_tree && argc > 3)
		of_flat_tree = (char *)simple_strtoul(argv[3], NULL, 16);
#ifdef DEBUG
	printf ("## Transferring control to Linux (at address 0x%08lx) " \
				"ramdisk 0x%08lx, FDT 0x%08lx...\n",
		(ulong) theKernel, rd_data_start, (ulong) of_flat_tree);
#endif

#ifdef XILINX_USE_DCACHE
#ifdef XILINX_DCACHE_BYTE_SIZE
	flush_cache(0, XILINX_DCACHE_BYTE_SIZE);
#else
#warning please rebuild BSPs and update configuration
	flush_cache(0, 32768);
#endif
#endif
	/*
	 * Linux Kernel Parameters (passing device tree):
	 * r5: pointer to command line
	 * r6: pointer to ramdisk
	 * r7: pointer to the fdt, followed by the board info data
	 */
	theKernel (commandline, rd_data_start, (ulong) of_flat_tree);
	/* does not return */

	return 1;
}
{% endhighlight c %}
commandline = getenv ("bootargs") "bootargs"可以通过在include/configs/webee210.h里用宏定义#define CONFIG_BOOTARGS


