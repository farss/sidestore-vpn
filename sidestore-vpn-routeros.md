# SideStore 局域网刷新：原理、OrbStack，以及 RouterOS NAT

用 SideStore 刷新 / 安装应用时，iOS 会去连一台假的「电脑」`10.7.0.1`。平时这件事由手机上的 StosVPN 或 WireGuard 做：把发往 `10.7.0.1` 的包源/目的 IP 对调，再送回手机，让 SideStore 自己接到。

[xddxdd/sidestore-vpn](https://github.com/xddxdd/sidestore-vpn) 把这套逻辑搬到局域网里的一台 Linux 上，这样整网 iOS 设备都可以不挂手机 VPN 就刷新。

本文说明：

- 这个工具在干什么
- 为什么默认装在 OrbStack 里基本不能用
- RouterOS 的 `dstnat` + `srcnat` 能不能等价
- 那两条 NAT 规则逐包在干什么

---

## 1. 手机上其实是两条 TCP

对调的不是「把原连接原地改个地址」，而是让手机协议栈看到两套四元组。端口不对调，只换 IP。

```
连接 A（iOS 当客户端，去连假电脑）
  phone:12345  →  10.7.0.1:54321

连接 B（SideStore 当服务端，被假电脑连进来）
  10.7.0.1:12345  →  phone:54321
```

中间盒把发给 `10.7.0.1` 的包对调后丢回去，两条连接在 IP 层对上。

`sidestore-vpn` 并没有在用户态 `connect()` 到手机。它建一个 TUN，吃掉发往 `10.7.0.1` 的包，交换源/目的 IP，再发出去。手机收到的是一个 **SYN**，`accept()` 的是连接 B。

作者后来那条能覆盖整网的 nftables 规则是**无状态改 IP 头**：

```nft
table ip sidestore {
  chain NAT_PREROUTING {
    type nat hook prerouting priority -350; policy accept;
    ip daddr 10.7.0.1 ip daddr set ip saddr ip saddr set 10.7.0.1 notrack
  }
}
```

每个发往 `10.7.0.1` 的包都当场对调，不依赖连接跟踪。RouterOS **没有** `ip saddr set` / `ip daddr set` 这种动作。

---

## 2. 环境约束

必须同时满足：

- 跑逻辑的那一端是 Linux（或路由器自己能做等价改包）
- 和 iOS 同一 LAN
- 中间不能有状态 NAT（对调后的回程包看起来像从 `10.7.0.1` 主动打到 iPhone 的新连接）
- 若不是跑在路由器上，要加静态路由：`10.7.0.1/32 → 那台机器`

官方 Docker：

```bash
docker run --rm \
  --cap-add=NET_ADMIN \
  --network=host \
  --device /dev/net/tun:/dev/net/tun \
  ghcr.io/xddxdd/sidestore-vpn
```

### OrbStack

二进制能装、TUN 也能建，默认网络下大概率不能用。

| 官方假设 | OrbStack 实际情况 |
|---|---|
| `--network=host` = Linux 主机的 LAN | host 是 OrbStack 自己的 Linux VM，再 NAT 到 Mac |
| 机器有局域网 IP | Machine 一般是 `198.19.x.x`，出网靠 NAT |
| TUN 以 `10.7.0.1` 为源直达 iPhone | 回程会被 OrbStack / macOS NAT 改源地址 |

「Expose ports to LAN」只对监听端口的服务有用。这个工具是 TUN 改包。桥接到 Wi-Fi 仍是未完成的功能请求。

正经做法：一台真正在局域网里的 Linux（树莓派，或 UTM / Parallels **桥接**网卡），RouterOS 只加：

```routeros
/ip route add dst-address=10.7.0.1/32 gateway=<那台Linux的LAN IP>
```

---

## 3. RouterOS 能不能自己改包？

**不能做成和 nftables 一样的东西。** 最多用 conntrack NAT 去赌等价效果。

| | sidestore-vpn / nftables | RouterOS dstnat + srcnat |
|---|---|---|
| 改包方式 | 无状态改 IP 头 + `notrack` | 连接跟踪 NAT |
| 整网一条规则自动对调 | 可以 | 做不到，必须一台手机写死一个 IP |
| 手机是否收到来自 `10.7.0.1` 的 SYN | 是 | 可以是 |
| SideStore 是否 `accept` 一条新连接 | 是 | 可以是 |
| 中间盒有没有自己的 TCP socket | 没有 | 没有 |
| 中间盒怎么记账 | 每包对调 | 一条 conntrack：去程 NAT，回程反 NAT |

「他要发起一条打到手机的 TCP」——对手机成立，对中间盒不成立。两种做法都是改包，不是路由器去 `connect` 手机。

**不要**在 raw 里对这批流量 `notrack`。RouterOS 的 NAT 完全依赖连接跟踪；`notrack` 之后包根本不进 `/ip firewall nat`。

未在 RouterOS 上用 SideStore 验证过。下面规则是实验，不是推荐的生产方案。失败就换 LAN 上的 Linux + `sidestore-vpn`。

---

## 4. 实验用 NAT 规则

假设 iPhone 是 `192.168.88.50`（改成实际地址，并尽量做 DHCP lease 绑定）。

```routeros
/ip firewall filter
add chain=forward dst-address=10.7.0.1 action=accept place-before=0 \
    comment="sidestore skip fasttrack"
add chain=forward src-address=10.7.0.1 action=accept place-before=0 \
    comment="sidestore skip fasttrack"

/ip firewall nat
add chain=dstnat src-address=192.168.88.50 dst-address=10.7.0.1 \
    action=dst-nat to-addresses=192.168.88.50 comment="sidestore dnat"
add chain=srcnat src-address=192.168.88.50 dst-address=192.168.88.50 \
    action=src-nat to-addresses=10.7.0.1 comment="sidestore snat"
```

注意：

- **不要**把 `10.7.0.1` 配成路由器自己的地址，否则包会进 input，不会转发出去。
- `srcnat` 必须放在所有可能误伤 LAN 流量的 masquerade **上面**。
- 多台 iPhone 就复制一对 NAT，改 `src-address` / `to-addresses`。
- 默认 `out-interface-list=WAN` 的 masquerade 不会碰到从 LAN 出去的包；太宽的 `src-address=192.168.88.0/24 action=masquerade` 会。

---

## 5. 两条 NAT 规则详细说明

这两条要一起看。它们分别在包走路由器的**两个不同阶段**改地址，合起来才等于「把源和目的对调」。

假设 SideStore 监听 `54321`，iOS 随机源端口 `12345`。

### 5.1 各自干什么

```routeros
# 规则 1：目的地址改写（dstnat 链，prerouting，进路由器后立刻执行）
add chain=dstnat src-address=192.168.88.50 dst-address=10.7.0.1 \
    action=dst-nat to-addresses=192.168.88.50

# 规则 2：源地址改写（srcnat 链，postrouting，马上要出接口前执行）
add chain=srcnat src-address=192.168.88.50 dst-address=192.168.88.50 \
    action=src-nat to-addresses=10.7.0.1
```

| | 规则 1 dstnat | 规则 2 srcnat |
|---|---|---|
| 何时看 | 包刚进路由器，还没做路由决定 | 已经决定从哪个口出去，马上要发出去 |
| 匹配什么 | 「这台手机发给 10.7.0.1」 | 「源和目的都已经是这台手机」（这是规则 1 改完后的样子） |
| 改什么 | 目的 `10.7.0.1` → `192.168.88.50` | 源 `192.168.88.50` → `10.7.0.1` |
| 端口 | 不动 | 不动 |

RouterOS 的 NAT **只处理每条连接的第一个包**。后面的包（包括手机回的 SYN-ACK）不再走这两条规则，而是由连接跟踪按「去程怎么改的」自动反着改。

### 5.2 包在路由器里的顺序

```
手机把包交给网关
        │
        ▼
  prerouting
  ── 规则 1 dstnat 在这里开火 ──
        │
        ▼
  路由决定（看「现在的目的 IP」该从哪走）
        │
        ▼
  forward 防火墙
        │
        ▼
  postrouting
  ── 规则 2 srcnat 在这里开火 ──
        │
        ▼
  从 LAN 口发出去
```

规则 1 改完目的之后，路由才看目的地址。目的已经变成手机自己，所以包不会进路由器的 input，而是 **hairpin 转发出 LAN**，再被规则 2 改源地址。

`dstnat` 必须先于路由；`srcnat` 必须后于 dstnat。这是 ROS 固定顺序，不是你写规则的先后。写规则的先后只影响同一条链里和别的 NAT 谁先匹配。

### 5.3 第一个包：iOS 的 SYN

iOS 要连假电脑，发出：

```
192.168.88.50:12345  →  10.7.0.1:54321     SYN
```

`10.7.0.1` 不在手机网段里，所以这个包会交给默认网关，也就是 RouterOS。

**规则 1 匹配**

```
src-address=192.168.88.50    ✓ 就是这台手机
dst-address=10.7.0.1         ✓ 就是假电脑
→ dest 改成 192.168.88.50
```

包变成：

```
192.168.88.50:12345  →  192.168.88.50:54321     SYN
```

端口没动。目的端口 `54321` 仍是 SideStore 的监听端口。源和目的已经是同一个 IP——这只是包在路由器内部的过渡态，还没出接口。

**路由**

目的是 `192.168.88.50`，在 LAN 上：转发，从 LAN 口出去，ARP 到手机的 MAC。

**规则 2 匹配**

规则 2 匹配的**不是**手机刚发出时的样子，而是 **规则 1 改完之后** 的样子。所以这条规则不能单独读，它是专门接规则 1 的。

匹配后源改成 `10.7.0.1`：

```
10.7.0.1:12345  →  192.168.88.50:54321     SYN
```

手机协议栈看到的是：`10.7.0.1` 用端口 `12345` 来连我的 `54321`。SideStore 正在听 `54321`，于是 `accept` 一条**新的入站 TCP**。

连接跟踪此时记下（简化）：

```
原始：    192.168.88.50:12345 → 10.7.0.1:54321
改后：    10.7.0.1:12345     → 192.168.88.50:54321
回程应是：192.168.88.50:54321 → 10.7.0.1:12345
```

### 5.4 回程：SideStore 的 SYN-ACK

SideStore 回：

```
192.168.88.50:54321  →  10.7.0.1:12345     SYN-ACK
```

这个包**不会再走那两条 NAT 规则**。连接跟踪一看：这正是上面记的「回程应是」，于是按去程反过来改：

- 去程把目的 `10.7.0.1` 改成了手机 → 回程把**源**从手机改回 `10.7.0.1`
- 去程把源从手机改成了 `10.7.0.1` → 回程把**目的**从 `10.7.0.1` 改回手机

得到：

```
10.7.0.1:54321  →  192.168.88.50:12345     SYN-ACK
```

这正是 iOS 那条「连假电脑」的连接在等的 SYN-ACK。之后的 ACK、数据、FIN 都走这张表。

整条连接在手机上是两条 TCP，在路由器上是**一条** NAT 会话。

三次握手对得上：

| 步骤 | 线上的包 | 谁在看 |
|---|---|---|
| 1 | `phone:12345 → 10.7.0.1:54321` SYN seq=X | iOS 发出连接 A |
| 2 | `10.7.0.1:12345 → phone:54321` SYN seq=X | SideStore `accept` 连接 B |
| 3 | `phone:54321 → 10.7.0.1:12345` SYN-ACK seq=Y ack=X+1 | SideStore 回连接 B |
| 4 | `10.7.0.1:54321 → phone:12345` SYN-ACK seq=Y ack=X+1 | iOS 收到连接 A 的 SYN-ACK |

步骤 2、4 是 NAT / 反 NAT 的产物。序号、标志位不变，conntrack 眼里这也是一条正常的客户端/服务端 TCP。

### 5.5 为什么规则要写成这样

**规则 1 为什么要写 `src-address=192.168.88.50`**

`to-addresses` 只能写死一个 IP，不能写「改成这个包的源地址」。所以必须一台手机一条规则，`to-addresses` 和 `src-address` 是同一个地址。  
不写 `src-address` 的话，局域网里任何设备访问 `10.7.0.1` 都会被改到这台手机上。

**规则 1 为什么必须是 dstnat、不能是 srcnat**

目的要在**路由之前**改掉。若等到 postrouting 再改目的，路由器早就按 `10.7.0.1` 做完路由了：这个地址既不是自己的、也没有路由的话，包直接丢，根本到不了 srcnat。

**规则 2 为什么匹配 `src=dst=手机`**

srcnat 发生在 dstnat **之后**。到 postrouting 时，目的已经是手机了，源还是手机，所以必须按「改完后的包」来写匹配。  
如果规则 2 仍写成 `dst-address=10.7.0.1`，到 srcnat 时目的已经不是 `10.7.0.1`，规则永远匹配不上，发出去的包就会是：

```
192.168.88.50:12345 → 192.168.88.50:54321
```

手机看到「我自己连我自己」，SideStore 对不上 iOS 心里那条连 `10.7.0.1` 的连接。

**规则 2 为什么要把源改成 `10.7.0.1`**

1. 手机看到的对端必须是 `10.7.0.1`，SideStore / iOS 才认。
2. 如果不改源，发出去源还是 `192.168.88.50`，手机以为是自己发给自己，回包也不会走网关。
3. 源改成 `10.7.0.1` 后，手机回 `10.7.0.1` 仍会交给网关，回程才能再被 conntrack 抓住。

### 5.6 只开一条会怎样

只开规则 1：

```
发出去：192.168.88.50:12345 → 192.168.88.50:54321
```

源没改。手机当本地环回或直接丢掉，假电脑的身份没了。

只开规则 2：

规则 2 要等「源和目的都是手机」。没有规则 1，目的一直是 `10.7.0.1`，规则 2 不匹配。`10.7.0.1` 若不是路由器自己的地址、又没有路由，包在路由阶段就丢了。

### 5.7 串起来

```
iOS 客户端 socket                    SideStore 监听 socket
phone:12345 → 10.7.0.1:54321         0.0.0.0:54321

        │ SYN                                  ▲
        ▼                                      │ 手机收到的 SYN
   规则1 改目的                                 │ 10.7.0.1:12345 → phone:54321
   phone → phone                               │
        │                                      │
   规则2 改源 ─────────────────────────────────┘
   10.7.0.1 → phone

        ▲                                      │ SYN-ACK
        │ 手机收到的 SYN-ACK                    │ phone:54321 → 10.7.0.1:12345
        │ 10.7.0.1:54321 → phone:12345         │
   conntrack 反 NAT ◄──────────────────────────┘
   （两条规则不再执行）
```

规则 1+2 只负责去程第一个包；回程整段都是连接跟踪「按记账反着改」。效果上可以等价于 nftables 的「每个发往 `10.7.0.1` 的包都对调」，但实现完全不是无状态改头。

---

## 6. 仍然可能翻车的地方

这些和「是不是新 TCP」无关，是 RouterOS 自己的检查：

1. **dstnat 之后、srcnat 之前**，forward 里会短暂变成 `phone:12345 → phone:54321`。默认 `drop invalid`、fasttrack 有可能在这里动手。
2. NAT 表只处理每条连接的第一个包，后面全靠 conntrack 反 NAT。状态机一旦标 invalid，后面的 SYN-ACK / ACK 会被丢。nftables 那条 `notrack` 没有这个问题。
3. 必须给每台手机写死 IP。
4. 没有公开发表过 SideStore + RouterOS 的成功配置。hairpin NAT 文档针对的是「访问公网 IP 回到内网服务器」，不是「改完地址打回自己」。

---

## 7. 怎么验证

不要只看 NAT 计数。

```routeros
/ip firewall nat print stats
/ip firewall connection print where dst-address~"10.7.0.1"
/tool sniffer quick ip-address=10.7.0.1
```

手机关掉 StosVPN / WireGuard，再在 SideStore 里刷新。嗅探里应该先后看到：

1. `phone:xxxxx → 10.7.0.1:监听端口` SYN
2. `10.7.0.1:xxxxx → phone:监听端口` SYN（打进手机的那条）
3. `phone:监听端口 → 10.7.0.1:xxxxx` SYN-ACK
4. `10.7.0.1:监听端口 → phone:xxxxx` SYN-ACK

2 和 4 都有，才说明手机真的看到了两条 TCP。只有 1、没有 2，就是 NAT 没把 SYN 打回去。一边通、一边不通，或 `connection-state=invalid` 在涨，就别再磨了。

局域网任意设备 `ping 10.7.0.1` **不会**像跑了 `sidestore-vpn` 那样对所有人通：规则是按手机 IP 写的，只有那台手机的包会被改。

---

## 8. 建议怎么选

- iPhone 就 1～3 台、能绑静态 IP：可以按第 4 节做一次短实验。
- 实验失败，或设备多、IP 不固定：LAN 上的 Linux + `sidestore-vpn`，RouterOS 只加 `/32` 路由。
- 不要把默认 NAT 的 OrbStack 当第一选择。

Linux 上跑工具：

```bash
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p

docker run -d --name sidestore-vpn --restart=unless-stopped \
  --cap-add=NET_ADMIN \
  --network=host \
  --device /dev/net/tun:/dev/net/tun \
  ghcr.io/xddxdd/sidestore-vpn
```

---

## 参考

- [xddxdd/sidestore-vpn](https://github.com/xddxdd/sidestore-vpn)
- [Using SideStore without StosVPN across your LAN](https://lantian.pub/en/article/modify-computer/sidestore-without-stosvpn-across-lan.lantian/)
- [StosVPN PacketTunnelProvider](https://github.com/SideStore/StosVPN/blob/main/TunnelProv/PacketTunnelProvider.swift)
- MikroTik：raw `notrack` 之后包不进 NAT 表（NAT 依赖连接跟踪）
