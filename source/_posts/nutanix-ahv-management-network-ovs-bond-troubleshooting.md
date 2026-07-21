---
title: Nutanix 节点服务起不来：从 br0、OVS bond 到 VLAN 的网络排查思路
date: 2026-07-21
tags:
  - Nutanix
  - AHV
  - OVS
  - 网络排障
  - 虚拟化
categories:
  - 知识笔记
description: 记录 Nutanix AHV 节点服务异常时，如何判断管理网走哪块物理网卡、br0 是否正常、OVS bond 当前 active slave 是否指向错误链路，以及如何用 ARP 和抓包定位二层/VLAN 问题。
---

在 Nutanix AHV 里排查节点服务起不来时，很容易先盯着 Genesis、Cassandra、Stargate 这些组件。但如果某个节点连其他节点都到不了，问题往往不是服务本身，而是底层网络不通。尤其是 AHV 使用 OVS 网桥和 bond 时，`ip a` 看到 `br0 UP` 并不等于网络真的正常。

<!-- more -->

## 核心结论

这类问题要先分清三件事：

- 管理 IP 通常挂在 `br0` 上，而不是直接挂在 `eth0` 或 `eth4` 上。
- `br0` 下面可能接的是单个物理口，也可能是一个 OVS bond，例如 `br0-up`。
- 如果 bond 当前 active slave 选到了交换机侧 VLAN 不通的网卡，就会出现“接口 UP、IP 正常、服务却起不来”的现象。

一句话：

```text
接口状态正常 ≠ 网络互通正常
```

真正要验证的是：`br0` 下面接了什么、bond 当前实际走哪块物理口、ARP 是否能通、交换机 VLAN/Trunk/Access 是否一致。

## 先判断管理 IP 在哪里

在 AHV Host 上执行：

```bash
ip a
```

如果看到类似：

```text
br0: inet 192.168.14.213/24
```

说明管理 IP 挂在 `br0` 上。此时不能说管理网直接走 `eth0` 或 `eth4`，只能说管理流量从逻辑接口 `br0` 出去。

下一步要查的是：

```text
br0 下面到底接了哪个物理口或哪个 bond？
```

## 看 br0 下挂了哪些端口

执行：

```bash
ovs-vsctl show
```

重点看 `Bridge br0` 下面的结构。常见情况有几种：

```text
Bridge br0
    Port br0
        Interface br0
            type: internal
    Port eth0
        Interface eth0
```

这种表示 `br0` 直接接了 `eth0`。

如果看到：

```text
Bridge br0
    Port br0-up
        Interface eth0
        Interface eth4
```

说明 `br0` 下面接的是一个 OVS bond，真实出口还要继续看 bond 当前 active slave。

更直观的命令是：

```bash
ovs-vsctl list-ports br0
```

如果输出是 `eth0`，说明 `br0` 只接了 `eth0`。如果输出是 `eth4`，说明只接了 `eth4`。如果输出是 `br0-up`，说明底层是 bond。

## 判断 eth0 / eth4 属于哪个网桥

如果现场有多块网卡，比如：

```text
eth0 = 千兆
eth4 = 万兆
```

可以直接查它们分别属于哪个 OVS bridge：

```bash
ovs-vsctl iface-to-br eth0
ovs-vsctl iface-to-br eth4
```

如果输出：

```text
br0
```

说明这个接口属于 `br0`。

如果报：

```text
no row "eth4" in table Interface
```

说明 `eth4` 没有加入 OVS 网桥。

## 如果是 bond，要看 active slave

当 `br0` 下层是 `br0-up` 这种 bond 时，执行：

```bash
ovs-appctl bond/show
```

重点看：

```text
bond_mode: active-backup
active slave mac: xx:xx:xx:xx:xx:xx(eth4)

slave eth0: enabled
slave eth4: enabled
```

如果 active slave 是 `eth4`，当前实际流量就走 `eth4`。如果 active slave 是 `eth0`，当前实际流量就走 `eth0`。

这一步很关键。因为有时两块网卡都是 `enabled`，但真正承载流量的只有 active slave。如果 active slave 指向的交换机端口 VLAN 不对，就会出现：

```text
br0 是 UP
IP 已配置
网卡链路也是 UP
但 ping 不通其他节点
Genesis / CVM 服务起不来
```

这时网卡硬件可能没坏，问题更可能在交换机端口、VLAN、Trunk/Access 或接线位置。

## 看 OVS 端口状态

继续查物理口在 OVS 里的状态：

```bash
ovs-vsctl list interface eth0
ovs-vsctl list interface eth4
```

重点看：

```text
admin_state
link_state
link_speed
ofport
```

正常状态一般是：

```text
admin_state : up
link_state  : up
ofport      : 正整数
```

如果 `ofport` 是 `-1`，说明这个 OVS 端口有问题。

也可以看 OpenFlow 端口：

```bash
ovs-ofctl show br0
```

## 用路由只能看到逻辑出口

执行：

```bash
ip route
```

可能看到：

```text
default via 192.168.14.1 dev br0
192.168.14.0/24 dev br0 proto kernel scope link src 192.168.14.213
```

这只能说明默认路由走 `br0`，并不能说明底层物理口是 `eth0` 还是 `eth4`。

也可以查网关路径：

```bash
ip route get 192.168.14.1
```

如果输出：

```text
192.168.14.1 dev br0 src 192.168.14.213
```

依然只是证明逻辑出口是 `br0`。

## ARP 比 ping 更适合判断二层问题

如果节点 IP 同网段，但互相不通，先看邻居表：

```bash
ip neigh show dev br0
```

如果看到目标节点是：

```text
FAILED
INCOMPLETE
```

基本说明二层 ARP 不通。

直接测 ARP：

```bash
arping -I br0 -c 5 192.168.14.211
arping -I br0 -c 5 192.168.14.212
arping -I br0 -c 5 192.168.14.223
```

如果 `arping` 都没有回应，优先怀疑：

```text
br0 底层物理口
OVS bond active slave
交换机 VLAN
交换机 Access / Trunk 配置
线缆接入位置
```

## 抓包确认实际从哪块网卡出去

假设要测试当前节点到 `192.168.14.211`，可以分别抓物理口：

```bash
tcpdump -eni eth0 host 192.168.14.211
```

另一个窗口：

```bash
tcpdump -eni eth4 host 192.168.14.211
```

然后执行：

```bash
ping 192.168.14.211
```

哪个口能看到 ARP/ICMP，说明当前流量实际从哪个口出去。

如果 `br0` 上能看到 ARP 请求，但物理口上看不到，可能是 OVS 转发或端口绑定问题。

如果物理口上能看到 ARP 请求，但没有回应，多半是交换机侧 VLAN、Access/Trunk 或链路接入问题。

## 临时切 active slave 验证链路

如果 `br0-up` 是 active-backup bond，而且 `eth0` 和 `eth4` 都是 enabled，可以临时切 active slave 做验证。

先确认当前状态：

```bash
ovs-appctl bond/show br0-up
```

临时切到 `eth0`：

```bash
ovs-appctl bond/set-active-slave br0-up eth0
```

确认是否切换成功：

```bash
ovs-appctl bond/show br0-up
```

然后测试：

```bash
ping -c 5 192.168.14.211
arping -I br0 -c 5 192.168.14.211
```

如果切到 `eth0` 后网络恢复，说明之前 active 的 `eth4` 所在链路或交换机配置有问题。

生产环境里这类操作要谨慎，最好在维护窗口或确认影响范围后再做。验证完如果需要回切，可以再指定回原来的 active slave。

## 节点服务起不来时怎么关联网络

如果执行 `cluster status` 或 `cs` 一直提示类似：

```text
Failed to reach a node where Genesis is up.
Ensure Genesis is running on all CVMs. Retrying...
```

不要只盯着 Genesis。它也可能是因为 CVM 之间网络不通，导致当前节点无法访问其他正常节点。

可以按顺序查：

```bash
genesis status
ps -ef | grep -i genesis | grep -v grep
svmips
allssh "genesis status | head -30"
cluster status
```

如果 `allssh` 不可用，分别 SSH 到每台 CVM 执行：

```bash
genesis status
```

再结合网络测试：

```bash
ping <其他Host_IP>
ping <其他CVM_IP>
arping -I br0 <其他节点IP>
```

如果 Host 能通但 CVM 不通，重点看 CVM 虚拟网卡、vnet 端口、OVS port 和 br0 的连接关系。

如果 Host 和 CVM 都不通，重点看物理链路、VLAN、bond active slave 和交换机端口。

## 推荐排障顺序

我会按这个顺序来：

```bash
ip a
ip route
ip neigh show dev br0
arping -I br0 <同网段其他节点IP>
ovs-vsctl show
ovs-vsctl list-ports br0
ovs-vsctl iface-to-br eth0
ovs-vsctl iface-to-br eth4
ovs-appctl bond/show
ovs-vsctl list interface eth0
ovs-vsctl list interface eth4
ovs-ofctl show br0
```

这组命令分别回答：

```text
IP 配在哪？
默认路由走哪？
ARP 是否通？
br0 下面接了什么？
物理口属于哪个网桥？
bond 当前 active slave 是谁？
OVS 端口状态是否正常？
```

## 我的理解

这次排障里最重要的转变是：不能把 `ip a` 看到的接口 UP 当成网络正常。Nutanix AHV 的管理网络通常挂在 `br0` 这种 OVS 网桥上，真正的物理出口可能藏在 `br0-up` bond 后面。

所以排障要从逻辑层一路往下走：

```text
管理 IP → br0 → OVS port / bond → active slave → ethX → 交换机端口 → VLAN
```

只要其中一层错了，就可能出现服务起不来、CVM 互通失败、Genesis 状态异常。

## 一句话总结

Nutanix AHV 节点服务起不来时，如果怀疑网络问题，不能只看 `br0` 是否 UP，而要继续确认 `br0` 底层 OVS bond 的 active slave、ARP 是否可达、物理口是否加入网桥、交换机 VLAN 是否一致；很多“服务异常”本质上是二层网络或 bond 选路问题。