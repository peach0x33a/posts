---
title: "关于 Proxmox 自身研究"
date: 2024-11-17T01:50:12+08:00
# weight: 1
# aliases: ["/first"]
tags: ["Proxmox","ProxmoxVE"]
author: "Me"
# author: ["Me", "You"] # multiple authors
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
description: "关于我本人对ProxmoxVE的一些研究"
disableHLJS: false
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

本处用于记载本人对 ProxmoxVE 虚拟化系统 的一些研究

部分内容来自于互联网、朋友、群友。

> [!Tip] 在编写博客的文章时，将默认读者有流畅访问 Github / Google 等网站的能力。

## ProxmoxVE 更换主机名

> 在本例中主机名从 `s3` 迁移到 `tc-s3-pve`

首先修改主机名

> 只有主机名有关,域名后面的部分都无所谓

```bash
hostnamectl set-hostname xxx-pve.mihari.top
# 一样可以
hostnamectl set-hostname xxx-pve
```

还需要编辑 hosts 保证解析

!!!! 注意拼写 不要打错 !!!!

```txt
123.123.123.123 xxx-pve.mihari.top xxx-pve
```

先备份整个 pmxcfs

```bash
cp -R /etc/pve ./pve
```

或者你更愿意打包备份

```bash
# 打包为当前 年-月-日-时-分-秒 的压缩包
tar -zcvf pve-config-backup-$(date +%Y-%m-%d-%H-%M-%S).tar.gz /etc/pve
```

然后在 pmxcfs 里面搬配置文件

> 这里只迁移了 qemu-server

```bash
cd /etc/pve
mv nodes/s3/qemu-server/* nodes/tc-s3-pve/qemu-server
```

接下来重启 pve-cluster 服务保证所有文件正确链接  
还需要重新 pveproxy 保证访问正常  
最后重启 pvestatd 来保证状态收集正常

```bash
systemctl restart pvedaemon
systemctl restart pveproxy
systemctl restart pve-cluster
systemctl restart pvestatd
```

## ProxmoxVE 删除集群

```bash
# 查看所有的集群
ls /etc/pve/nodes
```

1.  停止节点上的corosync和PVE集群服务

```bash
systemctl stop pve-cluster #PVE集群服务
systemctl stop corosync #corosync
```

2.  将集群文件系统设置为本地模式
    
```bash
pmxcfs -l
```
    
3.  删除corosync配置文件

```bash
rm /etc/pve/corosync.conf
rm -r /etc/corosync/*
```

4.  重新启动PVE集群服务

```bash
killall pmxcfs
systemctl start pve-cluster
```