## 元认知：从传统 VPN 到 Mesh VPN 的范式转变

传统 VPN（OpenVPN、IPsec）是星型拓扑：所有节点连到中心服务器，节点间通信经服务器中转。这带来三个问题：

```
节点 A ──→ 中心服务器 ←── 节点 B
         ↑
  1. 服务器宕机 = 全断（单点故障）
  2. 所有流量过服务器（带宽瓶颈）
  3. 两端都在 NAT 后？服务器做中继（延迟翻倍）
```

Mesh VPN 改变了一个根本假设：**节点之间可以直接通信，不需要中转**。拓扑从星型变成全连接：

```
节点 A ←──直连──→ 节点 B
  ↕                  ↕
节点 C ←──直连──→ 节点 D
```

但"Mesh"只是一个拓扑概念。实现 Mesh 的方式可以完全不同——有的用 WireGuard，有的用自研协议，有的甚至不建隧道（零信任）。**选 Mesh VPN 不是选"哪个最好"，是选"哪种协议 + 哪种拓扑 + 哪种故障策略适合你的场景"。**

要理解 13 个方案的差异，需要五个原理维度。

---

## 搭积木：五个原理维度

### 维度一：底层协议——WireGuard 为什么赢了

| 协议派系 | 代表方案 | 原理特点 |
|---------|---------|---------|
| WireGuard | Tailscale / NetBird / Netmaker / EasyTier / Firezone / InnerNet | 内核态、4000 行代码、UDP、Noise 协议框架 |
| 自研协议 | ZeroTier（L2）/ Nebula（Noise）/ Tinc / Yggdrasil | 各有侧重：L2/去中心化/纯 P2P |
| 零信任 | OpenZiti / NetFoundry | 不建隧道，做应用层身份访问 |

13 个方案中，**6 个基于 WireGuard**。这不是巧合——WireGuard 在协议层面赢了，原因有三：

**1. Noise 协议框架：1-RTT 握手**

WireGuard 用 Noise IK 模式做密钥协商——**1 个往返（1-RTT）完成握手**，之后数据直接加密传输。对比 IPsec 的 IKEv2 需要 2-3 个往返，WireGuard 的连接建立快一个数量级。

整个协议只有 **4 种消息类型**（握手发起、握手回应、cookie、传输数据）。代码量约 4000 行——OpenVPN 超过 10 万行，IPsec 的 StrongSwan 超过 50 万行。代码少意味着可审计、漏洞少、攻击面小。

**2. 内核态：性能接近裸线**

WireGuard 在 Linux 内核态运行——数据包不需要在用户态和内核态之间拷贝。这是它"最快"的根本原因。后续会看到，Tailscale 用的是 `wireguard-go`（用户态 Go 实现），有额外拷贝开销，直连时差异在 5% 内，但理论上不如内核 WireGuard 快。

**3. UDP-only：避免 TCP-in-TCP 塌陷**

WireGuard 只用 UDP。这是刻意设计——如果在 TCP 上封装加密隧道，TCP 的拥塞控制会和内层 TCP 冲突（TCP-in-TCP），性能会塌陷到原来的 1/10。

> 但 UDP-only 有代价：**UDP 被封锁的网络里，WireGuard 直接失效**。这是为什么 Tailscale 发明了 DERP fallback（后面讲）。

Nebula 也用 Noise 协议框架，但不是 WireGuard——它自研了基于 Noise 的控制面（Lighthouse），加密用 AES-256-GCM。Slack 用它在生产环境跑了 5 万+ 节点。

### 维度二：网络层级——L2 vs L3 vs 应用层

| 层级 | 方案 | 能做什么 | 不能做什么 |
|------|------|---------|----------|
| L2（链路层） | ZeroTier | 广播、多播、非 IP 协议、二层发现 | 性能略低 |
| L3（IP 层） | Tailscale / NetBird / Netmaker / Nebula / Firezone / InnerNet | IP 路由、TCP/UDP | 广播、非 IP 协议 |
| 应用层 | OpenZiti | 身份级访问、零信任 | 不做网络层打通 |

**90% 的场景 L3 够用。** 但 L2 有不可替代的刚需场景：

- **mDNS/Bonjour 跨网段**：Apple 设备的 AirDrop、HomeKit 用 mDNS 发现，L3 的 IP 路由跨不过子网边界。只有 L2 能让两地的设备在同一个广播域里互相发现
- **老旧工业协议**：某些工业控制协议（Modbus 广播、EtherCAT）依赖二层广播
- **游戏局域网发现**：部分老游戏的局域网联机用二层 ARP 发现

> ZeroTier 是 13 个方案中**唯一的主流 L2 方案**。它封装的是以太网帧（不只是 IP 包），多了 L2 头部开销，但换来了广播能力。这就是为什么即使有 WireGuard，ZeroTier 仍有不可替代的位置。

OpenZiti 的"应用层"和 L2/L3 是正交的——它不解决"网络通不通"，解决的是"谁能访问什么应用"。它的 Zero Trust 模型把**身份**而非 **IP** 作为网络的核心：没有开放端口，服务对互联网不可见，只有认证身份才能访问。

### 维度三：拓扑方式——控制面 vs 数据面

这是理解稳定性差异的关键维度。**控制面**（谁能加入、密钥分发）和**数据面**（实际流量走哪）是两个独立的维度：

| 拓扑 | 控制面 | 数据面 | 代表方案 |
|------|-------|-------|---------|
| 混合 | 中心服务器 | P2P 直连 | Tailscale / ZeroTier / NetBird |
| 去中心 | Lighthouse | P2P 直连 | Nebula |
| 完全 P2P | 无 | P2P | Yggdrasil / Tinc |

Tailscale 的巧妙：**控制面中心化（简单），数据面 P2P（快）**。协调服务器只做密钥分发和节点发现，不参与数据转发。一旦节点间建立直连，流量不经过 Tailscale 的服务器。

> 但控制面依赖 Tailscale 的 SaaS——这是 Headscale（自建替代）存在的理由。Headscale 让你享受 Tailscale 的客户端体验，但控制面完全自控。

Nebula 的 Lighthouse 不是传统服务器——它是"协调节点"，只帮助节点互相发现 + UDP 打洞。Lighthouse 宕机**不影响已建立的 P2P 连接**，但**新连接建立会受影响**。可以配多个 Lighthouse 实现冗余。

Yggdrasil 是真正的无中心——纯 ad-hoc P2P 路由，没有任何"协调节点"。代价是没有统一管理入口，配置靠手动 peer。

### 维度四：NAT 穿透与稳定性——哪个最稳定

这是最实际的维度：**在真实网络环境下，哪个方案最不会断？**

NAT 穿透是 Mesh VPN 的核心技术挑战。Tailscale 的研究数据揭示了真实情况：

| NAT 组合 | 穿透方法 | 成功率/耗时 |
|---------|---------|-----------|
| 一端 Easy NAT（EIM） | STUN + 同时发包 | ~100% |
| 一端 Hard NAT（对称 NAT） | birthday attack | 50% 需 174 次探测（~2 秒）；99.9% 需 2048 次（~20 秒） |
| 两端 Hard NAT | birthday attack | 99.9% 需 170,000 次（~28 分钟）→ 通常放弃 |

> **只需路径中有一个 Hard NAT，简单穿透即失败。** Hard NAT 常见于企业路由器和云 NAT 网关；家用路由器多是 Easy NAT。

基于这个现实，各方案的稳定性差异如下：

| 方案 | P2P 成功率 | Fallback 机制 | 稳定性评价 |
|------|-----------|-------------|-----------|
| **Tailscale** | **>90%** | DERP 全球 relay（HTTPS 伪装，DERP 无法解密流量） | **最稳定**：即使 UDP 被封，DERP 走 HTTPS 仍通 |
| Nebula | 依赖 UDP 打洞 | **无明确 fallback** | 可控网络下稳定；Hard NAT 场景可能连不上 |
| ZeroTier | 高（root servers 辅助） | Root servers 中转 | 多 root 冗余，较稳定 |
| OpenZiti | Fabric 智能路由 | 多 Router 冗余 | 架构韧性，但依赖 Fabric 部署 |
| Yggdrasil | 自愈网络 | 无 fallback 概念 | 无单点，但 **alpha 阶段**不稳定 |

> **Tailscale 最稳定的根本原因**：DERP（Designated Encrypted Relay for Packets）是"保底连通"机制。它用 HTTP/HTTPS 流传输——即使网络封锁了所有 UDP（如企业 guest Wi-Fi），DERP 仍能走 HTTPS 端口通。DERP 服务器**无法解密流量**——私钥永不离开节点，DERP 只盲目转发已加密数据包。

> 这是 Tailscale 的双层保险：超过 90% 的节点对能 P2P 直连（快），剩余不到 10% 由 DERP 保底连通（慢但通）。**99.9% 的场景都能连通**——这就是"最稳定"的工程含义。

Nebula 没有 DERP 类 fallback。如果两端的 NAT 都是 Hard 类型，Nebula 可能**根本连不上**——没有 fallback 机制兜底。在可控网络环境（如云服务器之间、自建数据中心）Nebula 极其稳定，但在不可控的互联网环境，它的连通性不如 Tailscale。

Yggdrasil 明确标注自己 **still alpha-stage**，代码未经正式安全审计，"不应在 mission-critical 工作负载上运行"。它的自愈能力是真的，但整体稳定性不足以用于生产。

### 维度五：性能对比——哪个最快

| 方案 | 吞吐量 | 延迟 | 连接建立 | 性能瓶颈 |
|------|--------|------|---------|---------|
| WireGuard（内核态） | **最高**（接近裸线） | μs 级开销 | 快 | 无 |
| Tailscale（wireguard-go） | 略低于内核 WireGuard（~5% 内） | 直连低；DERP 有中继延迟 | >90% 秒级；Hard NAT 2-20s | 用户态拷贝开销 |
| Nebula（Go + AES-256-GCM） | 高 | 低 | 依赖 Lighthouse 发现 | Go 运行时开销 |
| ZeroTier（L2 封装） | L2 开销略高 | 略高 | 快 | 以太网帧封装 |
| Yggdrasil | MTU 65535 优化 | 自愈快 | 需配 peer | alpha 阶段不稳定 |
| OpenZiti | Fabric 路由优化 | 未公布具体数字 | 依赖 Tunneler 模式 | 应用层处理 |

**WireGuard 最快的根本原因**：
1. **内核态运行**——数据包不在用户态和内核态之间拷贝
2. **代码量 4000 行**——无冗余逻辑，执行路径短
3. **只做加密传输**——不做策略、不做 ACL、不做日志，这些都在上层处理

> WireGuard 本身不是一个完整的 Mesh VPN 方案——它是**数据平面协议**。Tailscale / NetBird / Nebula 都构建在 WireGuard（或类似协议）之上，差异主要在控制平面和 NAT 穿透层，而非加密传输本身。

**Tailscale 略慢的原因**：用 `wireguard-go`（用户态 Go 实现），有用户态↔内核态拷贝开销。但**直连时差异在 5% 内**——对大多数应用不可感知。DERP fallback 时延迟显著增加（中继跳数 + HTTPS 封装），但这是"慢但通"的兜底，不是常态。

**ZeroTier 的 L2 代价**：封装的是以太网帧（不只 IP 包），多了 L2 头部开销。实际吞吐量比 L3 方案低约 5-15%（估计值，取决于场景），但换来了广播能力——这是 L2 方案的固有 tradeoff。

> **纯内核 WireGuard 最快。在完整方案中，Nebula 和内核 WireGuard 方案最快（无用户态开销），Tailscale 直连时接近，DERP fallback 时最慢。**

---

## 案例即原理：四个场景的最佳方案

### 场景一：个人自用（2-5 台设备）

| 需求 | 最佳方案 | 理由 |
|------|---------|------|
| 最省心 | **Tailscale**（官方 SaaS） | 5 分钟组网，>90% 直连，DERP 保底，免费额度够用 |
| 想自建 | **Tailscale + Headscale** | 同客户端体验，控制面自控，数据不出门 |
| 极客玩家 | **Yggdrasil** | 完全 P2P，体验另一种网络哲学，但 alpha 不稳定 |

> 个人场景的核心诉求是"能用、不折腾"。Tailscale 的 SaaS 方案安装即用，DERP 保底意味着你几乎不会遇到"连不上"的情况。如果介意数据经过 Tailscale 的协调服务器，Headscale 是自建替代——同样的客户端，控制面自己控。

### 场景二：小团队（10-50 台设备）

| 需求 | 最佳方案 | 理由 |
|------|---------|------|
| 跨地点协作 | **Tailscale / NetBird** | ACL + MagicDNS，按人授权 |
| 需要 Web 管理后台 | **NetBird / Firezone** | Web UI 比 CLI 更适合团队协作 |
| 需要 L2 广播 | **ZeroTier** | 唯一的 L2 主流方案，跨网段 mDNS |

> 团队场景的核心诉求是"可管理"。Tailscale 的 ACL 按用户/标签授权（"dev 团队能访问 staging，不能访问 production"），MagicDNS 让你用 `db.internal` 而非 `10.0.0.5` 访问服务。NetBird 和 Firezone 提供自建的 Web 管理后台，适合不想依赖 SaaS 的团队。

### 场景三：极客/去中心化

| 需求 | 最佳方案 | 理由 |
|------|---------|------|
| 不依赖任何中心 | **Yggdrasil** | 完全 P2P，自分配 IPv6，但 alpha 阶段 |
| Slack 同款 | **Nebula** | 5 万节点生产验证，Lighthouse 架构 |
| 国产/中文社区 | **EasyTier** | 文档中文，兼容多平台 |

> 极客场景的核心诉求是"不依赖第三方"。Yggdrasil 是最极端的选择——没有任何中心节点，每个节点既是客户端又是路由器。但代价是 IPv6 only、会为他人路由流量、不稳定。Nebula 是更务实的选择——Lighthouse 是你自己的服务器，但你只负责发现，不负责数据中转。

### 场景四：特殊场景

| 需求 | 最佳方案 | 理由 |
|------|---------|------|
| L2 广播/多播 | **ZeroTier** | 唯一主流 L2，mDNS/工业协议刚需 |
| 零信任应用层 | **OpenZiti** | 身份即网络，服务无开放端口 |
| K8s 跨集群 | **Netmaker** | 原生 K8s 支持，大规模节点 |
| 老牌极端稳定 | **Tinc** | 20 年历史，协议成熟 |

> OpenZiti 是 13 个方案中最特殊的一个——它不是传统 VPN 的替代品，是**安全模型的升级**。传统 VPN 的逻辑是"连上内网就能访问内网"，OpenZiti 的逻辑是"deny by default，只有明确授权的身份才能访问特定应用"。服务没有开放端口，对互联网扫描工具不可见。如果安全比连通更重要，OpenZiti 是正确选择。

---

## 缺陷与批判：每个方案的硬伤

### Tailscale

- **控制面默认在 SaaS**：依赖 Tailscale 公司的协调服务器。Headscale 是自建替代，但功能滞后官方（新特性如 Funnel、Serve 需要等社区适配）
- **wireguard-go 用户态**：比内核 WireGuard 慢约 5%。对性能敏感场景（如大文件传输）可感知
- **DERP 不是免费的**：免费用户的 DERP 流量有带宽限制

### ZeroTier

- **L2 性能开销**：封装以太网帧的代价，吞吐量比 L3 方案低 5-15%
- **Moon/Controller 自建文档少**：官方文档偏向 SaaS 方案，自建 Controller 和 Moon 节点需要翻社区帖子
- **免费额度限制**：免费版限 25 个设备

### Nebula

- **无 DERP 类 fallback**：Hard NAT 场景可能连不上——没有兜底机制
- **Lighthouse 单点**：虽然可配多个，但文档没有详细的高可用配置指南
- **无 Web UI**：纯配置文件管理，不适合非技术用户

### Yggdrasil

- **官方明确 alpha 阶段**：不应在生产环境使用
- **未经安全审计**：加密实现用了 Go 标准库，但代码库无正式外部审计
- **会为他人路由流量**：你的节点会为其他 Yggdrasil 用户中转流量——流量受限的用户需注意
- **IPv6 only**：只传输 IPv6 流量，部分老旧应用不兼容

### OpenZiti

- **学习曲线陡**：不是传统 VPN 替代品，需要理解身份、服务、策略、Tunneler 等概念
- **不是网络层打通**：如果你需要"ping 对方 IP"，OpenZiti 不是正确选择——它做的是应用级访问，不是 IP 级连通

### "完全自建"的陷阱

> 自建不等于更好。自建意味着**你是运维**——你要负责可用性、版本升级、密钥轮换、故障排查。Tailscale 的 DERP 全球 relay 网络是 Tailscale 公司在维护，你自建 NetBird 时，relay 服务器的高可用是你的责任。

> 选择"完全自建"之前，问自己：**我有没有能力在凌晨 3 点排查 relay 服务器为什么连不上？** 如果没有，SaaS 方案更务实。

---

## 总结：Mesh VPN 是什么

Mesh VPN 不是一种技术，是**拓扑概念**的升级——从星型（中心中转）到全连接（节点直连）。

三个关键选型轴：

| 轴 | 选项 | 决策依据 |
|----|------|---------|
| 协议 | WireGuard / 自研 / 零信任 | 除非要 L2 或零信任，选 WireGuard |
| 层级 | L2 / L3 / 应用层 | 除非要广播，选 L3 |
| 控制面 | 中心 / 去中心 / 完全 P2P | 省心选中心（Tailscale），可控选去中心（Nebula） |

**最稳定**：Tailscale。DERP 保底 + >90% 直连 + 全球 relay。即使 UDP 被封锁仍能通。代价是控制面依赖 SaaS。

**最快**：纯内核 WireGuard。完整方案中 Nebula 最快（无用户态开销 + AES-256-GCM 硬件加速）。Tailscale 直连时接近，DERP fallback 时最慢。

**不可替代**：ZeroTier（唯一 L2）/ OpenZiti（唯一零信任）/ Yggdrasil（唯一完全 P2P）。

> 硬件画线（内核态 vs 用户态决定性能上限），拓扑分层（控制面 vs 数据面决定稳定性），穿透保底（DERP 决定最坏情况下的连通性）。选 Mesh VPN 不是选"最好"的，是选"恰好够用"的——一个在 Hard NAT 后面仍能连通的 Tailscale，比一个理论最快但连不上的方案更有价值。
