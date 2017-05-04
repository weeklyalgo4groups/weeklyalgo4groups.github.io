---
title: "operation not permitted" with root
date: 2017-05-04 22:28:37
categories: Mac
---

# operation not permitted
在Mac下root权限操作仍旧得到"operation not permitted"提示

重启按住`Command+R`进入恢复模式
`csrutil disable`
再重启进入系统即可解决。
详细原因是...
> - by hook

<!-- more -->

# SIP

在Mac OS X 11以后，就算是root用户，去操作一些系统目录，也会收到"operation not permitted"提示。
例如

```
sudo ln -s xxxx /usr/xxx
```
等等敏感命令

这是因为`El Capitan`引入了`Rootless`机制
这里有两种解决办法：

## 1.更改目录
改变操作目录，比如把链接到`/usr/bin`的操作改为`/usr/local/bin`这种同效果操作。

## 2.关闭SIP
看到别人说Rootless机制将成为对抗恶意程序的最后防线，所以这种办法自己选择吧。

重启按住`Commadn+R`进入恢复模式
运行`csrutil disable`关闭SIP

要恢复的话
`csrutil enable`

### csrutil用法

csrutil命令参数格式：

csrutil enable [--without kext | fs | debug | dtrace | nvram][--no-internal]

禁用：`csrutil disable`

>（等同于csrutil enable --without kext --without fs --without debug --without dtrace --without nvram）

其中各个开关，意义如下：

>B0: [kext] 允许加载不受信任的kext（与已被废除的kext-dev-mode=1等效）
B1: [fs] 解锁文件系统限制
B2: [debug] 允许task_for_pid()调用
B3: [n/a] 允许内核调试 （官方的csrutil工具无法设置此位）
B4: [internal] Apple内部保留位（csrutil默认会设置此位，实际不会起作用。设置与否均可）
B5: [dtrace] 解锁dtrace限制
B6: [nvram] 解锁NVRAM限制
B7: [n/a] 允许设备配置（新增，具体作用暂时未确定）


