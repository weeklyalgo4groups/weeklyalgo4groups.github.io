---
title: 关于v2ray科学上网的简单配置过程
date: 2017-08-03 18:30:25
categories: v2ray
---


本教程**服务器OS**,**客户端OS**分别使用**Debian 8**和**ubuntu 16.04**

由于v2ray给的文档有些比较复杂，所以这里简化一下步骤

<!-- more -->
> - by wkzcn


**配置`config.json`时，记得去掉带有"//"的内容**


prerequirements：

     1. 大陆以外的vps

     2. ssh客户端

     3. sublime-text/jq/tcptrack

     4. [生成UUID](https://www.uuidgenerator.net/)

     5. 有点点地理知识(个人认为)

---


### 服务器配置

   1. 先查看vps服务器和客户端的时间是否有误差，误差要不大于一分钟。如有误差，请自行设置，这里推荐设置客户端的时间。

     ```bash
     $ date -R
     Sun, 22 Jan 2017 10:10:36 +0000
     ```

     这个+0500是东五区，因为我们是在东8区，相差三个时区，所以转换过来的时间就是2017-01-22 13:10:36。如果客户端时间与vps时间误差大于**1分钟**，请设置客户端的时间为转换后的时间。

   2. vps安装v2ray
         ```bash
         $ sudo apt-get update
         $ sudo apt-get install curl jq tcptrack
         $ curl https://install.direct/go.sh | sudo bash
         $ sudo service v2ray status  # 出现如下信息，表示v2ray安装成功
         # ● v2ray.service - V2Ray Service
         # Loaded: loaded (/lib/systemd/system/v2ray.service; enabled)
         #  Active: inactive (dead)
         ```
   3. 备份v2ray的配置文件

         ```bash
         $ sudo cp /etc/v2ray/config.json /etc/v2ray/config.json.bak
         ```
   4. 生成UUID

        [这里](https://www.uuidgenerator.net/)生成UUID,并复制

   5. 服务器配置

        ```bash
        $ sudo vim /etc/v2ray/config.json
        ```
       ```
       {
          "inbound": {
            "port": 16823, // 服务器监听端口，更改为你喜欢的端口
            "protocol": "vmess",    // 主传入协议，无需更改
            "settings": {
              "clients": [
                {
                  "id": "b831381d-6324-4d53-ad4f-8cda48b30811",  // 用户 ID，客户端与服务器必须相同，填上刚才复制的uuid
                  "alterId": 64  // 无需更改，官方推荐50-100
                }
              ]
            }
          },
          "outbound": {
            "protocol": "freedom",  // 主传出协议，参见协议列表，无需更改
            "settings": {}
           }
       }
       ```

   6. 检查config.json
      ```json
      $ sudo jq . /etc/v2ray/config.json
      ```
   7. 启动v2ray
      ```bash
      $ sudo service v2ray start
      ```

## 客户端配置
1. v2ray 安装
   ```bash
    $ wget https://install.direct/go.sh
    $ sudo bash ./go.sh
    ```
2. 备份v2ray的配置文件

  ```bash
  $ sudo cp /etc/v2ray/config.json /etc/v2ray/config.json.bak
  # 如果报错,执行下面命令行
  $ sudo touch /etc/v2ray/config.json
  ```
3. 编辑config.json
  ```bash
  $ sudo subl /etc/v2ray/config.json
  ```
  ```json
  {
    "inbound":
    {
        "port": 1080, // 监听端口, 无需更改
        "protocol": "socks", // 入口协议为 SOCKS 5， 无需更改
        "settings":
        {
            "auth": "noauth" // 不认证， 无需更改
        }
    },
    "outbound":
    {
        "protocol": "vmess", // 出口协议， 无需更改
        "settings":
        {
            "vnext": [
            {
                "address": "serveraddr.com", // 服务器地址，请修改为你自己的服务器 ip 或域名
                "port": 16823, // 服务器端口，更改为你喜欢的端口
                "users": [
                {
                    "id": "b831381d-6324-4d53-ad4f-8cda48b30811", // 用户 ID，必须与服务器端配置相同，填上刚才生成的UUID
                    "alterId": 64 // 此处的值也应当与服务器相同
                }]
            }]
        }
    }
}
```

## 如何科学上网
1. 启动v2ray
    ```bash
    $ sudo service v2ray start
    ```
2. chrome(谷歌)浏览器设置
   1. Proxy Switchy Omega
      
      [下载ProxySwitchyOmega](http://s1.alyzq.com/0ad8b71257e86f55b30dadece8fd1643/5989ad79/Page/crx/PxxxroxySxxxwitchyOmega.crx)
   2. 安装ProxySwitchyOmega
      >第一种方法. Tools(工具)--->Extensions(扩展)--->拖拽ProxySwitchyOmega到Extensions界面安装
      >
      >第二种方法. 上网搜索如何安装
   3. 设置ProxySwitchyOmega
      ![设置ProxySwitchyOmega](http://wx4.sinaimg.cn/mw690/7372620bgy1ficm1f2btpj211v0h6jua.jpg)

3. firefox(火狐浏览器)
   >1. preferences(偏好设置)--->Advanced(高级)--->Network(网络)--->Settings(设置)--->Manual Proxy configuration(手动设置代理)--->Socks Host
   > 2. 第一个方框填127.0.0.1, 第二个填上1080
   > 3. 勾选proxy DNS when using socks v5
   > 4. 保存(OK)
- - -
这时候浏览器google一下，同时服务器tcptrack

```bash
$ sudo tcptrack -i eth0
# 这里看一下server和State(ESTABLISHED代表已经建立客户端和服务器端的连接)
# Enjoy it！！！
```

