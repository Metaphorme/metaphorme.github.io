---
title: "巧用反向代理绕过单向防火墙"
date: 2022-06-03T12:00:00+08:00
tags: ['network']
---

记一次与学校防火墙斗智斗勇的故事。

<!--more-->

> 身无彩凤双飞翼，心有灵犀一点通。
>
> ———《无题》 李商隐

**1. 起因**

学校的校园网比较开放。当宿舍路由器以 DHCP 客户端的身份接入校园网的时候会被分配到一个内网 IP，但是这个 IP 段是不能通过认证上网的，但是可以连接局域网内主机。正确上网方法是需要购买运营商服务后通过 PP­PoE 协议建立连接外网的通道，但是每晚 11 点断网实在是令人无法正常学（mo）习（yu）。为了快乐学（mo）习（yu），我们在可以认证上网的区域建立了服务器进行数据流转发。就这样快乐学习了一学期后，学校斥巨资建立了堡垒机。为了所谓的安全将所有不可认证 IP 段（下文简写为 “宿舍”）到可认证 IP 段（下文简写为 “实验室”）发起的连接路由到错误的地址进而阻断连接（因此无法 Ping 通），但是没有阻断从实验室发起到宿舍的数据流。因此绕过堡垒机的方案变得简单很多 —— 只需要实验室主机主动建立与宿舍设备的一条全双工通道便可以成功建立连接。

本文力求通过简单的工具实现被动建立连接，而不是一味追求底层（显然会获得更高的性能）。本文所使用的工具被用来穿透内网，与某著名防火墙无关。**仅以学习交流为目的写就此文，请勿利用本文内容做任何反社会、非法的事，本人不承担任何造成的后果。**另外，本人能力不佳，行文有很多纰漏，还是烦请看官多批评指教。

**请注意，在被动连接的时候都需要宿舍机 IP 不发生变化，变化以后需要重新修改配置文件的 IP 处。**

**2. 方法一：利用 SSH 隧道建立远程端口转发（不推荐）**

优点：配置相对简单，自带加密，**不需要额外组件**

不足：需要宿舍机完整 SSH 权限，速度较慢（单通道），udp 支持麻烦，宿舍机需要开启 sshd。

流量拓扑图：

> 实验室机 —— 建立 ssh 通道到宿舍机器 ——> 宿舍机器 ssh server 监听 26000 端口，并将所有数据通过 ssh tunnel 加密后转发给实验室机器的 21 端口（表面是普通的 ssh 连接）——> 实验室机将 21 端口数据转发到本机 9200 端口 ——> SagerNet

配置方法：

1. 在实验室路由器上运行：

   ```bash
   ssh -R 26000:lo­cal­host:9200 {用户名}@{宿舍机 IP}
   ```

   R 参数接受三个值，分别是 "远程主机端口：目标主机：目标主机端口"。这条命令的意思，就是让宿舍机监听它自己的 26000 端口，然后将所有数据发给自身主机 9200 端口。

   这里也可以通过 -NT,-D 等参数进行仅流量转发而不建立远程 Shell、后台运行等操作。

2. 在实验室机器用任意工具监听 9200 端口（例如 *ray，在 9200 上 Socks5 入站，Free­dom 出站即可）

3. 宿舍机配置用 Sock5:26000 作为流量入口

**进阶操作：**

​      在宿舍机和实验室机入口出口用 ipt­a­bles 转发，性能会高很多。

**3. 方法二：利用 \*ray 反向代理（推荐）**

  优点：可操作性强、玩法多样、配置简单

  不足：*ray 的体积（近 20 Mb）对硬路由来说有点大。如果仅做反向代理可以不要 geo* 等两个名单，这样可以省很多空间。

  流量拓扑图：

> 实验室机 —— 主动发起连接，建立全双工隧道 ——> 宿舍机在隧道上向实验室发送数据流 ——> 实验室机路由判断进行数据转发

  配置方法：

 1. 实验室机配置文件：

    ```json
    {
        "log": {
            "loglevel": "debug"
        },
        "reverse": {    
            "bridges": [
            {
                "tag": "bridge",
                "domain": "test.v2fly.org"     # test.v2fly.org 只做为路由判断的标识，不需要（最好不要）真正存在
            }
        ]},
        "outbounds": [{
            "tag": "out",
            "protocol": "freedom"
        },
        {
            "protocol": "shadowsocks",
            "settings": {
                "servers": [
                    {
                        "address": "宿舍机 IP",
                        "port": 20000,
                        "method": "aes-128-gcm",  # 内网传输，稍微加密一下即可；大多数 cpu 都有解密 aes 的硬模块，会更快
                        "password": "password",
                        "network": "tcp,udp"
                    }
                ]
            },
            "tag": "interconn"
        }],
        "routing": {
            "rules": [
                {
                    "type": "field",
                    "inboundTag": [
                        "bridge"
                    ],
                    "domain": [
                        "full:test.v2fly.org"
                    ],
                    "outboundTag": "interconn"
                },
                {
                    "type": "field",
                    "inboundTag": [
                        "bridge"
                    ],
                    "outboundTag": "out"
                }
            ]
        }
    }
    ```

2. 宿舍机配置文件

   ```json
   {
       "log": {
           "loglevel": "debug"
       },
       "reverse": {
           "portals": [
           {
               "tag": "portal",
               "domain": "test.v2fly.org"
           }
       ]},
       "inbounds": [                           # 流量入口，可以再加个 HTTP 入口之类的
           {
               "tag": "external",
               "port": 26174,
               "protocol": "socks",
               "settings": {
                   "auth": "noauth",
                   "udp": true,
                   "ip": "127.0.0.1"
               }
           },
           {
               "port": 20000,
               "tag": "interconn",
               "protocol": "shadowsocks",
               "settings": {
                   "method": "aes-128-gcm",
                   "password": "password",
                   "network": "tcp,udp"
               }
           }
       ],
       "routing": {
           "rules": [
               {
                   "type": "field",
                   "inboundTag": [
                       "external"
                   ],
                   "outboundTag": "portal"
               },
               {
                   "type": "field",
                   "inboundTag": [
                       "interconn"
                   ],
                   "outboundTag": "portal"
               }
           ]
       }
   }
   ```

3. 在宿舍机防火墙放通 20000 端口

4. 建立连接即可

这里已知有一个问题，就是无法接收 UDP 流量。我不太清楚这个问题是怎样产生的，但是利用 Shad­ow­socks 原生转发 UDP 显然是最优解。在官方文档中利用了 VMESS 的 UDP over TCP 来规避这个问题。

**进阶操作：利用服务自启动 V2ray 服务**

1. vim /etc/init.d/v2ray 内容如下

   ```bash
   #!/bin/sh /etc/rc.common
   # "new(er)" style init script
   # Look at /lib/functions/service.sh on a running system for explanations of what other SERVICE_
   # options you can use, and when you might want them.
   
   START=99
   
   SERVICE_USE_PID=1
   
   SERVICE_WRITE_PID=1
   
   SERVICE_DAEMONIZE=1
   
   start() {
           service_start /usr/bin/v2ray -config=/etc/v2ray/config.json  # 配置文件位置
   }
   
   stop() {
           service_stop /usr/bin/v2ray
   }
   ```

2. chmod 755 /etc/init.d/v2ray  # 设置文件权限

3. /etc/init.d/v2ray en­able  # 开机启动

配置成功后，当实验室路由启动时便会自动发起连接，非常的方便。

**4. 小结与思考**

我不禁思考，这种单向墙真的可以有效阻断黑客入侵吗？它确实可以有效阻止 Eter­nal­Blue 等 0day 漏洞从外网发起到内网的渗透。而对于一心攻破内网的黑客而言，他们只需要通过某种途径（甚至是最简单的社工方法）将经过设计的 pay­load 安置在内网机器，然后用 me­ter­preter**_re­verse_tcp 之类的东东反向连接服务端即可建立持久地连接。因此，在需要绝对安全需求的内网建立物理隔离是必要的，而这种绝对安全的 “内网” 一定不应该太大。对于实验室、教室、宿舍这种超大型设备网来说，单纯以 IP 段区别绝对安全的 “内网” 与不那么安全的 “内网” 太宽泛了，是没有意义的，这也许预示了这堵墙以后的命运。

这个故事也有一些社会学色彩。缘分总是那么多，何况感情更是可遇而不可求。如果真的累了，不如回到自己的舒适区，努力地提升自己，勇敢地追寻自己的梦想和阳光。因为真爱无论山河艰险，一定会踩着星河与波涛奔赴而来。正如文头所说，身无彩凤双飞翼，心有灵犀一点通。

**5. 后续**

在堡垒机高强度无差别阻断的一周之后，它神秘地消失了。                            -------- 2022-06-20

使用 webVPN 从内网绕过了防火墙 [webVPN_mitm](https://github.com/Metaphorme/webVPN_mitm)。                                      -------- 2022-10-12

**参考文献：**

[1]: [SSH 原理与运用（二）：远程操作与端口转发](https://www.ruanyifeng.com/blog/2011/12/ssh_port_forwarding.html)；阮一峰；2011 年 12 月 23 日

[2]:[V2Fly 配置文档：Reverse 反向代理](https://www.v2fly.org/config/reverse.html)；V2Fly Com­mu­nity

[3]:[openwrt 官方固件使用 v2 瑞的手动配置教程](https://www.right.com.cn/forum/thread-938058-1-1.html)；99010；2019-8-27
