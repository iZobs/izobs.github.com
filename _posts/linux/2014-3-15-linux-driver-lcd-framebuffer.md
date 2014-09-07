---

layout: post
title: "linux LCD 驱动之linux framebuffer"
date: 2014-3-25 19:34
category: "linux"
comments: false
tags : "linux driver"

---
##1.Linux framebuffer简介

帧缓冲（framebuffer）是Linux为显示设备提供的一个接口，把显存抽象后的一种设备，他允许上层应用程序在图形模式下直接对显示缓冲区进行读写操作。framebuffer是LCD对应的一中HAL（硬件抽象层），提供抽象的，统一的接口操作，用户不必关心硬件层是怎么实施的。这些都是由Framebuffer设备驱动来完成的.                                     
帧缓冲设备对应的设备文件为/dev/fb*，如果系统有多个显示卡，Linux下还可支持多个帧缓冲设备，最多可达32个，分别为/dev/fb0到 /dev/fb31，而/dev/fb则为当前缺省的帧缓冲设备，通常指向/dev/fb0，在嵌入式系统中支持一个显示设备就够了。帧缓冲设备为标准字符设备，主设备号为29，次设备号则从0到31。分别对应/dev/fb0-/dev/fb31。             

通过/dev/fb，应用程序的操作主要有这几种：                  

- 读/写（read/write）/dev/fb：相当于读/写屏幕缓冲区。                
- 映射（map）操作：由于Linux工作在保护模式，每个应用程序都有自己的虚拟地址空间，在应用程序中是不能直接访问物理缓冲区地址的。而帧缓冲设备可以通过mmap()映射操作将屏幕缓冲区的物理地址映射到用户空间的一段虚拟地址上，然后用户就可以通过读写这段虚拟地址访问屏幕缓冲区，在屏幕上绘图了。 
- I/O控制：对于帧缓冲设备，对设备文件的ioctl操作可读取/设置显示设备及屏幕的参数，如分辨率，屏幕大小等相关参数。ioctl的操作是由底层的驱动程序来完成的


##1.Linux frambuffer的数据结构

在`linux/fd.h`目录下定义了linux framebuffer的数据结构体。

#(1).struct fb_info

fb_info 结构体是linux为帧缓冲设备定义的驱动层接口，用于用户在内核空间的调用。它不仅包含了低层函数，而且还有记录设备状态的数据。每个帧缓冲设备都有一个fb_info结构相对应。

<pre class="prettyprint" id="c">

struct fb_info {
	atomic_t count;
	int node;
	int flags;
	struct mutex lock;		/* Lock for open/release/ioctl funcs */
	struct mutex mm_lock;		/* Lock for fb_mmap and smem_* fields */
	struct fb_var_screeninfo var;	/* 当前可变的参数 */
	struct fb_fix_screeninfo fix;	/* 当前固定的参数 */
	struct fb_monspecs monspecs;	/* 描述当前显示器的特有固定信息*/
	struct work_struct queue;	/* Framebuffer event queue */
	struct fb_pixmap pixmap;	/* Image hardware mapper */
	struct fb_pixmap sprite;	/* Cursor hardware mapper */
	struct fb_cmap cmap;		/* 当前颜色映射表 */
	struct list_head modelist;      /* mode list */
	struct fb_videomode *mode;	/* current mode */

#ifdef CONFIG_FB_BACKLIGHT
	/* assigned backlight device */
	/* set before framebuffer registration, 
	   remove after unregister */
	struct backlight_device *bl_dev;

	/* Backlight level curve */
	struct mutex bl_curve_mutex;	
	u8 bl_curve[FB_BACKLIGHT_LEVELS];
#endif
#ifdef CONFIG_FB_DEFERRED_IO
	struct delayed_work deferred_work;
	struct fb_deferred_io *fbdefio;
#endif

	struct fb_ops *fbops;        /*指向当前显示器的特有固定的信息,可以通过ioctl()系统调用操作设备*/
	struct device *device;		/* This is the parent */
	struct device *dev;		/* This is this fb device */
	int class_flag;                    /* private sysfs flags */
#ifdef CONFIG_FB_TILEBLITTING
	struct fb_tile_ops *tileops;    /* Tile Blitting */
#endif
	char __iomem *screen_base;	/* Virtual address */
	unsigned long screen_size;	/* Amount of ioremapped VRAM or 0 */ 
	void *pseudo_palette;		/* Fake palette of 16 colors */ 
#define FBINFO_STATE_RUNNING	0
#define FBINFO_STATE_SUSPENDED	1
	u32 state;			/* Hardware state i.e suspend */
	void *fbcon_par;                /* fbcon use-only private area */
	/* From here on everything is device dependent */
	void *par;
	/* we need the PCI or similar aperture base/size not
	   smem_start/size as smem_start may just be an object
	   allocated inside the aperture so may not actually overlap */
	struct apertures_struct {
		unsigned int count;
		struct aperture {
			resource_size_t base;
			resource_size_t size;
		} ranges[0];
	} *apertures;
};

</pre>

`struct fb_fix_screeninfo` 这个结构体用来描述设备无关，不可变更的信息。用户可以使用FBIOGET_FSCREENINFO命令来获得这些信息。

<pre class="prettyprint" id="c">
struct fb_fix_screeninfo {
	char id[16];			/* identification string eg "TT Builtin" */
	unsigned long smem_start;	/* 描述缓冲起始地址(物理地址) */
	__u32 smem_len;			/*描述缓冲区长度 */
	__u32 type;			/* 描述fb类型，比如TFT或STN类型*		*/
	__u32 type_aux;			/* Interleave for interleaved Planes */
	__u32 visual;			/*描述显示颜色是真彩色，伪彩色,还是单色 	*/ 
	__u16 xpanstep;			/* zero if no hardware panning  */
	__u16 ypanstep;			/* zero if no hardware panning  */
	__u16 ywrapstep;		/* zero if no hardware ywrap    */
	__u32 line_length;		/* length of a line in bytes    */
	unsigned long mmio_start;	/* I/O内存映射起始地址,物理地址 */
	__u32 mmio_len;			/* Length of Memory Mapped I/O  */
	__u32 accel;			/* Indicate to driver which	*/
					/*  specific chip/card we have	*/
	__u16 capabilities;		/* see FB_CAP_*			*/
	__u16 reserved[2];		/* Reserved for future compatibility */
};

</pre>

`struct fb_var_screeninfo`该结构描述设备无关的，可以更改的配置信息。应用程序可以使用FBIOGET_VSCREENINFO命令来获得这些信息，使用FBIOPUT_VSCREENIFO命令写入这些信息


<pre class="prettyprint" id="c">
struct fb_var_screeninfo {
	__u32 xres;			/* visible resolution		*/
	__u32 yres;
	__u32 xres_virtual;		/* virtual resolution		*/
	__u32 yres_virtual;
	__u32 xoffset;			/* offset from virtual to visible */
	__u32 yoffset;			/* resolution			*/

	__u32 bits_per_pixel;		/* guess what			*/
	__u32 grayscale;		/* 0 = color, 1 = grayscale,	*/
					/* >1 = FOURCC			*/
	struct fb_bitfield red;		/* bitfield in fb mem if true color, */
	struct fb_bitfield green;	/* else only length is significant */
	struct fb_bitfield blue;
	struct fb_bitfield transp;	/* transparency			*/	

	__u32 nonstd;			/* != 0 Non standard pixel format */

	__u32 activate;			/* see FB_ACTIVATE_*		*/

	__u32 height;			/* height of picture in mm    */
	__u32 width;			/* width of picture in mm     */

	__u32 accel_flags;		/* (OBSOLETE) see fb_info.flags */

	/* Timing: All values in pixclocks, except pixclock (of course) */
	__u32 pixclock;			/* pixel clock in ps (pico seconds) */
	__u32 left_margin;		/* time from sync to picture	*/
	__u32 right_margin;		/* time from picture to sync	*/
	__u32 upper_margin;		/* time from sync to picture	*/
	__u32 lower_margin;
	__u32 hsync_len;		/* length of horizontal sync	*/
	__u32 vsync_len;		/* length of vertical sync	*/
	__u32 sync;			/* see FB_SYNC_*		*/
	__u32 vmode;			/* see FB_VMODE_*		*/
	__u32 rotate;			/* angle we rotate counter clockwise */
	__u32 colorspace;		/* colorspace for FOURCC-based modes */
	__u32 reserved[4];		/* Reserved for future compatibility */
};

</pre>

#(2).struct fb_ops

对帧缓冲设备的操作在linux内核中是由fb_ops结构体定义，应用程序可以通过调用ioctl系统来操作设备，该结构就是用来支持ioctl的这些操作的.帧缓冲设备的驱动主要就是编写这些接口函数.


<pre class="prettyprint" id="c">
/*
 * Frame buffer operations
 *
 * LOCKING NOTE: those functions must _ALL_ be called with the console
 * semaphore held, this is the only suitable locking mechanism we have
 * in 2.6. Some may be called at interrupt time at this point though.
 *
 * The exception to this is the debug related hooks.  Putting the fb
 * into a debug state (e.g. flipping to the kernel console) and restoring
 * it must be done in a lock-free manner, so low level drivers should
 * keep track of the initial console (if applicable) and may need to
 * perform direct, unlocked hardware writes in these hooks.
 */

struct fb_ops {
	/* open/release and usage marking */
	struct module *owner;
	int (*fb_open)(struct fb_info *info, int user);
	int (*fb_release)(struct fb_info *info, int user);

	/* For framebuffers with strange non linear layouts or that do not
	 * work with normal memory mapped access
	 */
	ssize_t (*fb_read)(struct fb_info *info, char __user *buf,
			   size_t count, loff_t *ppos);
	ssize_t (*fb_write)(struct fb_info *info, const char __user *buf,
			    size_t count, loff_t *ppos);

	/* checks var and eventually tweaks it to something supported,
	 * DO NOT MODIFY PAR */
	int (*fb_check_var)(struct fb_var_screeninfo *var, struct fb_info *info);

	/* set the video mode according to info->var */
	int (*fb_set_par)(struct fb_info *info);

	/* set color register */
	int (*fb_setcolreg)(unsigned regno, unsigned red, unsigned green,
			    unsigned blue, unsigned transp, struct fb_info *info);

	/* set color registers in batch */
	int (*fb_setcmap)(struct fb_cmap *cmap, struct fb_info *info);

	/* blank display */
	int (*fb_blank)(int blank, struct fb_info *info);

	/* pan display */
	int (*fb_pan_display)(struct fb_var_screeninfo *var, struct fb_info *info);

	/* Draws a rectangle */
	void (*fb_fillrect) (struct fb_info *info, const struct fb_fillrect *rect);
	/* Copy data from area to another */
	void (*fb_copyarea) (struct fb_info *info, const struct fb_copyarea *region);
	/* Draws a image to the display */
	void (*fb_imageblit) (struct fb_info *info, const struct fb_image *image);

	/* Draws cursor */
	int (*fb_cursor) (struct fb_info *info, struct fb_cursor *cursor);

	/* Rotates the display */
	void (*fb_rotate)(struct fb_info *info, int angle);

	/* wait for blit idle, optional */
	int (*fb_sync)(struct fb_info *info);

	/* perform fb specific ioctl (optional) */
	int (*fb_ioctl)(struct fb_info *info, unsigned int cmd,
			unsigned long arg);

	/* Handle 32bit compat ioctl (optional) */
	int (*fb_compat_ioctl)(struct fb_info *info, unsigned cmd,
			unsigned long arg);

	/* perform fb specific mmap */
	int (*fb_mmap)(struct fb_info *info, struct vm_area_struct *vma);

	/* get capability given var */
	void (*fb_get_caps)(struct fb_info *info, struct fb_blit_caps *caps,
			    struct fb_var_screeninfo *var);

	/* teardown any resources to do with this framebuffer */
	void (*fb_destroy)(struct fb_info *info);

	/* called at KDB enter and leave time to prepare the console */
	int (*fb_debug_enter)(struct fb_info *info);
	int (*fb_debug_leave)(struct fb_info *info);
};
</pre>
