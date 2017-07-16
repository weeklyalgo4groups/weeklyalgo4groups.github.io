---
title: Reverse Engineer a stripped binary
date: 2017-07-17 23:25:37
categories: pwn
---

# 逆向分析静态编译并裁减了符号表的二进制文件
做题时遇到了这类文件，记录记录恢复函数名过程。
<!-- more -->
> - by hook

## 原程序结构

分析原程序时可以看到一堆未识别的函数，并且未调用任何动态库。
![](/images/Reverse Engineer a stripped binary/1.png)

## 利用函数签名恢复函数名称
### <1>. lscan

使用lscan来比对引入函数量的动态库，这里可以看到libc-2.23.sig这个签名库包含了11.29%的信息
![](/images/Reverse Engineer a stripped binary/2.png)
所以这里把这个签名库拷贝到IDA的sig文件夹下，使用File->Load File->FLIRT signature file功能，加载这个签名库，来识别出一些静态编译在程序中的函数。
![](/images/Reverse Engineer a stripped binary/3.png)
虽然不多，但还是成功识别出一些函数了。
![](/images/Reverse Engineer a stripped binary/4.png)

### <2>. 使用Rizzo来自己生成签名库

当确定静态编译使用的版本时，就可以利用这个来很方便的导出库签名，并加载到要分析的程序中去。
这里我使用了本地libc做了实验，虽然版本不对，但是识别出了更多的函数。
![](/images/Reverse Engineer a stripped binary/5.png)


