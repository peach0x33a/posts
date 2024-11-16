---
title: "关于 Docker 自身研究"
date: 2024-11-17T01:47:57+08:00
# weight: 1
# aliases: ["/first"]
tags: ["Docker"]
author: "Me"
# author: ["Me", "You"] # multiple authors
showToc: true
TocOpen: false
draft: true
hidemeta: false
comments: false
description: "关于我本人对 Docker 的一些研究"
disableHLJS: false # to disable highlightjs
disableShare: false
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

本处用于记载本人对docker的一些研究

部分内容可能来自互联网、群友、朋友


## docker-args | Docker 启动参数

为了避免```/etc/docker/daemon.json```中的设置与```/usr/lib/systemd/system/docker.service```中的启动参数冲突，安插一个服务覆盖将其清理，所有配置来自于JSON，即单一事实来源理论

```bash
cat /etc/systemd/system/docker.service.d/90-no-args.conf
[Service]
# 此处需要先清理一下,才能做到覆盖
ExecStart=
ExecStart=/usr/bin/dockerd%
```

## docker-logrotate | Docker 日志滚动
默认情况下， 容器的 stdout 和 sdterr 写在 `/var/lib/docker/containers/[container-id]/[container-id]-json.log` 中的 json 中, 没有维护的情况下将会占用大量的磁盘空间

所以需要做日志 rotation , 修改配置文件 `/etc/docker/daemon.json`

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "50m",
    "max-file": "5"
  }
}
```

然后重启应用

```
$ systemctl daemon-reload

$ systemctl restart docker
```
