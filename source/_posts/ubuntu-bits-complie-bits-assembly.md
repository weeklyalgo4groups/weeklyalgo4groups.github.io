---
layout: post
title: "Complie 32bits assembly on Ubuntu 64bits"
date: 2018-11-30 11:45:00 +0800
comments: true
categories: assembly
tag: ubuntu
---
<!-- more -->
> - by jay
## 64位Ubuntu系统编译32位汇编
### 汇编链接生成可执行文件

    ld -o eatsyscall eatsycscall.o

### 输出

*ld:i386 architecture of input file `eatsyscall.o' is incompatible with
i386:x86-64 ouput`*

### 解决
经过搜索64位下ld提供了一个选项-m emulation，简写-m。说明文档如下：

    -m emulation
           Emulate the emulation linker. You can list the available
           emulations with the --verbose or -V options.

           If the -m option is not used，the emulation is taken from the
           "LDEMULSYION" environment variable，if that is defined.

           Otherwise，the default emulation depends upon how the linker was
           configured.
即，这个选项可以模拟其他平台上的ld链接器。使用-m elf_i386可以模拟32位平台上
ld指令。
