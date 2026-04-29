# CF-Workers-SoftEther

基于 Cloudflare Workers 的 SoftEther / SSTP 出站实验项目。它不是普通的 `WebSocket → connect() → 目标` 字节转发，而是在 Worker 内连接 SSTP 服务端，完成 SSTP、PPP 和部分 IPv4/TCP 栈逻辑，再把 VLESS/WebSocket 流量转发到目标地址。

> 这是协议研究和学习代码，用来探索受限边缘运行时中能否重建一条虚拟网络通路。实际代理体验不保证稳定，性能和可用性取决于 SSTP/SoftEther 服务端、PPP 分配、目标网络和 Cloudflare Workers 运行时限制。

## 项目定位

普通 Worker 代理通常是：

```text
WebSocket → connect() → 目标 TCP
```

`Softether.js` 的实验路径是：

```text
VLESS over WebSocket
  → Cloudflare Worker
  → SSTP over TLS
  → PPP 协商
  → 虚拟 IPv4 地址
  → 手工 IPv4/TCP 封包
  → 目标 TCP
```

核心意义不是“又写了一个代理”，而是验证：在没有完整系统网络栈、没有 TUN/TAP、没有原生 VPN 客户端的 Workers 环境中，是否可以用脚本重建一条可用的虚拟通信路径。

## 公共服务器来源

主要可用服务器来自 VPN Gate 公共中继列表。VPN Gate 是筑波大学维护的公共 VPN Relay 项目，服务器由志愿者提供，列表会动态变化。

- VPN Gate 官网：<https://www.vpngate.net/>
- VPN Gate 英文页：<https://www.vpngate.net/en/>
- 公共 Relay 服务器列表：<https://www.vpngate.net/en/volunteer_servers.aspx?number=0>

`Softether.js` 的 `/sstp://host:port` 通常指向这些支持 MS-SSTP / SoftEther 的公共节点。节点质量、可用性和出口位置会随 VPN Gate 列表实时变化。

## 功能特性

- 支持 VLESS over WebSocket 入口。
- 支持 `/sstp://host:port` 指定 SSTP/SoftEther 服务端。
- 使用 Workers `connect()` 建立 TLS 出站连接。
- 实现 SSTP `SSTP_DUPLEX_POST` 建链。
- 实现 PPP 协商：LCP / PAP / IPCP。
- 获取 PPP 分配的虚拟 IPv4 地址。
- 手工构造 IPv4 / TCP 包。
- 手工处理 TCP `SYN / ACK / PSH / FIN`。
- 计算 IPv4 / TCP 校验和。
- 通过 WebSocket 与手工 TCP 栈做双向桥接。

## 文件结构

| 文件 | 说明 |
| --- | --- |
| [Softether.js](./Softether.js) | Worker 主实现：VLESS 入口、SSTP、PPP、IPv4/TCP 封包和转发。 |

## 协议层次

```text
客户端
  VLESS over WebSocket
    ↓
Cloudflare Worker
  SSTP over TLS
    ↓
SoftEther / SSTP Server
  PPP
    ↓
虚拟 IPv4 链路
  手工 IPv4/TCP
    ↓
目标 TCP 服务
```

## SSTP / SoftEther 流程

`Softether.js` 连接 SSTP 服务端后，会发起类似下面的 HTTP over TLS 请求：

```text
SSTP_DUPLEX_POST /sra_{...}/ HTTP/1.1
Host: <sstp-host>
Content-Length: 18446744073709551615
SSTPCORRELATIONID: {...}
```

然后进入 SSTP 数据帧和 PPP 协商阶段：

```text
SSTP Connect
  → PPP LCP Configure-Request / Ack
  → PAP Authenticate-Request / Ack
  → IPCP Configure-Request / Ack
  → 获取虚拟 IPv4 地址
```

之后 Worker 不再只是搬运字节，而是在脚本里生成 IPv4/TCP 包并通过 PPP 送入虚拟链路。

## 手工 TCP 路径

建立目标 TCP 连接时，代码会在 Worker 内完成：

```text
SYN
  ← SYN+ACK
ACK
PSH+ACK(data)
FIN+ACK
```

并维护：

- 源端口；
- TCP sequence / acknowledgement；
- IPv4 header；
- TCP header；
- IPv4 checksum；
- TCP pseudo-header checksum；
- MSS 分段。

这也是本项目区别于普通 Worker TCP 转发的核心部分。

## 节点格式

Worker WebSocket path 使用：

```text
/sstp://sstp_host:port?ed=2560
```

VLESS 分享链接模板：

```text
vless://<uuid>@<front-host>:<front-port>/?type=ws&encryption=none&host=<worker-host>&path=%2Fsstp%3A%2F%2F<sstp_host>%3A443%3Fed%3D2560&security=tls&sni=<worker-host>&fp=chrome&packetEncoding=xudp#SSTP
```

字段说明：

| 字段 | 说明 |
| --- | --- |
| `<uuid>` | 与 `Softether.js` 中的 `uuid` 一致。 |
| `<front-host>:<front-port>` | 客户端连接入口。 |
| `host` / `sni` | Worker 域名。 |
| `path` | URL 编码后的 `/sstp://sstp_host:port?ed=2560`。 |
| `<sstp_host>:443` | SSTP / SoftEther 服务端地址。 |
| `packetEncoding=xudp` | 客户端侧参数；本实现主要处理 VLESS TCP 流。 |

## 与普通 Worker TCP 代理对比

| 能力 | 普通 Worker TCP 代理 | Softether.js |
| --- | --- | --- |
| VLESS/WebSocket 入口 | ✅ | ✅ |
| `connect()` 直连目标 | ✅ | 非核心 |
| SSTP 建链 | ❌ | ✅ |
| PPP 协商 | ❌ | ✅ |
| 虚拟 IPv4 地址 | ❌ | ✅ |
| 手工 IPv4/TCP 封包 | ❌ | ✅ |
| TCP 序号 / ACK 维护 | ❌ | ✅ |
| 协议研究价值 | 低 | 高 |

## 适合场景

- SoftEther / SSTP 协议路径研究。
- PPP / IPCP / IPv4 / TCP 脚本实现参考。
- 边缘运行时虚拟链路实验。
- 受限环境中协议重组和网络栈最小化验证。

## 已知限制

- 这是协议实验项目，不是稳定 VPN 客户端。
- 只模拟必要的 IPv4/TCP 路径，不是完整系统网络栈。
- 只适合 TCP 目标；UDP 不在当前主线内。
- 兼容性取决于 SSTP/SoftEther 服务端行为。
- 长连接、大流量和复杂网页加载可能受 Workers 限制影响。
- 代码中的认证、参数和路径需要按实际环境自行调整。

## 相关链接

- 开源协议：[GPL-3.0](./LICENSE)
- SoftEther VPN 官网：<https://www.softether.org/>
- SoftEther GitHub：<https://github.com/SoftEtherVPN/SoftEtherVPN>
- VPN Gate 官网：<https://www.vpngate.net/>
- VPN Gate 公共服务器列表：<https://www.vpngate.net/en/volunteer_servers.aspx?number=0>
- 频道 / 交流群组：<https://t.me/Enkelte_notif>
