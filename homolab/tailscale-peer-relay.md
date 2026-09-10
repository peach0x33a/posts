---
title: Tailscale Peer Relay —— 拯救你枣糕的 Tailnet 网络！
date: 2026-09-07T08:16:59+08:00
# weight: 1
# aliases: ["/first"]
tags: ["Tailscale", "homelab"]
author: "Me"
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
description: "介绍Tailscale Peer Relay的作用、如何配置、注意事项"
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

博客又长草啦。最近也没啥可写的，正好前几天把 Tailscale Peer Relay 给配了，体验还不错，顺手写一下~

部分内容来自于互联网、朋友、群友。

> [!Tip] 在编写博客的文章时，将默认读者有流畅访问 Github / Google 等网站的能力。

## 基本名词

开篇介绍一下：

- 家宽： 家庭宽带环境
- Tailnet： Tailscale官方对于Tailscale上的网络的称呼。Tailnet，直译为`尾网`
- DERP：全称为 `Designated Encrypted Relay for Packets`，在Tailscale环境下指中继节点。

## Peer Relay 是什么？什么情况下需要它？

[Tailscale](https://tailscale.com/) 是个好东西，不过本文不会赘述 Tailscale 如何安装配置，只讲与 Peer Relay 相关的东西。

不过，在弄懂 Peer Relay 之前，还是需要对 Tailscale 的工作方式有一点认知的，不想看的话，可以选择跳过：[开始配置 Peer Relay](#配置-peer-relay)。

在两台设备建立连接时，Tailscale 会先通过 DERP 服务器建立初始连接，然后在后台不断尝试升级路径。优先级是：**直连 > Peer Relay > DERP**。一旦条件允许，Tailscale 会自动切换到更优的路径。

聪明的朋友此时已经发现了问题——大内网环境下，打洞基本不可能成功，有的网络环境还要更加复杂，例如学校和公司。这些公用网络通常有很多层 NAT 或者不同区域的 Vlan 隔离，把本就不高的打洞成功率变得更低了！而由于合规性原因，Tailscale 官方无法在大陆部署 DERP 节点，只能用到官方内置的境外 DERP。这样一来，哪怕是两台物理距离只有1km的设备，他们之间的数据交互也白白出海转了一圈，延迟直升 300ms。这太让人难受了~

Peer Relay 解决的就是这个让人难受的场景。在没有 Peer Relay 之前，你只能自建 DERP 节点，但这条路也挺折腾的。由于 DERP 协议是建立在HTTPS上的，天生不支持纯 IP 访问，需要域名 + TLS证书。而大陆的服务器，域名走标准端口跑 TLS 流量，不备案直接阻断。至于家宽公网，`80` `443`等标准端口早就在 ISP 的黑名单上了，想都不用想。你当然可以改 derper 源码来使用纯 IP 访问你的自建 DERP ，但……这很麻烦，不是吗。

Peer Relay的出现解决了这个问题——你不再需要绑定域名了。Peer Relay允许两台设备通过它使用 UDP 来转发已经加密好的 WireGuard包，不需要 TLS，不需要域名，只需要开一个 UDP 端口。而既然这台设备能进入你的 Tailnet，本身就证明了它足够安全，也就无须额外的身份认证了。

那么问题来了：**Peer Relay 节点需要公网 IP 吗？**

> *本回答由 Claude Opus 4.6生成*
>
> 严格来说，不是硬性要求，但**强烈建议有**。Peer Relay 启动后会通过 STUN 探测自己的公网 `IP:port`，然后广播给 Tailnet 里的其他节点。如果你的 NAT 类型比较友好（Full Cone NAT / NAT1），STUN 探测到的地址确实是从外部可达的，理论上能工作。但问题在于，这个映射不在你的控制之下——端口可能会变，IP 也可能会漂移，稳定性没法保证。所以结论是：**NAT1 环境下能用但不稳，有公网 IP 才是靠谱的方案。**

如果你家有公网 IP 的话，你还能省下一台中继服务器的钱！甚至顺带降低延迟——数据到了你家公网开放的端口以后就直接进内网了，不用再绕出去转一圈。举个例子，以前可能是 A地(你的位置,离你家不远) → B地(DERP) → A地(你家,公网开放端口) → 内部设备，现在数据不需要去外地绕一圈再回家了~

在动手之前，最好先确认一下你的中继节点到底有没有公网 IP，这里分两种情况：

- **家宽环境**：由于三大运营商对个人用户停发公网IP，路由器/光猫拿到的其实是内部私有地址，外面又套了一层电信级 NAT（CGNAT），而且这层 NAT 你完全无法控制。确认方法很简单，在想运行中继节点的设备上跑一下：

  ```bash
  curl -4 ifconfig.me
  ```

  再去路由器/光猫的后台看一眼"WAN 口信息"里显示的 IP，两者一致就说明是真公网 IP，可以正常做端口转发；如果不一致（路由器显示的是 `100.x.x.x`、`10.x.x.x` 这类私有地址段），说明是 CGNAT，端口转发这条路走不通，只能考虑换一台有真公网 IP 的机器做中继，或者放弃 Peer Relay 回去优化 DERP 路径。

- **云服务器/VPS**：即使你没买独立 IP，只要你能在控制台里自己配置端口映射、且映射关系是稳定的，就不受影响——跟家宽 CGNAT 的区别在于，这层映射是不是你能控制、会不会莫名其妙变动。但需要注意：云服务器厂商的使用策略是否允许 `WireGuard` 流量。

没有公网 IP 的话，仍然建议使用一台中继服务器，在中继服务器上运行 Peer Relay 还是有收益的。你不需要域名、TLS、也不需要备案。~~不过可能要小心被服务商查水表...？毕竟是 WireGuard 流量。~~

## 配置 Peer Relay

讲了这么多，是时候开始配置了！由于我家里有公网 IP，我就用家宽公网环境做演示了，服务器环境其实大差不差~命令行部分也是一样的，Windows的GUI也有等价操作。

### 前提条件

- 所有相关设备的 Tailscale 版本 ≥ **1.86**
- 中继节点不能是 iOS、Apple TV 或 Android 上运行的 Tailscale
- 客户端没有系统限制，iOS 也能用
- 中继节点需要有至少一个可用的 **UDP 端口**，且该端口能被其他设备访问到
- 你的 Tailscale 账户拥有 Owner / Admin / Network admin 权限

### Step 1：在中继节点上启用 Peer Relay

在你选定的中继设备上执行一行命令：

```bash
tailscale set --relay-server-port=6199
```

端口号随便，原则上建议用大数字随机端口，我使用了`6199`，本文就用`6199`了。

### Step 2：让外部能连上这个端口

#### 方案 A：直接访问（有公网 IP）

如果你搭建中继节点的设备本身就有公网 IP（比如云服务器），或者所在的路由器拿到了公网 IP（比如家庭宽带），直接用就可以了。如果这台设备在路由器后面，需要把发到路由器这个端口的 UDP 流量转发到内网 IP，本机防火墙也要记得放行：

```bash
# ufw
sudo ufw allow 6199/udp

# firewalld
sudo firewall-cmd --add-port=6199/udp --permanent && sudo firewall-cmd --reload

# iptables
sudo iptables -A INPUT -p udp --dport 6199 -j ACCEPT
```

#### 方案 B：没有公网 IP，借 FRP 转发一下

**注意：本节只是提供一个思路，我没有实测过是否真正可行，但是理论上是完全可行的！不过请注意FRP服务的使用策略是否允许`WireGuard`流量。也不会推荐任何 FRP 厂商。**

如果你实在搞不到公网 IP ，可以试试本节的思路：

大概是这样：在 relay 节点上运行 FRP 客户端，添加一条 UDP 类型的隧道，把relay的端口通过 FRP 服务穿透出去，然后查看 FRP 客户端的日志是否回报了穿透后的地址，再告诉 Tailscale "我的公网地址其实是这个"：

```bash
tailscale set --relay-server-port=<你的端口> \
  --relay-server-static-endpoints="frps的公网IP:映射端口"
```

这样 Tailscale Peer Relay 就不再需要公网了。而是直接把你填写的内网穿透地址广播给Tailnet内的其他节点。流量路线图大致如下：

```text
Tailnet 内的其他设备 → FRP节点 → 你的FRP客户端 → 内网设备
```

重要的事情说三遍：

**请注意使用的 FRP 服务是否允许`WireGuard`流量，账号被封本人概不负责！**

**请注意使用的 FRP 服务是否允许`WireGuard`流量，账号被封本人概不负责！**

**请注意使用的 FRP 服务是否允许`WireGuard`流量，账号被封本人概不负责！**

### Step 3：配置 ACL 策略

进入 [Tailscale 管理后台 - Access Controls](https://console.tailscale.com/admin/acls/file)，
切换到 JSON editor，做两件事：

**1. 定义 Tag**

在 `tagOwners` 中给中继节点定义一个标签：

```jsonc
"tagOwners": {
    "tag:relay": ["autogroup:admin"]
}
```

保存后，去 [Machines 页面](https://console.tailscale.com/admin/machines) 找到
中继设备，给它打上 `tag:relay`。

**2. 添加 Grant 规则**

在 `grants` 部分添加中继授权：

```jsonc
"grants": [
    {
        "src": ["autogroup:member"],
        "dst": ["tag:relay"],
        "app": {
            "tailscale.com/cap/relay": []
        }
    }
]
```

`src` 是"谁可以使用中继"，`dst` 是"哪些设备是中继节点"。上面的写法是让所有成员都能用，你也可以换成更精细的 tag 来控制范围。

⚠️ Tailscale 官方的建议： `src` 不要写 `*`，否则所有设备都会尝试走 Peer Relay，可能导致意料之外的流量路由。一般来说，`src` 应该是那些位置固定、网络环境差（打洞困难）的设备，而不是经常换网络的手机和笔记本。

#### Step 4：验证

从另一台设备 ping 一下：

```bash
tailscale ping <目标设备名或Tailscale IP>
```

如果看到 `via peer-relay(...)`，说明中继已生效：

```text
pong from my-nas (100.x.x.x) via peer-relay(1.2.3.4:6199:vni:xxxx) in 13ms
```

如果还是 `via DERP(tok)`，逐项排查：

1. 中继节点和客户端的 Tailscale 版本是否都 ≥ 1.86
2. Tag 是否正确打上了
3. Grant 规则是否保存成功
4. UDP 端口是否放行（本机防火墙 + 路由器转发都要检查，或者检查FRP隧道类型是否错误）

也可以用 `tailscale status` 看全局连接状态：

```bash
tailscale status | grep peer-relay
```

### 关闭 Peer Relay

不需要了？先把端口设为空：

```bash
tailscale set --relay-server-port=""
```

然后去[Tailscale - Console](https://console.tailscale.com/admin/)，把 ACL 里那条 `relay` 的 `grants` 规则删掉，顺便把设备上的 `tag:relay` 标签也摘掉，就算彻底清干净了。

## 踩坑记录

配置过程看起来只有三步，但实际操作中每一步都有可能翻车。以下是我自己踩过的坑，
按排查顺序列出来，遇到问题时从上往下逐项检查：

### Grant 规则没加 / 没生效

这是最容易被忽略的一步。即使你在管理后台给设备打了 `tag:relay`，机器页面也显示了
"Peer Relay" 的徽章——**这只代表这台设备"具备资格"，不代表其他设备有权使用它**。

你需要在 ACL 策略里显式添加一条 `grants` 规则：

```jsonc
"grants": [
    {
        "src": ["autogroup:member"],
        "dst": ["tag:relay"],
        "app": {
            "tailscale.com/cap/relay": []
        }
    }
]
```

注意，这条 `grants` 和你原来的 `acls`（网络连通性规则）是**两个完全独立的维度**，
即使 `acls` 里已经 `* -> *:*` 全部放行，也不会自动带来 relay 权限。

如何验证 Grant 是否生效？在客户端上执行：

```bash
tailscale whois <中继节点的Tailscale IP>
```

看输出末尾，有没有出现：

```text
Capabilities:
  - tailscale.com/cap/relay-target
```

出现了就代表，Tailscale 认为这个 IP 具有peer-relay的能力，不再是白板一块了。
没有这一行就说明 Grant 还是没配对，回去检查 ACL 策略。

### 路由器端口转发选错了协议、主机防火墙没放行

Peer Relay 只产生 **UDP** 流量，**不走 TCP**。如果你在路由器上加端口转发规则时选成了 TCP...你会知道后果的，也许看到这里你也应该反应过来了。回去改协议类型吧。

路由器转发对了，不代表流量就能进的去系统。老生常谈之**防火墙策略**——记得放行哦。

~~别问我怎么知道的这些，一定不是我手快忘记改协议和忘记放行防火墙了。~~

### UPnP 不会自动帮 Peer Relay 端口做映射

即使你的路由器支持 UPnP / NAT-PMP / PCP（`tailscale netcheck` 里 `PortMapping`
显示正常），Tailscale **目前不会帮 Peer Relay 端口做 UPnP 映射**——它只会
自动映射常规的 magicsock 主端口（41641）。Peer Relay 的端口必须手动去路由器上
加转发规则。所以请务必手动添加端口转发！

这个是已知的功能缺口

### 排查顺序

总结一下，如果配置完了 `tailscale ping` 还是显示 `via DERP(...)`，按这个顺序查：

1. **版本**：两端 Tailscale 是否 ≥ 1.86
2. **Tag**：中继节点打上 `tag:relay` 的标签了吗
3. **Grant**：ACL 里有 `tailscale.com/cap/relay` 的授权吗（用 `tailscale whois` 验证）
4. **端口监听**：中继节点上 `ss -ulnp | grep <端口>` 能看到 tailscaled 在监听吗
5. **主机防火墙**：`firewall-cmd --query-port=<端口>/udp` 或 `ufw status` 放行了吗
6. **路由器转发**：外部 UDP 端口有没有转发到内网中继节点（注意协议要选 **UDP**）
