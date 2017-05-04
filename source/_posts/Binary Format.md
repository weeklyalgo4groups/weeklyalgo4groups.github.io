---
title: Binary Format
date: 2017-05-03 22:28:37
categories: iOS
---

# MacOS/iOS上的可执行二进制文件格式
记录下学习过程
描述了MachO/Universal Binaries/Scripts Image这三种文件格式
后续再写可执行文件加载的流程
> - by hook

<!-- more -->

# MachO文件格式

## 结构总览
官方结构图示:
![](/images/MachOFormat/format0.png)

## 1. MachO Header

```/* * The 32-bit mach header appears at the very beginning of the object file for * 32-bit architectures.*/struct mach_header {    uint32_t magic; /* mach magic number identifier */    cpu_type_t cputype; /* cpu specifier */    cpu_subtype_t cpusubtype; /* machine specifier */    uint32_t filetype; /* type of file */    uint32_t ncmds; /* number of load commands */    uint32_t sizeofcmds; /* the size of all the load commands */    uint32_t flags; /* flags */
    uint32_t reserved; /* 仅限64位MachO，预留字段，默认0 */};/* Constant for the magic field of the mach_header (32-bit architectures) */#define MH_MAGIC 0xfeedface /* the mach magic number */#define MH_CIGAM 0xcefaedfe /* NXSwapInt(MH_MAGIC) */
```

这里列出了32位MachO头(64位为mach_header_64, 多了一个预留字段uint32_t reserved;`目前未使用`)

mach_header:
>1.magic(armv7_armv7s_0xfeedface, arm64_0xfeedfacf)
>2.cputype(armv7_armv7s_0x0000000c, arm64_0x0100000c)
>3.cpusubtype(armv7_0x00000009, armv7s_0x0000000b,...)
>4.filetype:
> >字段值在\<mach-o/loader.h>文件中可以看到
> >
| 文件类型 | 用途 |
| --- | --- |
| MH_OBJECT | 可重定位的目标文件 |
| MH_EXECUTABLE | 可执行二进制文件 |
| MH_DYLIB | 动态库 |
| ... | ... |

>5.加载器加载命令的数量
>6.加载器加载命令的大小
>7.flags二进制属性标示
>>同样位于\<mach-o/loader.h>文件中
>>
| 标志 | 用途 |
| --- | --- |
| MH_TWOLEVEL | 两层名称绑定 |
| MH_ALLOW_STACK_EXECUTION | 允许栈执行 |
| MH_PIE | 启用地址空间布局随机化 |
| MH_NO_HEAP_EXECUTION | 标记堆不可执行，防止”堆喷(heap spray)"攻击 |
| ... | ... |

-------

## 2. Load Commands
通用主体结构在下面列出，但详细的LC存在不同的结构:

```struct load_command {
	uint32_t cmd;		/* type of load command */
	uint32_t cmdsize;	/* total size of command in bytes */
};
```

load_command:
>1. 32位的cmd值描述命令类型，命令在\<mach-o/loader.h>文件中可以看到
> 2. cmdsize描述命令本身长度(32位为4的倍数, 64位为8的倍数)

```
/* Constants for the cmd field of all load commands, the type */
#define	LC_SEGMENT	0x1	/* segment of this file to be mapped */
#define	LC_SYMTAB	0x2	/* link-edit stab symbol table info */
#define	LC_SYMSEG	0x3	/* link-edit gdb symbol table info (obsolete) */
#define	LC_THREAD	0x4	/* thread */
#define	LC_UNIXTHREAD	0x5	/* unix thread (includes a stack) */
#define	LC_LOADFVMLIB	0x6	/* load a specified fixed VM shared library */
#define	LC_IDFVMLIB	0x7	/* fixed VM shared library identification */
#define	LC_IDENT	0x8	/* object identification info (obsolete) */
#define LC_FVMFILE	0x9	/* fixed VM file inclusion (internal use) */
#define LC_PREPAGE      0xa     /* prepage command (internal use) */
#define	LC_DYSYMTAB	0xb	/* dynamic link-edit symbol table info */
#define	LC_LOAD_DYLIB	0xc	/* load a dynamically linked shared library */
#define	LC_ID_DYLIB	0xd	/* dynamically linked shared lib ident */
#define LC_LOAD_DYLINKER 0xe	/* load a dynamic linker */
#define LC_ID_DYLINKER	0xf	/* dynamic linker identification */
#define	LC_PREBOUND_DYLIB 0x10	/* modules prebound for a dynamically */
...
#define LC_CODE_SIGNATURE 0x1d	/* local of code signature */
...
#define LC_LINKER_OPTIMIZATION_HINT 0x2E /* optimization hints in MH_OBJECT files */
#ifndef __OPEN_SOURCE__
#define LC_VERSION_MIN_TVOS 0x2F /* build for AppleTV min OS version */
#endif /* __OPEN_SOURCE__ */
#define LC_VERSION_MIN_WATCHOS 0x30 /* build for Watch min OS version */

```

加载的命令大概有30多条，加载过程在内核部分负责新进程的基本设置，分配虚拟内存，创建主线程，处理代码签名，加密等工作。
对于包含了动态链接的文件，主要的库加载文件和符号解析工作是通过LC_LOAD_DYLINER命令指定的动态链接器(/usr/lib/dyld)在用户态完成的，控制权转交给链接器，进而处理文件头中的其他加载命令。

接着详细说明一些Load Commands段中的组成：
### <1>. LC_SEGMENT

```
struct segment_command { /* for 32-bit architectures */
	uint32_t	cmd;		/* LC_SEGMENT */
	uint32_t	cmdsize;	/* includes sizeof section structs */
	char		segname[16];	/* segment name */
	uint32_t	vmaddr;		/* memory address of this segment */
	uint32_t	vmsize;		/* memory size of this segment */
	uint32_t	fileoff;	/* file offset of this segment */
	uint32_t	filesize;	/* amount to map from the file */
	vm_prot_t	maxprot;	/* maximum VM protection */
	vm_prot_t	initprot;	/* initial VM protection */
	uint32_t	nsects;		/* number of sections in segment */
	uint32_t	flags;		/* flags */
};
```

64位的LC_SEGMENT结构定义和32位类似，但在vmaddr等虚拟内存大小，地址还有segment大小和相对文件偏移方面采用了uint64_t长度。

>maxprot: 段页面的最高内存保护
>initprot: 初始内存保护
>nsects: 其中包含的section数量
>flag: 段页面标志(比如0x08,加密的段)

LC_SEGMENT命令将fileoff偏移处的filesize大小内容加载到大小为vmsize的虚拟内存地址vmaddr处。
段页面权限由`initprot`进行初始化，而且段的权限可以动态改变,但不能超过`maxprot`的值。(iOS中段权限不允许x和w同时存在)

至于段的`flag`标志可以用以下几种，例如`SG_PROTECTED_VERSTION_1`标记该段受保护，苹果通过该技术加密二进制文件(AppStore中的应用就被加密了，但仍旧被insert_dylib dump出解密后文件)

```
#define	SG_HIGHVM	0x1	/* the file contents for this segment is for
				   the high part of the VM space, the low part
				   is zero filled (for stacks in core files) */
#define	SG_FVMLIB	0x2	/* this segment is the VM that is allocated by
				   a fixed VM library, for overlap checking in
				   the link editor */
#define	SG_NORELOC	0x4	/* this segment has nothing that was relocated
				   in it and nothing relocated to it, that is
				   it maybe safely replaced without relocation*/
#define SG_PROTECTED_VERSION_1	0x8 /* This segment is protected.  If the
				       segment starts at file offset 0, the
				       first page of the segment is not
				       protected.  All other pages of the
				       segment are protected. */
```
#### SECTION
一个段(Segment)是由 0 到 n 个区(section)组成的:

```
struct section { /* for 32-bit architectures */
	char		sectname[16];	/* name of this section */
	char		segname[16];	/* segment this section goes in */
	uint32_t	addr;		/* memory address of this section */
	uint32_t	size;		/* size in bytes of this section */
	uint32_t	offset;		/* file offset of this section */
	uint32_t	align;		/* section alignment (power of 2) */
	uint32_t	reloff;		/* file offset of relocation entries */
	uint32_t	nreloc;		/* number of relocation entries */
	uint32_t	flags;		/* flags (section type and attributes)*/
	uint32_t	reserved1;	/* reserved (for offset or index) */
	uint32_t	reserved2;	/* reserved (for count or sizeof) */
};
```

64位的section结构和32位相同，但memory addr&size为`uint64_t`长度，并且增加了个预留字段`uint32_t reserved3`;

>1. sectname: 当前区的名字
>2. segname: 对应后面真实Section段的名字
>3. addr: 这个区在虚拟内存中的地址
>4. size: 这个区在虚拟内存中将占据的大小
>5. offset: 该区相对文件的偏移
>6. align: 该区的字节对齐大小n，最后计算结果为2^n
>7. reloff: 该区的第一个重定位入口对应文件的偏移
>8. nreloc: 该区包含的重定位入口数量
>9. flag: 该标志包含两个信息，(`section的类型` 和 `section的属性`)
>
>>`0x000000ff`最后2位定义section type
>>`0xffffff00`前面6位定义section attributes

```
section type:
...
#define	S_REGULAR		0x0	/* regular section */
#define	S_ZEROFILL		0x1	/* zero fill on demand section */
#define	S_CSTRING_LITERALS	0x2	/* section with only literal C strings*/
#define	S_SYMBOL_STUBS	0x8	/* section with only symbol
						   stubs, byte size of stub in
						   the reserved2 field */
...
section attributes:
#define S_ATTR_PURE_INSTRUCTIONS 0x80000000	/* section contains only true
						   machine instructions */
#define S_ATTR_SOME_INSTRUCTIONS 0x00000400	/* section contains some
						   machine instructions */
...
```
而这些不同的flag标志构成了不同的section:

| section | 用途 |
| --- | --- |
| __text | 主程序代码 |
| __stubs, __stub_helper | 用于动态链接的stub |
| __cstring | 程序中硬编码的字符串 |
| ... | ... |
| \__TEXT.__obj_method | OC方法名  |
| ... | ... |
| \__DATA.__bss | BSS  |
| ... | ... |

```
`S_ATTR_PURE_INSTRUCTIONS`: This section contains only executable machine instructions. The standard tools set this flag for the sections __TEXT,__text, __TEXT,__symbol_stub, and __TEXT,__picsymbol_stub.`S_ATTR_SOME_INSTRUCTIONS`: This section contains executable machine instructions.
...
`S_REGULAR`: This section has no particular type. The standard tools create a __TEXT,__text section of this type.`S_ZEROFILL`: Zero-fill-on-demand section—when this section is first read from or written to, each page within is automatically filled with bytes containing zero. __DATA.__bss
`S_CSTRING_LITERALS`: This section contains only constant C strings. The standard tools create a __TEXT,__cstring section of this type.
```

### <2>. LC_LOAD_DYLINER

```
struct dylib_command {
	uint32_t	cmd;		/* LC_ID_DYLIB, LC_LOAD_{,WEAK_}DYLIB,
					   LC_REEXPORT_DYLIB */
	uint32_t	cmdsize;	/* includes pathname string */
	struct dylib	dylib;		/* the library identification */
};

struct dylib {
    union lc_str  name;			/* library's path name */
    uint32_t timestamp;			/* library's build time stamp */
    uint32_t current_version;		/* library's current version number */
    uint32_t compatibility_version;	/* library's compatibility vers number*/
};

union lc_str {
	uint32_t	offset;	/* offset to the string */
#ifndef __LP64__
	char		*ptr;	/* pointer to the string */
#endif 
};
```

dyld加载命令

### <3>. LC_CODE_SIGNATURE
```
struct linkedit_data_command {
    uint32_t	cmd;		/* LC_CODE_SIGNATURE, LC_SEGMENT_SPLIT_INFO,
                                   LC_FUNCTION_STARTS, LC_DATA_IN_CODE,
				   LC_DYLIB_CODE_SIGN_DRS or
				   LC_LINKER_OPTIMIZATION_HINT. */
    uint32_t	cmdsize;	/* sizeof(struct linkedit_data_command) */
    uint32_t	dataoff;	/* file offset of data in __LINKEDIT segment */
    uint32_t	datasize;	/* file size of data in __LINKEDIT segment  */
};
```

代码签名，这个命令记录了该MachO二进制文件代码签名的分布信息，如果代码签名与二进制不符合，或者在iOS上检测machO文件没有该命令时，内核会向进程发送SIGKILL信号，杀死进程。

### <4>. LC_MAIN
```
struct entry_point_command {
    uint32_t  cmd;	/* LC_MAIN only used in MH_EXECUTE filetypes */
    uint32_t  cmdsize;	/* 24 */
    uint64_t  entryoff;	/* file (__TEXT) offset of main() */
    uint64_t  stacksize;/* if not zero, initial stack size */
};
```
该命令记录主线程的入口信息，从Lion版本时抛弃了LC_UNIXTHREAD命令改为LC_MAIN,但同时意味着不能在旧版本上运行使用LC_MAIN命令入口的程序

### <...>. 其他命令就暂不作记录了

-------


## 3. DATA区

接着就是MachO的主体，Section Data, 上文LC命令中包含了section command的信息指向了这里
### <1>. Section(__TEXT)
这里一般情况下存储一些主程序代码，动态链接的stub和helper代码,还有一些硬编码和plist信息
### <2>. Section(__DATA)
这里存放一些OC原型和镜像信息，还有symbol bind信息,bss等
### <3>. Symbol Table Info
### <4>. Dynamic Symbol Table Info
### <5>. Dynamic Loader Info
### <6>. String Table
### <7>. Code Signature
等等信息，因为这里数据可变，所以就不做详细记录了

-------
-------
-------

# Universal Binaries(Fat File)
## 结构总览
结构实例:
![](/images/MachOFormat/format1.png)
## Fat Header:

```
#define FAT_MAGIC 0xcafebabe#define FAT_CIGAM 0xbebafeca /* NXSwapLong(FAT_MAGIC) */
struct fat_header {    uint32_t magic;    uint32_t nfat_arch;};struct fat_arch {    cpu_type_t cputype; /* cpu specifier (int) */    cpu_subtype_t cpusubtype; /* machine specifier (int) */    uint32_t offset; /* file offset to this object file */
    uint32_t size; /* size of this object file */
    uint32_t align; /* alignment as a power of 2 */ };
```
fat_header:
>1.magic描述文件类型, 例如`Fat File[0xcafebabe]`
>2.nfat_arch描述当前的FatFile包含多少架构的MachO文件

fat_arch：紧跟在fat_header后，用来描述后续的MachO信息
>...
>3.offset对应MachO相对于此文件的偏移
>4.size对应MachO大小
>5.align对齐页边界(4k),表示为2的n次幂(即12)


-------
接下来就是nfat_arch个与之对应的MachO文件
## MachO File 1
## MachO File 2
...
...
...
## MachO File n

-------
# Scripts Image
可执行的脚本文件，这个本不应该出现在这篇binary format介绍里，我一直以为脚本文件由sh等类似的shell去解析。
但在翻内核源码时，看到imgact的启动时将MachO/Fat File/Scripts三种文件都当作image文件去加载，只是最后使用了不同的image activator去解析。

这里把script仍旧分解成image去image activator。

```
...
if (vdata[0] != '#' ||
	    vdata[1] != '!' ||
	    (imgp->ip_flags & IMGPF_INTERPRET) != 0) {
		return (-1);
	}

	if (imgp->ip_origcputype != 0) {
		/* Fat header previously matched, don't allow shell script inside */
		return (-1);
	}

	imgp->ip_flags |= IMGPF_INTERPRET;
	imgp->ip_interp_sugid_fd = -1;
	imgp->ip_interp_buffer[0] = '\0';

	/* Check to see if SUGID scripts are permitted.  If they aren't then
	 * clear the SUGID bits.
	 * imgp->ip_vattr is known to be valid.
	 */
	if (sugid_scripts == 0) {
		imgp->ip_origvattr->va_mode &= ~(VSUID | VSGID);
	}

	/* Try to find the first non-whitespace character */
	for( ihp = &vdata[2]; ihp < &vdata[IMG_SHSIZE]; ihp++ ) {
		if (IS_EOL(*ihp)) {
			/* Did not find interpreter, "#!\n" */
			return (ENOEXEC);
		} else if (IS_WHITESPACE(*ihp)) {
			/* Whitespace, like "#!    /bin/sh\n", keep going. */
		} else {
			/* Found start of interpreter */
			break;
		}
	}
	...
	/* The character after the last non-whitespace is our logical end of line */
	line_endp = ihp + 1;

	/*
	 * Now we have pointers to the usable part of:
	 *
	 * "#!  /usr/bin/int first    second   third    \n"
	 *      ^ line_startp                       ^ line_endp
	 */

	/* copy the interpreter name */
	interp = imgp->ip_interp_buffer;
	for ( ihp = line_startp; (ihp < line_endp) && !IS_WHITESPACE(*ihp); ihp++)
		*interp++ = *ihp;
	*interp = '\0';

	exec_reset_save_path(imgp);
	exec_save_path(imgp, CAST_USER_ADDR_T(imgp->ip_interp_buffer), UIO_SYSSPACE, NULL);
。..
```


