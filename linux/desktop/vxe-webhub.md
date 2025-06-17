---
title: 如何解决Linux下ATK Webhub无法连接到设备的问题
date: 2025-06-15T19:59:39+08:00
# weight: 1
# aliases: ["/first"]
tags: ["LinuxDesktop", "VXE"]
author: "Me"
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
description: "解决Linux下ATK Webhub无法连接到设备的问题"
disableHLJS: true # to disable highlightjs
disableShare: true
hideSummary: false
searchHidden: true
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: true
ShowRssButtonInSectionTermList: true
UseHugoToc: true
cover:
  image: "<image path/url>" # image path/url
  alt: "<alt text>" # alt text
  caption: "<text>" # display caption under cover
  relative: false # when using page bundles set this to true
  hidden: true # only hide on current single page
---

## 前言

部分内容来自于互联网、朋友、群友。

> [!Tip] 在编写博客的文章时，将默认读者有流畅访问 Github / Google 等网站的能力。

> [!Tip] 使用本文章中的方法需要使用 Linux ROOT 用户，请确保你可以访问 ROOT 权限。

本文引用于 [Reddit: USERNAME123_321 的回答](https://www.reddit.com/r/linux_gaming/comments/1feizmm/comment/mx5sbam/?context=3) ，本站对其回答进行翻译处理。

原po使用的鼠标为蜻蜓R1-SE，很巧的是译者是同款鼠标。

## 正文

1. 使用命令`bash lsusb`查找鼠标的 idVendor 和 idProduct，在"ID"后方有两个 16 进制数值，使用冒号分割，例如：

```
Bus 003 Device 005: ID 3554:{idProduct} Compx VXE Mouse 1K Dongle
                        ^      ^
                    idVendor  idProduct
```

记住其中的idVendor值

2. 切换至 ROOT 用户
3. 在/etc/udev/rules.d/中新建文件:
   - 命名为 50-usb-{name}.rules (例如 50-usb-vxemouse.rules)
   - 扩展名必须以.rules 结尾
4. 在刚才的文件中，写入以下内容：

```
KERNEL=="hidraw\*", ATTRS{idVendor}=="idVendor", MODE="0666"
```

- 这将为每个用户授予指定制造商 ID 的 USB 设备读写权限

1. 使用命令将你的用户添加到 input 组中
   ```bash
   usermod -a -G input {你的用户名}
   ```
2. 重新启动操作系统
3. 打开[ATK Webhub](https://hub.atk.pro/)，此时重新执行配对操作，配对完成后应该可以正常对设备进行操作。



## 问题诊断

如果仍然无法使用，请在使用ATKWebhub的浏览器的地址栏中输入 chrome://device-log/ 打开Chrome设备日志页面以获取更多信息