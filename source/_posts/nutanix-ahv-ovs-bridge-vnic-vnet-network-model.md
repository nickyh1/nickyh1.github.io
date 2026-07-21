---
title: Nutanix AHV 网络模型：OVS Bridge、vNIC、vnet 和物理上联的关系
date: 2026-07-21
tags:
  - Nutanix
  - AHV
  - OVS
  - Linux Bridge
  - 虚拟网络
categories:
  - 知识笔记
description: 梳理 Nutanix AHV 和常见虚拟化/容器网络里，虚拟机 vNIC、宿主机 vnet/tap、OVS Bridge、Linux Bridge、bond、物理网卡之间的连接关系，以及如何用命令显式查看这些关系。
---

理解 Nutanix AHV 网络时，最容易混淆的是这些词：OVS Bridge、Linux Bridge、虚拟交换机、br0、vnet、vNIC、bond、uplink、ethX。它们有些是同一层概念，有些是不同层的连接点。如果只看名字，很容易把 `br0` 当成普通网卡，或者把 vNIC 和 vnet 混成一回事。

<!-- more -->

## 核心结论

可以先用一条链路把 AHV 的网络模型记住：

```text
VM vNIC
  ↓
Host 上的 vnet/tap 端口
  ↓
OVS Bridge br0
  ↓
OVS bond / uplink，例如 br0-up
  ↓
物理网卡 eth0 / eth4
  ↓
物理交换机
```

其中：

```text
vNIC = 虚拟机里看到的网卡
vnet/tap = 宿主机上代表这块虚拟机网卡的端口
OVS Bridge = 宿主机上的软件交换机
bond/uplink = 虚拟交换机通向物理网络的上联
ethX = 真实物理网卡
```

虚拟机不会直接接到物理网卡，而是先通过宿主机侧的 vnet/tap 端口接入虚拟交换机，再由虚拟交换机通过上联口转发到物理网络。

## OVS Bridge 和 OVS bond 的关系

可以这样理解：

```text
OVS Bridge = 虚拟交换机
OVS bond = 挂在虚拟交换机上的一个聚合端口 / 上联口
```

在 Nutanix AHV 里，常见结构类似：

```text
Bridge br0
├── Port br0
│   └── Interface br0
│       type: internal
├── Port vnet0
│   └── Interface vnet0
├── Port vnet2
│   └── Interface vnet2
└── Port br0-up
    ├── Interface eth0
    └── Interface eth4
```

这里 `br0` 是 OVS Bridge，也就是软件交换机。`vnet0`、`vnet2` 是虚拟机接进来的端口。`br0-up` 是上联 bond，下面绑定真实物理网卡 `eth0`、`eth4`。

如果 bond 模式是 active-backup，那么同一时间只有一块 active slave 真正承载流量。可以用：

```bash
ovs-appctl bond/show
```

查看当前 active slave。

## br0 到底是什么

`br0` 容易让人迷惑，因为它同时出现在多个层面。

在 OVS 里：

```text
Bridge br0 = OVS 虚拟交换机对象
Port br0 / Interface br0 type: internal = 宿主机可配置 IP 的内部接口
```

也就是说，`br0` 既是一个虚拟交换机的名字，也可能有一个同名 internal interface。管理 IP 通常配在这个 internal interface 上，例如：

```text
br0: inet 192.168.14.213/24
```

但网桥本身做二层转发并不一定需要 IP。IP 是给宿主机自己通信用的，不是给交换机转发以太网帧用的。

更准确地说：

```text
桥 / 虚拟交换机负责二层转发。
桥上的 internal interface 配 IP，是为了让宿主机自己接入这个二层网络。
```

## 网桥一定要有 IP 吗

不一定。

如果一个网桥只负责二层转发，它可以没有 IP。比如只负责把虚拟机流量转发到物理网卡，网桥本身不作为管理入口，就不需要 IP。

如果宿主机自己也需要通过这个网桥通信，比如 AHV Host 的管理 IP 放在 `br0` 上，那就需要给 `br0` 的 internal interface 配 IP、掩码、网关、DNS。

所以可以这样记：

```text
网桥转发二层流量，不依赖 IP。
宿主机要通过网桥通信，才需要在网桥 internal interface 上配 IP。
```

## vNIC 和 vnet 的区别

这两个词也很容易混。

```text
vNIC = 虚拟机内部看到的网卡
vnet/tap = 宿主机侧对应这块虚拟网卡的接口
```

虚拟机里看到：

```text
eth0 / ens3
```

宿主机上可能看到：

```text
vnet0 / vnet2 / tapX
```

它们是一对关系：虚拟机以为自己有一块网卡，宿主机则用一个 tap/vnet 端口代表这块网卡，并把它接到 Linux Bridge 或 OVS Bridge 上。

在 AHV / OVS 场景里可以理解为：

```text
VM eth0
  ↓
VM vNIC
  ↓
Host vnetX
  ↓
OVS Bridge br0
```

所以，vnet 可以理解成虚拟机接入虚拟交换机的“宿主机侧网线头”。

## 虚拟交换机只关心端口和转发

虚拟交换机/网桥主要关心：

```text
端口
MAC 地址
VLAN
转发表
上联口
流表 / 策略
```

它不关心虚拟机里面跑的是 Windows 还是 Linux，也不关心容器里跑的是 Nginx 还是 Redis。

对 OVS Bridge 来说，接进来的只是端口：

```text
vnet0
vnet1
vethxxx
tapxxx
eth0
bond0
br0-up
```

它要做的是：这个 MAC 在哪个端口、这个 VLAN 该怎么转发、这个流量该从哪个上联出去。

普通 Linux Bridge / OVS Bridge 默认主要做二层转发，不是传统意义上的三层路由器。OVS 配合 OpenFlow、OVN 或 SDN 控制器时，可以扩展出 ACL、隧道、NAT、三层路由等能力，但基础概念仍然是“软件交换机”。

## OVS Bridge 和 Linux Bridge 怎么区分

不能只看名字，因为二者都可能叫 `br0`。

看 OVS Bridge：

```bash
ovs-vsctl show
ovs-vsctl list-br
ovs-vsctl br-exists br0
ovs-vsctl list-ports br0
ovs-vsctl iface-to-br eth0
```

如果能被 `ovs-vsctl` 正常管理，基本就是 OVS Bridge。

看 Linux Bridge：

```bash
ip link show type bridge
bridge link
brctl show
ls /sys/class/net/br0/bridge
```

如果 `brctl show` 或 `bridge link` 能看到从属接口，或者 `/sys/class/net/br0/bridge` 存在，说明它是 Linux Bridge。

简单对比：

| 项目 | OVS Bridge | Linux Bridge |
|---|---|---|
| 常见命令 | `ovs-vsctl`、`ovs-ofctl`、`ovs-appctl` | `ip`、`bridge`、`brctl` |
| bond | OVS bond，例如 `br0-up` | Linux bond，例如 `bond0` |
| 常见场景 | Nutanix AHV、OpenStack、OVN | KVM 简单桥接、libvirt NAT、Docker 默认网络 |
| 能力 | 支持 OpenFlow、流表、SDN 集成 | Linux 内核基础二层网桥 |

## Linux Bridge 的流量路径

Linux Bridge 的逻辑和 OVS Bridge 很像，只是实现和管理方式不同。

普通 KVM + Linux Bridge 常见路径：

```text
VM vNIC
  ↓
Host tap/vnet0
  ↓
Linux Bridge br0
  ↓
Host eth0
  ↓
Physical Switch
```

如果使用 Linux bond：

```text
VM vNIC
  ↓
vnet0 / tap0
  ↓
Linux Bridge br0
  ↓
bond0
  ↓
eth0 + eth1
  ↓
Physical Switch
```

Linux Bridge 里通常没有 `br0-up` 这种 OVS 风格对象。它的“上联”一般就是加入 bridge 的物理网卡、bond 口或 VLAN 子接口。

## 容器和虚拟机接入方式有什么不同

虚拟交换机/网桥可以看作网络转发平面，虚拟机和容器是工作负载隔离平面。关键连接点是：工作负载的网卡如何接到虚拟交换机。

可以拆成三层：

```text
工作负载层：虚拟机 / 容器
接入层：tap / vnet / veth / macvlan / SR-IOV VF
转发层：Linux Bridge / OVS Bridge / Virtual Switch
```

虚拟机场景常见路径：

```text
虚拟机 eth0
  ↓
虚拟机 vNIC
  ↓
QEMU/KVM 后端设备
  ↓
Host tap/vnetX
  ↓
OVS Bridge / Linux Bridge
```

容器场景常见路径：

```text
容器 eth0
  ↓
veth pair 容器端
  ↓
veth pair 宿主机端 vethXXX
  ↓
Linux Bridge / OVS Bridge
```

Docker 默认 bridge 网络通常是：

```text
容器 eth0 → veth pair → docker0 Linux Bridge → iptables NAT → Host 物理网卡
```

Kubernetes / OVN / Kube-OVN 类网络可能是：

```text
Pod eth0 → veth pair → OVS br-int → Geneve / VXLAN / VLAN → 其他节点或外部网络
```

所以虚拟机和容器的差别之一就是：

```text
虚拟机常用 tap/vnet 接入网桥。
容器常用 veth pair 接入网桥。
```

## 如何显式查看“是谁决定接到哪个网桥”

这次对话里最重要的问题不是流量路径，而是定义关系在哪里看。可以按四个问题查：

```text
谁决定接到哪个网桥？
谁创建了宿主机侧端口？
这个端口现在挂在哪个虚拟交换机上？
虚拟交换机里有没有这个端口？
```

### 虚拟机场景

查看虚拟机网卡定义：

```bash
virsh list --all
virsh domiflist <虚拟机名>
```

典型输出：

```text
Interface  Type     Source   Model    MAC
vnet2      bridge   br0      virtio   50:6b:8d:xx:xx:xx
```

这说明：

```text
vnet2 = 宿主机侧接口
Source br0 = 接入的网桥
Model virtio = 虚拟网卡类型
MAC = 虚拟机 vNIC 的 MAC
```

更底层可以看 XML：

```bash
virsh dumpxml <虚拟机名> | grep -A20 "<interface"
```

可能看到：

```xml
<interface type='bridge'>
  <mac address='50:6b:8d:xx:xx:xx'/>
  <source bridge='br0'/>
  <target dev='vnet2'/>
  <model type='virtio'/>
</interface>
```

这段定义说明：

```text
虚拟机 vNIC → Host vnet2 → br0
```

### OVS 交换机侧

确认 vnet 是否挂到了 OVS：

```bash
ovs-vsctl show
ovs-vsctl iface-to-br vnet2
ovs-vsctl list interface vnet2
ovs-vsctl list port vnet2
```

如果：

```bash
ovs-vsctl iface-to-br vnet2
```

返回：

```text
br0
```

说明 `vnet2` 属于 OVS Bridge `br0`。

看 VLAN：

```bash
ovs-vsctl get port vnet2 tag
ovs-vsctl get port vnet2 trunks
```

看 OVS 端口状态：

```bash
ovs-ofctl show br0
```

### Linux Bridge 交换机侧

确认 vnet 是否挂到了 Linux Bridge：

```bash
brctl show
bridge link
ip link show master br0
bridge fdb show br br0
ip -d link show br0
```

这些命令能看到 `vnet0`、`vnet1`、`eth0`、`bond0` 等接口是否挂在 `br0` 上。

### Docker 容器场景

看 Docker 网络定义：

```bash
docker network ls
docker network inspect bridge
```

看容器网络：

```bash
docker inspect <容器名或ID>
docker inspect <容器名> --format '{{json .NetworkSettings.Networks}}'
```

进入容器网络命名空间：

```bash
pid=$(docker inspect -f '{{.State.Pid}}' <容器名>)
nsenter -t $pid -n ip addr
```

看宿主机侧 veth 和 bridge：

```bash
ip -d link show
bridge link
brctl show docker0
```

### Kubernetes / CNI 场景

CNI 配置通常在：

```bash
ls /etc/cni/net.d/
cat /etc/cni/net.d/*.conf
cat /etc/cni/net.d/*.conflist
```

如果看到：

```json
{
  "type": "bridge",
  "bridge": "cni0"
}
```

说明 Pod 通过 bridge CNI 接入 `cni0`。

如果是 OVN / Kube-OVN / OVS 类网络，可能看到 `ovn-k8s-cni-overlay`、`kube-ovn`、`ovs` 等类型，Pod 可能通过 veth 接入 OVS Bridge，比如 `br-int`。

常用查看命令：

```bash
kubectl get pod -o wide
kubectl describe pod <pod名> -n <namespace>
ip link
bridge link
ovs-vsctl show
ovs-vsctl list-ports br-int
```

## 我的理解

这套网络模型最关键的是分层：虚拟交换机/网桥是一套网络转发组件，虚拟机和容器是一套计算隔离组件。二者不是同一个东西，它们之间靠虚拟接口连接。

更严谨地说：

```text
虚拟交换机/网桥负责端口之间的二层转发；
虚拟机或容器如何接入这个网桥，是由上层组件定义和创建的。
虚拟机场景通常由 libvirt/QEMU/Nutanix 管理层创建 vnet/tap 并挂到 bridge；
容器场景通常由 Docker 或 CNI 插件创建 veth pair，并把宿主机侧 veth 挂到 Linux Bridge 或 OVS Bridge。
```

排查时不要只问“流量从哪里走”，还要问“这个连接关系是谁定义的、在哪里生效的”。

## 一句话总结

Nutanix AHV 里，虚拟机 vNIC 通过宿主机侧 vnet/tap 接入 OVS Bridge，OVS Bridge 再通过 bond/uplink 接到物理网卡；Linux Bridge 和 OVS Bridge 都能当虚拟交换机用，区别在实现和管理命令；虚拟机通常用 tap/vnet 接入，容器通常用 veth pair 接入，具体接到哪个网桥由 libvirt、Docker 或 CNI 等上层组件定义。