---

layout: post
title: "给uboot添加命令和命令菜单"
date: 2014-2-8 15:34
category: "学习"
comments: false
tags : "uboot"

---


# 一、uboot命令添加过程 #

##$U-boot命令的解析##

在uboot main_loop中执行一个命令,将调用函数 run_command()执行命令

{% highlight c %}

int run_command (const char *cmd, int flag)
{
	cmd_tbl_t *cmdtp;
	char cmdbuf[CONFIG_SYS_CBSIZE];	/* working copy of cmd		*/
	char *token;			/* start of token in cmdbuf	*/
	char *sep;			/* end of token (separator) in cmdbuf */
	char finaltoken[CONFIG_SYS_CBSIZE];
	char *str = cmdbuf;
	char *argv[CONFIG_SYS_MAXARGS + 1];	/* NULL terminated	*/
	int argc, inquotes;
	int repeatable = 1;
	int rc = 0;

#ifdef DEBUG_PARSER
	printf ("[RUN_COMMAND] cmd[%p]=\"", cmd);
	puts (cmd ? cmd : "NULL");	/* use puts - string may be loooong */
	puts ("\"\n");
#endif

	clear_ctrlc();		/* forget any previous Control C */

	if (!cmd || !*cmd) {
		return -1;	/* empty command */
	}

	if (strlen(cmd) >= CONFIG_SYS_CBSIZE) {
		puts ("## Command too long!\n");
		return -1;
	}

	strcpy (cmdbuf, cmd);

	/* Process separators and check for invalid
	 * repeatable commands
	 */

#ifdef DEBUG_PARSER
	printf ("[PROCESS_SEPARATORS] %s\n", cmd);
#endif
	while (*str) {

		/*
		 * Find separator, or string end
		 * Allow simple escape of ';' by writing "\;"
		 */
		for (inquotes = 0, sep = str; *sep; sep++) {
			if ((*sep=='\'') &&
			    (*(sep-1) != '\\'))
				inquotes=!inquotes;

			if (!inquotes &&
			    (*sep == ';') &&	/* ;是两个命令的分隔		*/
			    ( sep != str) &&	/* past string start	*/
			    (*(sep-1) != '\\'))	/* and NOT escaped	*/
				break;
		}

		/*
		 * Limit the token to data between separators
		 */
		token = str;
		if (*sep) {
			str = sep + 1;	/* start of command for next pass */
			*sep = '\0';
		}
		else
			str = sep;	/* no more commands for next pass */
#ifdef DEBUG_PARSER
		printf ("token: \"%s\"\n", token);
#endif

		/* find macros in this token and replace them */
		process_macros (token, finaltoken); //处理环境宏

		/* 解析命令的参数 */
		if ((argc = parse_line (finaltoken, argv)) == 0) {
			rc = -1;	/* no command at all */
			continue;
		}

		/* Look up command in command table */
		/*argv[0]是命令的名字*/
		if ((cmdtp = find_cmd(argv[0])) == NULL) {
			printf ("Unknown command '%s' - try 'help'\n", argv[0]);
			rc = -1;	/* give up after bad command */
			continue;
		}

		/* found - check max args */
		if (argc > cmdtp->maxargs) {
			cmd_usage(cmdtp);
			rc = -1;
			continue;
		}

#if defined(CONFIG_CMD_BOOTD)
		/* avoid "bootd" recursion */
		if (cmdtp->cmd == do_bootd) {
#ifdef DEBUG_PARSER
			printf ("[%s]\n", finaltoken);
#endif
			if (flag & CMD_FLAG_BOOTD) {
				puts ("'bootd' recursion detected\n");
				rc = -1;
				continue;
			} else {
				flag |= CMD_FLAG_BOOTD;
			}
		}
#endif

		/* OK - call function to do the command */
		/*调用命令对应的函数*/
		if ((cmdtp->cmd) (cmdtp, flag, argc, argv) != 0) {
			rc = -1;
		}

		repeatable &= cmdtp->repeatable;

		/* Did the user stop this? */
		if (had_ctrlc ())
			return -1;	/* if stopped then not repeatable */
	}

	return rc ? rc : repeatable;
}
{% endhighlight c %}




这段代码的作用是匹配是否存在我们输入的命令(argv[0])

{% highlight c %}

/* Look up command in command table */
		/*argv[0]是命令的名字*/
		if ((cmdtp = find_cmd(argv[0])) == NULL) {
			printf ("Unknown command '%s' - try 'help'\n", argv[0]);
			rc = -1;	/* give up after bad command */
			continue;
		}

{% endhighlight c %}

find_cmd()函数定义如下:

{% highlight c %}
cmd_tbl_t *find_cmd (const char *cmd)
{
	int len = &__u_boot_cmd_end - &__u_boot_cmd_start;
	return find_cmd_tbl(cmd, &__u_boot_cmd_start, len);
}


cmd_tbl_t *find_cmd_tbl (const char *cmd, cmd_tbl_t *table, int table_len)
{
	cmd_tbl_t *cmdtp;
	cmd_tbl_t *cmdtp_temp = table;	/*Init value */
	const char *p;
	int len;
	int n_found = 0;

	if (!cmd)
		return NULL;
	/*
	 * Some commands allow length modifiers (like "cp.b");
	 * compare command name only until first dot.
	 */
	len = ((p = strchr(cmd, '.')) == NULL) ? strlen (cmd) : (p - cmd);

	for (cmdtp = table;
	     cmdtp != table + table_len;
	     cmdtp++) {
		if (strncmp (cmd, cmdtp->name, len) == 0) {
			if (len == strlen (cmdtp->name))
				return cmdtp;	/* full match */

			cmdtp_temp = cmdtp;	/* abbreviated command ? */
			n_found++;
		}
	}
	if (n_found == 1) {			/* exactly one match */
		return cmdtp_temp;
	}

	return NULL;	/* not found or ambiguous command */
}

{% endhighlight c %}


	__u_boot_cmd_start与__u_boot_cmd_end 位于u-boot.lds中:

		 __u_boot_cmd_start = .;
		 .u_boot_cmd : { *(.u_boot_cmd) }
		 __u_boot_cmd_end = .;

find_cmd()做的其实就是遍历u-boot.lds,从 .u_boot_cmd段中找出是否存在argv[0]这个命令.

而我们的命名是如何存入.u_boot_cmd这个段的呢?其实他是通过下面的过程.

U-Boot的每一个命令都是通过U_BOOT_CMD宏定义的。这个宏在include/command.h头文件中定义，每一个命令定义一个cmd_tbl_t结构体。

{% highlight c %}
/*命令宏U_BOOT_CMD*/  
#define U_BOOT_CMD(name,maxargs,rep,cmd,usage,help) \  
cmd_tbl_t   __u_boot_cmd_##name     Struct_Section = {#name, maxargs, rep, cmd, usage, help}  
{% endhighlight c %}

1.cmd_tbl_t 命令相关的结构体,定义如下:

		typedef struct cmd_tbl_s	cmd_tbl_t;

		struct cmd_tbl_s {
			char		*name;		/* Command Name			*/
			int		maxargs;	/* maximum number of arguments	*/
			int		repeatable;	/* autorepeat allowed?		*/
							/* Implementation function	*/
			int		(*cmd)(struct cmd_tbl_s *, int, int, char * const []);
			char		*usage;		/* Usage message	(short)	*/
		#ifdef	CONFIG_SYS_LONGHELP
			char		*help;		/* Help  message	(long)	*/
		#endif
		#ifdef CONFIG_AUTO_COMPLETE
			/* do auto completion on the arguments */
			int		(*complete)(int argc, char * const argv[], char last_char, int maxv, char *cmdv[]);
		#endif
		};


2.##name = #name :#是字符串操作符,_##为字符串连接符

3.`Struct_Section` 这是定义一个结构的属性，将其放在.u_boot_cmd这个段当中(`common.h`)，相当于.data/.bss这些段 

		#define Struct_Section  __attribute__ ((unused,section (".u_boot_cmd"))) 

4.cmd_tbl_t 这个结构拥有一个属性,即Struct_section.这个的作用是所有的命令将被集中到.u_boot_cmd这个段中.[位于`uboot.lds`]

##$实践-添加一个U-boot命令##
在common目录下`touch cmd_hello.c`, `vim cmd_hello.c`

{% highlight c %}
#include <common.h>
#include <command.h>

int do_hello(cmd_tbl_t * cmdtp, int flag, int argc, char * const argv[])
{
	int i;
	printf("hello world!\n");
	for(i = 0;i<argc;i++)
	{
		printf("the argv[%d] is :%s\n",i, argv[i]);
	}
	return 0;

}

U_BOOT_CMD(
	hello,	CONFIG_SYS_MAXARGS,	1,	do_hello,
	"hello,a command test",
	"print detailed usage of 'hello'"
);

{% endhighlight c %}

打开common目录下的Makefile,`vim Makefile`,加下面句子:

		 COBJS-y += image.o
		 COBJS-y += s_record.o
		 COBJS-$(CONFIG_SERIAL_MULTI) += serial.o
		 COBJS-y += xyzModem.o
	    +COBJS-y += cmd_hello.o

cd 到uboot根目录重新编译就ok了!



# 二、uboot命令添加menu#

uboot menu 可以当成一个普通命令文件.而在其中调用了许多命令集.


{% highlight c %}
#include <common.h>
#include <command.h>
#include <nand.h>

#ifdef CONFIG_CMD_MENU



extern char console_buffer[];
extern int readline (const char *const prompt);



#define USE_USB_DOWN		1


void main_menu_usage(char menu_type)
{

	printf("\r\n#####	 Boot for Smart210 Main Menu	#####\r\n");

	if( menu_type == USE_USB_DOWN)
	{
		printf("#####    Smart210 USB download mode     #####\r\n\n");
	}
	printf("[1] Download U-boot to Nand Flash\r\n");
	printf("[2] Download Linux Kernel (uImage.bin) to Nand Flash\r\n");
	printf("[3] Download YAFFS image (root.bin) to Nand Flash\r\n");
	printf("[4] Download Program to SDRAM and Run it\r\n");
	printf("[5] Boot the system\r\n");
	printf("[6] Format the Nand Flash\r\n");
	printf("[q] Return main Menu \r\n");
	printf("Enter your selection: ");
}






void menu_shell(void)
{
	char keyselect;
	char cmd_buf[200];

	while (1)
	{
		main_menu_usage(USE_USB_DOWN);
		keyselect = getc();
		printf("%c\n", keyselect);
		switch (keyselect)
		{
			case '1':
			{
				printf("\n");
				printf("Download the uboot into Nand flash by DNW\n");
				strcpy(cmd_buf, "dnw 0x20000000; nand erase 0x0 0x80000; nand write 0x20000000 0x0 0x80000");

				run_command(cmd_buf, 0);
				break;
			}
			
			

			case '2':
			{
				printf("\n");
				printf("Download the kernel into Nand flash by DNW\n");
				strcpy(cmd_buf, "dnw 0x20000000; nand erase 0x300000 0x500000; nand write 0x20000000 0x300000 0x500000");

				run_command(cmd_buf, 0);
				break;
			}


			case '3':
			{
//#ifdef CONFIG_MTD_DEVICE
//				strcpy(cmd_buf, "dnw 0x20000000; nand erase root; nand write.yaffs 0x20000000 root $(filesize)");
//#else
				strcpy(cmd_buf, "dnw 0x20000000; nand erase 0xe00000 0xF8D0000; nand write.yaffs 0x20000000 0xe00000 $(filesize)");
//#endif /* CONFIG_MTD_DEVICE */
				run_command(cmd_buf, 0);
				break;
			}

			case '4':
			{
				char addr_buff[12];
				printf("Enter download address:(eg: 0x20000000)\n");
				readline(NULL);
				strcpy(addr_buff,console_buffer);
				sprintf(cmd_buf, "dnw %s;go %s", addr_buff, addr_buff);
				run_command(cmd_buf, 0);
				break;
			}

			case '5':
			{

				printf("Start Linux ...\n");

				printf("Boot the linux (YAFFS2)\n");
				strcpy(cmd_buf, "setenv bootargs noinitrd root=/dev/mtdblock2 init=/linuxrc console=ttySAC0,115200 rootfstype=yaffs2 mem=512M");
				run_command(cmd_buf, 0);

				strcpy(cmd_buf, "setenv bootcmd 'nand read 0x21000000 0x300000 0x500000;bootm 0x21000000';save");

				run_command(cmd_buf, 0);
				break;
			}

			case '6':
			{
				strcpy(cmd_buf, "nand erase.chip ");
				run_command(cmd_buf, 0);
				break;
			}

			
#ifdef CONFIG_SMART210
			case 'Q':
			case 'q':
			{
				return;	
				break;
			}
#endif
		}
				
	}
}

int do_menu (cmd_tbl_t *cmdtp, int flag, int argc, char *argv[])
{
	menu_shell();
	return 0;
}

U_BOOT_CMD(
	menu,	3,	0,	do_menu,
	"display a menu, to select the items to do something",
	"\n"
	"\tdisplay a menu, to select the items to do something"
);

#endif

{% endhighlight c %}

这里用的比较多的是下面句子:                      
strcpy(cmd_buf, "dnw 0x20000000; nand erase 0x0 0x80000; nand write 0x20000000 0x0 0x80000");
run_command(cmd_buf, 0);

这个句子很简单,调用strcpy把"dnw 0x20000000; nand erase 0x0 0x80000; nand write 0x20000000 0x0 0x80000"的内容复制到cmd_buf中,然后调用run_command();实践上就执行了dnw 0x20000000; nand erase 0x0 0x80000; nand write 0x20000000 0x0 0x80000
这些命令,这与uboot的命令行模式相同的.

