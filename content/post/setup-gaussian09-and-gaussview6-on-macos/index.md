---
title: "全网首发！如何在 M1 chip 的 macOS 上优雅的使用 Gaussian 09 与 GaussView 6"
date: 2022-03-08T11:20:00+08:00
tags: ['gaussian', 'chemistry', 'macOS']
---

在 M1 chip 的 macOS 上优雅的使用 Gaussian 09 与 GaussView 6。

<!--more-->

故事起源于早上的物理化学课，老师要求安装 Gauss­ian 09 来预测结构数据。当然 Mac 用户是永远得不到关爱的，老师不会给我提供安装包，于是我就在网上探寻一番，发现 Gauss­ian 09 真的已经太古老了：官方仅仅发行了适用于 Ma­cOS 的 32 位版本。而激进的苹果从 ma­cOS Catalina 开始不再兼容 32 位 App。且网上相关资料较少，我看到了大量求助帖却得不到有效的回复。经过一上午的摸索我终于实现了在 ma­cOS 上优雅的使用 Gauss­ian 09 与 GaussView 6，在这里记录一下踩坑的路，希望可以帮上各位科研工作者。本篇文章力求简单易懂，各位科研工作者可以大胆地迎难而上。

 首先聊一聊 Gauss­ian 套件的结构，简单的说是在 GaussView 上作图，然后通过 Gauss­ian 进行分析运算。最直接粗暴的方法是用 Wine 容器直接安装 Win­dows 版本的 Gauss­ian 与 GaussView（我没试过，盲猜性能极低且兼容性很差）。而我这里的结构是：利用 Crossover 容器运行 GaussView 6，导出草图后放入运行在原生 ma­cOS 上的 Gauss­ian 09 进行运算，实现性能最大化。

先回答几个问题：

- 为什么我要用 GaussView 6 的 Windows 版本？

  答：我能找到的 GaussView 5 的 Mac 版本是 32 位的，并且找不到 GaussView 6 的 Mac 版本；

- 我可以在 Crossover 中运行 GaussView 5 吗？

  答：经过各种参数的枚举，所有我能找到的 GaussView 5 版本均无法成功的在 Crossover 中安装，当然这个结论不一定对，您可以自行安装测试；

- Gaussian 09 需要破解吗？

  答：不需要，但是 GaussView 需要提供一个序列号。

那么，背景铺设完毕，跟着我一起来吧！

------

## 1. 材料准备

- macOS Catalina 以上版本（如果是 Inter 芯片将获得更佳性能，Apple 芯片则需要 Rosetta 2 转译）
- [**Gaussian 09M**](https://github.com/Z-H-Sun/CS_CCME_Posts/blob/hidden/gaussian/gvm.md) 下载 Mac 文件夹下的 G09M.zip
- [**GaussView 6**](https://github.com/Z-H-Sun/CS_CCME_Posts/blob/hidden/gaussian/gvm.md) 下载 Windows 文件夹下的 GV6.0.16_win64.exe，记住文件夹下文本文件提供的序列号
- **需要一些命令基础和反复阅读的双眼**

## 2. 安装 Gaussian 09 

- 解压 G09M.zip 将获得的 gaussian09 文件夹拖入 “应用程序” 文件夹下（/Users/ 你的用户名 / Applications）;

- 按住 command + 空格 输入 terminal 打开终端（# 后内容为注释，不用输入）；

- 设定权限

- 输入命令 chmod 750 ~/Applications/gaussian09

- 设定环境变量

- 输入命令 vim ~/.zshrc # macOS Catalina 及以上版本默认终端应该都是 zsh
  - 按 G    # 跳转到末尾
  - 按 shift + 4  # 跳转到行尾
  - 按 a  # 进入编辑模式
  - 按回车，跳转到新的一行
  - 输入 export g09root=/Applications/gaussian09/  # 然后按回车
  - 输入 export GAUSS_SCRDIR=/Applications/gaussian09/Scratch  # 这句命令是配置临时文件夹，然后按回车
  - 输入 source $g09root/g09/bsd/g09.profile
  - 按 esc 键
  - 按：键
  - 输入 wq
  - 按回车
  - 确保在英文输入模式下
  - 关闭终端，在 dock 栏右击 terminal 图标，点退出，确认下方圆点消失
- 至此，Gaussian 09 安装完毕

## 3. 安装 GaussView 6 

- 安装 [**Crossover**](https://www.codeweavers.com/)**（这个就各显神通了，我个人安装的是** [**Crossover 21.2**](https://appstorrent.ru/185-crossover.html) **，打开需要魔法）；
- 打开 Crossover，选择 安装 Windows 程序；
- 点击左下角，查看所有应用程序；
- 选择 科学，技术与数学 --> 生物与化学 --> Palynodata （没有为什么，因为这是试出来的，可以完美运行），点击继续；
- 选择安装包 --> 下载安装程序 --> 选择之前下载的 GV6.0.16_win64.exe，点击继续
- 容器使用 “新 Windows 10 64-bit 容器”，右边取个名字，然后按继续，就像 Windows 那样安装，记得设桌面图标与文件后缀关联；
- 安装需要序列号，可以在之前百度云中文本文件找到；
- 打开运行吧！

## 4. 协同 Gaussian 09 与 GaussView 6

- 在 GaussView 6 中绘图，保存到 C 盘下；
- 打开 Crossover 主程序，选择右边的容器，右击，“打开 C: 盘”，
- 把文件拷贝到任意文件夹下，例如 “文档” 文件夹（这里按照 “文档” 文件夹做演示，其他文件夹改目录即可）；
- 右击文件，用任意文本编辑器打开，将第一行的 % chk=C:\ 你的文件名.chk 改为 % chk = 你的文件名.chk ，或者直接留空
- 打开终端，输入 cd ~/Documents/
- 按照需要的方法在终端运行

> 常见有以下几种，test.gjf 是输入文件
> g09 <test.gjf> test.out  （信息都输出到 test.out 里。末尾可以再加上 & 指令任务在后台运行）
> g09 < test.gjf |tee test.out （信息输出到 test.out 的同时也同时输出到屏幕上）
> g09 test.gjf （输出文件将默认为当前目录下的 test.log）

​	**大功告成**！ 

## 参考资料：

- [Gaussian 的安装方法及运行时的相关问题](http://bbs.keinsci.com/thread-10814-1-1.html)
- [Gaussian 09 for Mac OS X Installation Instructions](https://www.waseda.jp/mse/web/wp-content/uploads/2017/06/Gaussian09MacOSX_installationInstructions.pdf)
- [面向 Mac 用户 —GaussView 5 for Mac and Gaussian 09M](https://github.com/Z-H-Sun/CS_CCME_Posts/blob/hidden/gaussian/gvm.md)
- [[Gaussian/gview] Gaussian for linux 运行问题](http://bbs.keinsci.com/thread-6006-1-1.html)