# Linux Moonlight 串流选路配置（多网卡主机强制走有线链路）

两台机器之间存在两条互通的 L3 路径时，Moonlight 会在你"明明填了有线地址"的情况下走 WiFi。本文记录从现象到根因（客户端源码里的地址优先级 + 服务端 `serverinfo` 覆盖 + mDNS 重置）再到修复与验证的完整过程，环境为 x18（客户端）→ X19（Sunshine 服务端）。

> [!warning] 文中的 IP、网卡名、主机名、时间戳均为本环境实测值，照抄前必须替换成自己机器的值。

---

## 一、问题现象

同一对主机（x18 ⇄ X19）之间有两条彼此独立的路径：

| 路径 | x18 侧 | X19 侧 | 链路 | 实测（2026-10-23，`ping -c 20 -i 0.2`） |
| --- | --- | --- | --- | --- |
| 有线直连 | `enp44s0` 10.42.0.76/24 | `enp3s0` 10.42.0.1/24 | 千兆 | 0% 丢包，RTT min/avg/max = 0.051 / 0.356 / 0.884 ms |
| WiFi 同网 | `wlan0` 10.112.241.226/24 | `wlp2s0` 10.112.241.1/24 | 802.11n | **45% 丢包**，RTT 6.324 / 88.625 / 203.319 ms |

症状：在 Moonlight 里手动给主机 X19 填了 `10.42.0.1`，实际串流走的是 `10.112.241.1`（WiFi），画质抖动、卡顿。

> 2026-10-21 首次记录时 WiFi 一路是 0% 丢包、avg ≈ 44 ms、抖动到 86 ms；2026-10-23 复测已恶化到 45% 丢包。无线链路的劣化幅度本身就不稳定，这正是必须强制走有线的原因。

---

## 二、根因

三个机制叠加，缺一不可：

1. **地址优先级里"手动地址"排最后。**
   `moonlight-qt 6.2.0` 的 `app/backend/nvcomputer.cpp` 中 `uniqueAddresses()` 的选取顺序是：
   ```
   activeAddress → localAddress → remoteAddress → ipv6Address → manualAddress
   ```
   手动填的地址只是兜底，前面的都拿不到才轮到它。所以"我配了 10.42.0.1 它却走 10.112.241.1"不是 bug。

2. **`localAddress` 不由用户决定，而是被服务端覆盖。**
   每次轮询主机时，`localAddress` 会被 Sunshine `/serverinfo` 返回的 `<LocalIP>` 覆盖（`update()` 里的 `ASSIGN_IF_CHANGED_AND_NONNULL(localAddress)`）。而 `<LocalIP>` 实际是**该次请求在本机落到的源地址**，实测：

   ```bash
   for ip in 10.42.0.1 10.112.241.1; do
     printf '%s -> ' "$ip"
     curl -s -m 8 "http://$ip:47989/serverinfo" | grep -o '<LocalIP>[^<]*</LocalIP>'
   done
   # 10.42.0.1     -> <LocalIP>10.42.0.1</LocalIP>
   # 10.112.241.1  -> <LocalIP>10.112.241.1</LocalIP>
   ```

   即：从哪个口去问，它就回哪个口。

3. **mDNS 每次启动都会重置 `localAddress`。**
   启动时 mDNS 重新解析并 `addNewHost(mdns=true)`，把 `localAddress` 重设成主机名解析出的**第一个 IPv4**（本环境是 WiFi 那口）。所以**只改 `localaddress` 而不关 mDNS，下次启动又会被打回 WiFi。**

结果是自我强化：首次命中哪个地址由 mDNS 发现顺序决定，之后启动时 `activeAddress` 为空，先用 `localAddress`，即上次停住的那口。

---

## 三、定位过程（含假说与否证）

| 假说 | 否证证据 |
| --- | --- |
| ① 手动地址没写进去 / 写错 | `Moonlight.conf` 里 `1\manualaddress=10.42.0.1`、`manualport=47989` 均在 |
| ② 系统路由选错出口 | `ip route get 10.42.0.1` → `dev enp44s0 src 10.42.0.76`，路由正确 |
| ③ Moonlight 会测速/测 RTT 自动选最快 | 读源码：选路只做"取第一个能应答的地址"，无任何 RTT/带宽/丢包测量 |
| ④ 服务端 Sunshine 的 `LocalIP` 是写死的本机地址 | 用两个源地址分别请求，返回值随之改变（见上） |

真因只在下钻源码后成立：地址优先级 + `LocalIP` 覆盖 + mDNS 重置三者共同作用。

---

## 四、修复

**配置文件**：`~/.config/Moonlight Game Streaming Project/Moonlight.conf`

**配置内容**（主机段只列相关键）：

```ini
[General]
mdns=false

1\hostname=X19
1\localaddress=10.42.0.1
1\localport=47989
1\manualaddress=10.42.0.1
1\manualport=47989
1\remoteaddress=39.144.95.132
1\remoteport=47989
1\ipv6address=<X19 的公网 IPv6>
1\ipv6port=47989
```

两处改动就够了：

- `mdns=false` —— 图形界面里对应"设置 → 关闭 mDNS 主机发现"。**这一步不能省**，否则每次启动被重置回 WiFi。
- `1\localaddress=10.42.0.1` —— 直接把首选地址钉成有线口。

`remoteaddress`（STUN 公网地址，在本机走代理 TUN）与 `ipv6address`（走 WiFi）**保留作兜底，但不再被优先命中**，不要删。

> [!warning] 必须先**退出** Moonlight 再改配置。运行中改、或从运行中退出，都会把内存里的值回写文件，改动被覆盖。

---

## 五、验证

```bash
# 1) 后台启动并看 stderr，首次轮询应直接走有线
moonlight 2>&1 | grep 'online at'
# "X19" is now online at "10.42.0.1:47989"     ← 2026-10-21 实测输出

# 2) 确认 localAddress 已被自我纠正并持久化
grep '^1\\localaddress=' "$HOME/.config/Moonlight Game Streaming Project/Moonlight.conf"
# 1\localaddress=10.42.0.1                      ← 2026-10-23 复核仍为此值

# 3) 重启 Moonlight 后 2) 仍为 10.42.0.1（说明 mdns=false 生效）
```

`localAddress` 会在首轮轮询后自我纠正为 `10.42.0.1` 并落盘，因此重启不变。

> 第 1 步的 `online at` 行只在真正启动客户端时才有；纯配置文件检查用第 2 步即可，不需要拉图形界面。

---

## 六、上游能力缺口

`moonlight-qt 6.2.0` 并不具备"多宿主选最快路径"的能力：

- 选路只做"第一个能应答"，**没有** RTT / 带宽 / 丢包测量，也**不周期性重选**。
- 唯一的"探测"是 `NvComputer::getActiveAddressReachability()` 的一次 TCP connect（3 s 超时）+ 看本地 NIC 类型/MTU/点对点。它只用于 `session.cpp` 判断 `streamingRemotely` 和包大小（1024 vs 1392），**不参与选地址**。
- `detectnetblocking` 只在"添加主机"时用 STUN 测 UDP 端口是否被封，同样不参与选址。

相关上游 issue：#869（要求按主机指定网络 / 多地址按顺序尝试）、#914（多网卡行为疑问）。若将来上游支持多地址尝试或 RTT 选路，可以撤掉 `mdns=false` 让发现机制回归。

---

## 七、方法论沉淀

1. **客户端里"手动填的地址"可能是兜底而非优先。** 遇到"配了却不生效"，先去源码确认地址优先级，再怀疑配置。
2. **客户端展示的主机地址可能被服务端响应覆盖。** 验证方法很便宜：用不同源地址请求同一接口，看返回是否随之变化。
3. **发现机制（mDNS/SSDP/DHCP 选项）会在每次启动重置推导值。** 改完固定值必须同时关掉对应的自动发现，否则"改一次好一次，重启就回退"。
4. **多宿主主机判断优劣要看丢包和抖动，不只看连通性。** 两条路径 `ping` 都通，但一路 45% 丢包——只看"能 ping 通"会得出完全相反的结论。
5. **交互式工具的验证不能靠伪造输入。** 本机的路径确认靠 `online at` 日志行与配置文件落盘值，而不是靠截图/猜测。

---

相关笔记：[[Linux-免硬件显卡虚拟诱骗器(EDID固件模拟)配置指南]]（无头/远程串流场景下让服务端有虚拟显示器）
