# RouterOS 实现 SideStore 局域网刷新

SideStore 刷新或安装应用时，iOS 会向 `10.7.0.1` 发起 TCP，把对端当成一台运行开发者工具的电脑。SideStore 同时在手机本机打开对应端口，模拟那台电脑上的服务。

RouterOS 用两条 NAT 规则，在网关处把发往 `10.7.0.1` 的包源、目的 IP 对调后打回手机。手机协议栈因此看到两条 TCP，端口不变，只换 IP：

```
连接 A（iOS 当客户端，去连假电脑）
  phone:12345  →  10.7.0.1:54321

连接 B（SideStore 当服务端，被假电脑连进来）
  10.7.0.1:12345  →  phone:54321
```

路由器上这是一条 NAT 会话：去程改地址，回程由连接跟踪反向还原。

下文以 iPhone `192.168.88.50` 为例。换成实际地址，并做 DHCP lease 绑定。

---

## 命令

```routeros
/ip firewall nat
add chain=dstnat src-address=192.168.88.50 dst-address=10.7.0.1 \
    action=dst-nat to-addresses=192.168.88.50 comment="sidestore dnat"
add chain=srcnat src-address=192.168.88.50 dst-address=192.168.88.50 \
    action=src-nat to-addresses=10.7.0.1 comment="sidestore snat"
```

不要把 `10.7.0.1` 配成路由器自己的地址。多台 iPhone 就按 IP 再写一对。`srcnat` 放在会匹配 LAN 流量的 masquerade 上面。

---

## 包在路由器里的顺序

```
手机把包交给网关
        │
        ▼
  prerouting
  ── dstnat：改目的地址 ──
        │
        ▼
  路由决定（看改完后的目的 IP）
        │
        ▼
  forward
        │
        ▼
  postrouting
  ── srcnat：改源地址 ──
        │
        ▼
  从 LAN 口发回手机
```

`dstnat` 在路由之前执行，`srcnat` 在出接口之前执行。这是 RouterOS 固定顺序。NAT 表只处理每条连接的第一个包；之后的包（含回程 SYN-ACK）由连接跟踪按去程记录反向改写。

---

## 规则说明

### dstnat

```routeros
chain=dstnat
src-address=192.168.88.50
dst-address=10.7.0.1
action=dst-nat
to-addresses=192.168.88.50
```

匹配「这台手机发给 `10.7.0.1`」，把目的改成这台手机自己。

- 必须在 prerouting 改目的。若拖到出接口再改，路由器会按 `10.7.0.1` 做路由：该地址不是本机、又没有路由时，包在路由阶段被丢，到不了 `srcnat`。
- `to-addresses` 只能写死 IP，不能写成「改成这个包的源地址」，所以一台手机一条规则，`src-address` 与 `to-addresses` 相同。
- 不写 `src-address` 时，局域网任意设备访问 `10.7.0.1` 都会被改到这台手机。
- 端口不改。目的端口仍是 SideStore 的监听端口。

改完后包在路由器内部是：

```
192.168.88.50:12345  →  192.168.88.50:54321
```

源、目的暂时相同。路由看到目的是 LAN 上的手机，于是 hairpin 从 LAN 转发出去，而不是进 input。

### srcnat

```routeros
chain=srcnat
src-address=192.168.88.50
dst-address=192.168.88.50
action=src-nat
to-addresses=10.7.0.1
```

匹配的是 **dstnat 改完之后** 的包，不是手机刚发出时的样子。把源改成 `10.7.0.1`。

若这里仍写 `dst-address=10.7.0.1`，到 postrouting 时目的已经不是 `10.7.0.1`，规则匹配不上，发出去会变成手机连自己。

源必须改成 `10.7.0.1`：

1. 手机看到的对端必须是 `10.7.0.1`，SideStore 和 iOS 才认。
2. 源若不改，手机把回包当本地通信，不会再交给网关。
3. 源改成 `10.7.0.1` 后，手机回 `10.7.0.1` 仍走网关，回程才能被连接跟踪接住。

默认 `out-interface-list=WAN` 的 masquerade 碰不到从 LAN 出去的包。若存在过宽的规则，例如 `src-address=192.168.88.0/24 action=masquerade`，必须排在本条之后，否则源会被改成路由器 LAN 地址。

---

## 去程第一个包（SYN）

iOS 发出：

```
192.168.88.50:12345  →  10.7.0.1:54321     SYN
```

`10.7.0.1` 不在手机网段，包交给默认网关。

1. dstnat：目的 `10.7.0.1` → `192.168.88.50`  
   `192.168.88.50:12345 → 192.168.88.50:54321`
2. 路由：目的在 LAN，从 LAN 转发。
3. srcnat：源 `192.168.88.50` → `10.7.0.1`  
   `10.7.0.1:12345 → 192.168.88.50:54321`

手机收到来自 `10.7.0.1:12345`、打向本机 `54321` 的 SYN。SideStore 在该端口监听，`accept` 一条入站 TCP。

连接跟踪记录：

```
原始：    192.168.88.50:12345 → 10.7.0.1:54321
改后：    10.7.0.1:12345     → 192.168.88.50:54321
回程应是：192.168.88.50:54321 → 10.7.0.1:12345
```

---

## 回程（SYN-ACK 及之后）

SideStore 回：

```
192.168.88.50:54321  →  10.7.0.1:12345     SYN-ACK
```

两条 NAT 规则不再执行。连接跟踪按去程反向改写：

- 去程把目的从 `10.7.0.1` 改成手机 → 回程把源从手机改回 `10.7.0.1`
- 去程把源从手机改成 `10.7.0.1` → 回程把目的从 `10.7.0.1` 改回手机

得到：

```
10.7.0.1:54321  →  192.168.88.50:12345     SYN-ACK
```

这是连接 A 在等的 SYN-ACK。之后的 ACK、数据、FIN 走同一张表。TCP 序号和标志位不被 NAT 改写。

```
iOS 客户端                                 SideStore 监听
phone:12345 → 10.7.0.1:54321               0.0.0.0:54321

        │ SYN                                     ▲
        ▼                                         │
   dstnat 改目的                                   │ 10.7.0.1:12345 → phone:54321
   phone → phone                                  │
        │                                         │
   srcnat 改源 ───────────────────────────────────┘
   10.7.0.1 → phone

        ▲                                         │ SYN-ACK
        │ 10.7.0.1:54321 → phone:12345            │ phone:54321 → 10.7.0.1:12345
   conntrack 反 NAT ◄─────────────────────────────┘
```

---

## 只开一条的后果

只开 dstnat，发出去源仍是手机，对端不是 `10.7.0.1`，假电脑身份不成立。

只开 srcnat：该规则要等「源和目的都是手机」。没有 dstnat 时目的一直是 `10.7.0.1`，srcnat 不匹配；`10.7.0.1` 无路由则包在路由阶段丢掉。

---

## 使用

1. 按手机 IP 写入上述 NAT。
2. 手机关闭 StosVPN / WireGuard。
3. 用 SideStore 刷新。

```routeros
/ip firewall nat print stats
/ip firewall connection print where dst-address~"10.7.0.1"
/tool sniffer quick ip-address=10.7.0.1
```

刷新时应依次看到：

1. `phone:xxxxx → 10.7.0.1:监听端口` SYN
2. `10.7.0.1:xxxxx → phone:监听端口` SYN
3. `phone:监听端口 → 10.7.0.1:xxxxx` SYN-ACK
4. `10.7.0.1:监听端口 → phone:xxxxx` SYN-ACK
