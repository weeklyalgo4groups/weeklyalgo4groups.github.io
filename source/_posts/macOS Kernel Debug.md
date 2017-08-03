---
title: macOS Kernel Debug
date: 2017-06-13 23:25:37
categories: Mac
---

# macOS内核／驱动调试
之前做题时遇到了这种需要进入调试内核模式的题，来回顾记录下过程。
<!-- more -->
> - by hook

## Kernel Debug Kit

先从apple developer中心下载调试套件，里面附带了带有符号表的内核和一些驱动模块。

## 虚拟机调试环境
### <1>. 更改boot-args
sudo nvram boot-args="debug=0x141 kext-dev-mode=1 kcsuffix=development pmuflags=1 -v"

### <2>. 更新kernel
cp -rf /Library/Developer/KDKs/KDK_$version$/System /System/
kextcache -invalidate /


### <3>. 关闭SIP
重启虚拟机进入recovery模式，运行csrutil disable

### <4>. 虚拟机成功进入debug模式
重启进入系统，会发现系统处于等待远程debug连接状态
![](/images/macOS Kernel Debug/1.png)

### <5>. 本地lldb进行连接
本地lldb中使用kdp-remote ip连接调试。
![](/images/macOS Kernel Debug/2.png)
按c后可以看到虚拟机正常继续运行了。
![](/images/macOS Kernel Debug/3.png)
### <6>. 加载导致内核panic的模块
使用kextutil -i和kextstat配合得到模块加载的基址。
![](/images/macOS Kernel Debug/4.png)
按Y运行后，kernel panic!
![](/images/macOS Kernel Debug/5.png)
### <7>. 确定panic位置
根据panic地址和前面的基址可以得到原程序中的偏移量。
![](/images/macOS Kernel Debug/6.png)
![](/images/macOS Kernel Debug/7.png)
可以看到是这里的_oskext_call_kext赋值造成的，往上可以查到这里的未初始化操作，同时也可以看到基本分的flag。
![](/images/macOS Kernel Debug/8.png)
![](/images/macOS Kernel Debug/9.png)

### <8>. 模块修复

将上文未符合条件的初始化操作手动patch后再次运行，发现模块一切正常，再无产生kernel panic。


