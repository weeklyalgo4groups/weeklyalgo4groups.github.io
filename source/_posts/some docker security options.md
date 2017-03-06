---
title: some docker security options
date: 2017-03-6 16:20:37
categories: pwn
---

# 在搭建pwn环境时遇到的一些问题
> - by hook

原本是用虚拟机来进行pwn相关的调试开发等，后来因为固态太小，实在没地放了，就才转向docker。
这里记录下使用时遇到的一些问题。
## 1. gdb调试
docker默认是无法使用gdb调试的，所以在启动是加上参数：
`--security-opt seccomp=unconfined`
## 2. 内核参数设定
测试时经常需要改变一些内核参数，但docker中的proc是只读的，所以重新将其挂载到一个可写目录下
`-v /proc:/writeable-proc`
## 3. gdb attach
还是gdb的问题，当解决上面问题后，发现attach一个进程时，提示ptrace不允许操作。
按正常方法`echo 0 > /proc/sys/kernel/yama/ptrace_scope `却没有用，即使是写入之前挂载的可写目录下也提示没有权限。
这里还是docker的默认安全配置问题，所以启动时加上参数：
`--security-opt apparmor=unconfined`


