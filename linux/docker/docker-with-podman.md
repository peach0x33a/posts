---
title: Docker With Podman
date: 2025-12-18T14:45:52+08:00
# weight: 1
# aliases: ["/first"]
tags: [""]
author: "Me"
showToc: true
TocOpen: false
draft: true
hidemeta: false
comments: false
description: "本文介绍如何使用Podman + DockerCLI的奇妙组合"
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

本文介绍如何使用Podman + DockerCLI的奇妙组合

## 正文


### STEP 1

首先安装podman，本文不再废话

然后docker，不需要安装他的全部组建，只需要安装以下三个
- docker-ce-cli         (命令行)
- docker-compose-plugin (docker compose)
- docker-buildx-plugin  (docker buildx)

### STEP 2

安装完成后，不急于启动podman，首先需要给`podman.socket`写服务覆盖，同时确保当前用户在wheel组内
```bash
# /etc/systemd/system/podman.socket.d/02-podman-user-access.conf
[Socket]
#SocketUser=wheel
SocketGroup=wheel

# /etc/systemd/system/podman.socket.d/03-podman-dir-user-access.conf
[Socket]
SocketUser=root
SocketGroup=wheel
DirectoryMode=0755
ExecStartPost=/usr/bin/chmod -R 775 /run/podman
```

