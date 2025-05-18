---
title: 未标题的文章
date: 2025-05-16T10:54:28+08:00
# weight: 1
# aliases: ["/first"]
tags: [""]
author: "Me"
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
description: "untited post"
disableHLJS: true # to disable highlightjs
disableShare: true
disableHLJS: false
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

## 正文

### MSR-1 
首先进入系统视图：
```
<H3C>SYS
System View: return to User View with Ctrl+Z.
[H3C]sysname MSR-1
```
为路由器配置IP：
```
[MSR-1]int 0/0
[MSR-1]ip add 192.168.1.1 24
[MSR-1]qu
```
然后启用ssh：
```
[MSR-1]ssh server enable
[MSR-1]
```
设置用于SSH的用户:
```
[MSR-1]local-user WLJDKS123 class manage
[MSR-1-luser-manage-WLJDKS123]password-control length 9
[MSR-1-luser-manage-WLJDKS123]password simple WLJDKS123
[MSR-1-luser-manage-WLJDKS123]undo password-control complexity user-name check
[MSR-1-luser-manage-WLJDKS123]service-type ssh
[MSR-1-luser-manage-WLJDKS123]authorization-attribute user-role network-admin
[MSR-1-luser-manage-WLJDKS123]quit
[MSR-1]

```
配置同时可登录的最大数量：
```
[MSR-1]user-int vty 0 4
[MSR-1-line-vty0-4]authentication-mode scheme
[MSR-1-line-vty0-4]qu
[MSR-1]

```

### SW-1
首先进入系统视图：
```
<H3C>sys
System View: return to User View with Ctrl+Z.
[H3C]sysname SW-1
[SW-1]
```
为交换机配置IP：
```
[SW-1]int g 1/0/1
[SW-1-GigabitEthernet1/0/1]port link-mode route 
The configuration of the interface will be restored to the default. Continue? [Y/N]:y
[SW-1-GigabitEthernet1/0/1]%May 16 11:04:37:748 2025 H3C IFNET/3/PHY_UPDOWN: Physical state on the interface GigabitEthernet1/0/1 changed to down.
%May 16 11:04:37:748 2025 H3C IFNET/5/LINK_UPDOWN: Line protocol state on the interface GigabitEthernet1/0/1 changed to down.
%May 16 11:04:39:784 2025 H3C IFNET/3/PHY_UPDOWN: Physical state on the interface GigabitEthernet1/0/1 changed to up.
%May 16 11:04:39:785 2025 H3C IFNET/5/LINK_UPDOWN: Line protocol state on the interface GigabitEthernet1/0/1 changed to up.
[SW-1-GigabitEthernet1/0/1] ip add 192.168.1.2 24
[HSW-13C-GigabitEthernet1/0/1] undo shutdown
[SW-1-GigabitEthernet1/0/1] quit
[SW-1]

```

# SW-1 (user)
在SW-1切换到用户视图，然后尝试连接到MSR-1：
```
<SW-1> ssh2 192.168.1.1
Username: WLJDKS123
Press CTRL+C to abort.
Connecting to 192.168.1.1 port 22.
WLJDKS123@192.168.1.1's password: <输入设定的密码>
Enter a character ~ and a dot to abort.

******************************************************************************
* Copyright (c) 2004-2021 New H3C Technologies Co., Ltd. All rights reserved.*
* Without the owner's prior written consent,                                 *
* no decompiling or reverse-engineering shall be allowed.                    *
******************************************************************************

<MSR-1>display ssh server status
<MSR-1>display local-user
<MSR-1>display current-configuration configuration line
```
进行截图保存即可.