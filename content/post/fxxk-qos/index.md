---
title: "使用多倍发包抵抗QoS"
date: 2025-08-15T12:00:00+08:00
tags: ['network']
---

打个游戏真难。

<!--more-->

## 起因

起因是我想和远在三千公里外的好朋友联机玩我的世界，但彼此都没有公网 IP，因此我们用了自建的 zerotier 网络将彼此的设备连接起来，并且可以直连。然而在晚高峰时期会非常卡，游戏体验极差。众所周知，zerotier 使用的 UDP 通道会被运营商 QoS，为了保证传输数据的完整性，就必须大量重传，这对于游戏、直播（包括 moonlight）之类即时性的应用场景是灾难性的。而 zerotier 本身可定制性低、协议修改困难，因此，我在 zerotier 网络内部额外套了多倍发包的方案（kcptun+UDPspeeder），虽然此方案不优雅（因为外层的 zerotier 的 UDP 通道设计是可靠的，因此会造成比预期更高的带宽浪费），但是这个方案集成了 zerotier 的 P2P 通道优势和较好的稳定性。在这里，我列出了自己的配置方案，方便读者稍微修改就能嵌入到自己的使用场景中。

## 步骤

下文的配置目标如下：

> Zerotier 局域网具有设备A（IP: 10.10.0.1）和设备B (IP: 10.10.0.2)，在设备A上开设了服务1和服务2，服务1监听 TCP 的 50000 端口，服务2监听 UDP 的 50000 端口。期望设备B可以高性能地访问设备A的服务1和服务2。

1. 确定端口号和协议

   首先，需要确定你的服务使用的端口号和使用的协议（TCP or UDP）。

2. 下载工具

   * [kcptun](https://github.com/xtaci/kcptun/releases/tag/latest): 用于建立传输 TCP 数据流的通道
   * [UDPspeeder](https://github.com/wangyu-/UDPspeeder/releases/latest): 用于建立传输 UDP 数据流的通道

3. 启动 TCP 隧道

   * 设备A：

     ```cmd
     # kcptun 中的 server
     server_windows_amd64.exe -l ":4000" -t "127.0.0.1:50000" -crypt none -mode fast3
     ```

     将设备A的 TCP 50000 端口的业务包裹在 4000 端口的 KCP 通道服务端。关闭加密是因为 Zerotier 通道本身已提供加密。采用重传最积极的 `fast3` 模式。

   * 设备B:

     ```cmd
     # kcptun 中的 client
     client_windows_amd64.exe -l ":50000" -r "10.10.0.1:4000" -crypt none -mode fast3
     ```

     连接设备A的 4000 端口的通道服务端，将设备A的 TCP 50000 端口映射到设备B的 TCP 50000 端口。

4. 启动 UDP 通道

   * 设备A:

     ```cmd
     # UDPspeeder
     speederv2.exe -s -l 0.0.0.0:4096 -r 127.0.0.1:50000 -f 2:4 --timeout 0
     ```

     将设备A的 UDP 50000 端口的业务包裹在 4096 端口的 UDPspeeder 通道服务端。关闭加密是因为 Zerotier 通道本身已提供加密。使用大于 3 倍流量的游戏场景策略。

   * 设备B:

     ```cmd
     # UDPspeeder
     speederv2.exe -c -l 0.0.0.0:50000 -r 10.10.0.1:4096 -f 2:4 --timeout 0
     ```

     连接设备A的 4096 端口的通道服务端，将设备A的 UDP 50000 端口映射到设备B的 UDP 50000 端口。

⚠️ 提示

* 命令中使用的策略倾向于游戏等追求**低延迟、流量不敏感**的需求，如有其他需求（如直播等），请参考工具文档进行修改；
* 可以 TCP 和 UDP 同时监听相同的端口，因为 TCP 50000 端口和 UDP 50000 端口是独立的；
* 如果遇到配置无误却连接不上的情况，请检查 Windows 防火墙。😡

## 参考文献

[1] [UDPspeeder kcptun finalspeed $$ 同时加速tcp和udp流量](https://github.com/wangyu-/UDPspeeder/wiki/UDPspeeder---kcptun-finalspeed---$$-%E5%90%8C%E6%97%B6%E5%8A%A0%E9%80%9Ftcp%E5%92%8Cudp%E6%B5%81%E9%87%8F)

[2] [UDPspeeder 推荐设置](https://github.com/wangyu-/UDPspeeder/wiki/%E6%8E%A8%E8%8D%90%E8%AE%BE%E7%BD%AE)

[3] [kcptun 文档](https://github.com/xtaci/kcptun)