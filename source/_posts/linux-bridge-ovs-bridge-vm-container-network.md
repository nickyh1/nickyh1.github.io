---
title: Linux Bridge、OVS Bridge 与虚拟机/容器网络的关系
date: 2026-07-21
tags:
  - Linux Bridge
  - OVS
  - 容器网络
  - 虚拟机网络
  - CNI
categories:
  - 知识笔记
description: 梳理 Linux Bridge、OVS Bridge、虚拟交换机、虚拟机 vNIC/tap/vnet、容器 veth、容器运行时和 CNI 插件之间的关系。
---

理解虚拟化和容器网络时，最容易混在一起的是这些概念：Linux Bridge、OVS Bridge、虚拟交换机、虚拟机网卡、容器网卡、vnet、tap、veth、容器运行时、CNI 插件。它们不是同一层东西，而是分布在“工作负载、接入接口、转发平面、管理控制层”几个不同层次上。

<!-- more -->

## 核心结论

可以先用一句话抓住本质：

```text
虚拟交换机/网桥负责转发，虚拟机和容器负责运行工作负载，二者通过虚拟接口连接。
```

更完整一点：

```text
工作负载层：虚拟机 / 容器
接入接口层：tap / vnet / veth / macvlan / SR-IOV VF
转发平面层：Linux Bridge / OVS Bridge / Virtual Switch
控制管理层：libvirt / QEMU / Docker / containerd / CNI / SDN 控制器
```

所以不要把“虚拟机网络”和“虚拟交换机”混成一个东西。虚拟机或容器只是接入网络的工作负载，真正做二层转发的是 Linux Bridge、OVS Bridge 这类网桥/虚拟交换机。

## Linux Bridge 是什么

Linux Bridge 是 Linux 内核自带的二层网桥，可以把多个接口桥接在一起，行为类似一台简单的软件交换机。

典型 KVM + Linux Bridge 路径是：

```text
虚拟机内部 eth0
  ↓
虚拟机 vNIC
  ↓
宿主机 tap/vnet0
  ↓
Linux Bridge br0
  ↓
宿主机 eth0 / bond0
  ↓
物理交换机
```

如果有多台虚拟机：

```text
br0
├── vnet0  # VM1
├── vnet1  # VM2
└── eth0   # 物理上联
```

Linux Bridge 常见命令：

```bash
brctl show
bridge link
ip link show type bridge
ip link show master br0
bridge fdb show br br0
ip -d link show br0
```

它的特点是简单、内核原生、适合普通 KVM 桥接、libvirt 网络、Docker 默认 bridge 这类场景。

## OVS Bridge 是什么

OVS Bridge 来自 Open vSwitch，也是一种软件交换机，但能力比 Linux Bridge 更强，常用于 Nutanix AHV、OpenStack、OVN、Kube-OVN、SDN 网络等场景。

典型 OVS 路径是：

```text
虚拟机内部 eth0
  ↓
虚拟机 vNIC
  ↓
宿主机 vnet/tap
  ↓
OVS Bridge br0
  ↓
OVS bond / uplink，例如 br0-up
  ↓
eth0 / eth4
  ↓
物理交换机
```

在 Nutanix AHV 里常见结构类似：

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

OVS 常见命令：

```bash
ovs-vsctl show
ovs-vsctl list-br
ovs-vsctl list-ports br0
ovs-vsctl iface-to-br vnet0
ovs-vsctl list interface vnet0
ovs-vsctl list port vnet0
ovs-ofctl show br0
ovs-appctl bond/show
```

OVS 的优势在于支持更复杂的端口管理、bond、VLAN、OpenFlow、流表、隧道和 SDN 控制器集成。

## Linux Bridge 和 OVS Bridge 的区别

二者都能当“虚拟交换机”用，但实现和能力不同。

| 对比项 | Linux Bridge | OVS Bridge |
|---|---|---|
| 实现位置 | Linux 内核 bridge | Open vSwitch |
| 典型命令 | `bridge`、`brctl`、`ip` | `ovs-vsctl`、`ovs-ofctl`、`ovs-appctl` |
| 常见场景 | KVM 简单桥接、Docker 默认网络、libvirt NAT | Nutanix AHV、OpenStack、OVN、Kube-OVN、SDN |
| 端口模型 | 接口加入 bridge | Bridge / Port / Interface 三层对象 |
| bond | Linux bond，如 `bond0` | OVS bond，如 `br0-up` |
| 高级能力 | 基础二层转发 | VLAN、OpenFlow、隧道、SDN 控制更强 |

最简单的判断方式：

```text
ovs-vsctl 能看到并管理的，是 OVS Bridge。
brctl / bridge 能看到并管理的，是 Linux Bridge。
```

但要注意，`ip a` 只能看到接口，不能单独证明一个 `br0` 到底是 Linux Bridge 还是 OVS Bridge。

## vNIC、tap、vnet 是什么关系

虚拟机里的网卡叫 vNIC。它是虚拟机操作系统看到的网卡，比如：

```text
eth0
ens3
```

但在宿主机上，这块虚拟网卡通常会对应一个 tap/vnet 接口，比如：

```text
vnet0
tap0
```

可以这样理解：

```text
vNIC = 虚拟机内部看到的网卡
vnet/tap = 宿主机侧代表这块虚拟网卡的接口
```

完整链路是：

```text
VM 内部 eth0
  ↓
VM vNIC
  ↓
QEMU/KVM 后端设备
  ↓
Host tap/vnet0
  ↓
Linux Bridge / OVS Bridge
```

也就是说，虚拟机不会直接把自己的网卡插到物理交换机上，而是先通过宿主机侧的 tap/vnet 接到虚拟交换机。

## 容器为什么常用 veth pair

容器不是完整虚拟机，它主要依靠 Linux namespace 隔离网络栈。容器接入宿主机网络时，最常见的是 veth pair。

veth pair 可以理解为一根虚拟网线，两头分别放在不同网络命名空间里：

```text
容器内部 eth0
  ↓
veth pair 容器端
  ↓
veth pair 宿主机端 vethXXX
  ↓
Linux Bridge / OVS Bridge
```

Docker 默认 bridge 网络通常是：

```text
容器 eth0
  ↓
veth pair
  ↓
docker0 Linux Bridge
  ↓
iptables NAT
  ↓
宿主机物理网卡
```

Kubernetes 里则由 CNI 插件决定接入方式，可能是：

```text
Pod eth0
  ↓
veth pair
  ↓
cni0 / OVS br-int / 其他 CNI 网络设备
  ↓
VXLAN / Geneve / VLAN / 物理网络
```

## 虚拟机网络和容器网络的核心差异

虚拟机和容器都可以接入虚拟交换机，但接入方式不同。

| 场景 | 工作负载里看到的网卡 | 宿主机侧接口 | 接入设备 |
|---|---|---|---|
| KVM 虚拟机 | vNIC / eth0 | tapX / vnetX | Linux Bridge / OVS Bridge |
| Nutanix AHV 虚拟机 | vNIC | vnetX | OVS Bridge br0 |
| Docker 容器 | eth0 | vethXXX | docker0 Linux Bridge |
| Kubernetes Pod | eth0 | vethXXX | cni0 / OVS br-int / 其他 CNI |
| SR-IOV 虚拟机/容器 | VF 网卡 | 物理网卡 VF | 可能绕过软件网桥 |

所以区别不在于“有没有虚拟交换机”，而在于：虚拟机或容器如何把自己的网卡接进这个虚拟交换机。

一句话：

```text
虚拟机常用 tap/vnet 接入网桥，容器常用 veth pair 接入网桥。
```

## 虚拟化层、容器运行时和 CNI 分别负责什么

虚拟交换机本身主要负责转发，它并不主动决定某台虚拟机或某个容器应该接到哪里。这个“定义和创建关系”通常由上层组件负责。

虚拟机场景里，通常是：

```text
libvirt / QEMU / Nutanix 管理层
  ↓
定义虚拟机 vNIC
  ↓
创建宿主机侧 tap/vnet
  ↓
把 tap/vnet 挂到 Linux Bridge 或 OVS Bridge
```

容器场景里，通常是：

```text
Docker / containerd / CRI
  ↓
创建容器网络命名空间
  ↓
调用 CNI 插件或内置网络驱动
  ↓
创建 veth pair
  ↓
把宿主机侧 veth 挂到 bridge / OVS / overlay 网络
```

Kubernetes 里尤其要记住：Pod 网络不是 kubelet 自己完整实现的，而是由 CNI 插件落地，比如 bridge CNI、Calico、Flannel、Cilium、OVN-Kubernetes、Kube-OVN 等。

## 如何显式查看这些关系

排查时要回答四个问题：

```text
谁定义了这个网络？
谁创建了宿主机侧接口？
接口现在挂到了哪个网桥？
网桥里是否真的有这个端口？
```

### 虚拟机场景

看虚拟机网卡定义：

```bash
virsh domiflist <虚拟机名>
```

典型输出：

```text
Interface  Type     Source   Model    MAC
vnet2      bridge   br0      virtio   50:6b:8d:xx:xx:xx
```

这说明：

```text
虚拟机 vNIC → Host vnet2 → br0
```

看更底层 XML：

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

### OVS 侧

确认端口是否挂到 OVS：

```bash
ovs-vsctl show
ovs-vsctl iface-to-br vnet2
ovs-vsctl list interface vnet2
ovs-vsctl list port vnet2
```

查看 VLAN：

```bash
ovs-vsctl get port vnet2 tag
ovs-vsctl get port vnet2 trunks
```

### Linux Bridge 侧

确认接口是否挂到 Linux Bridge：

```bash
brctl show
bridge link
ip link show master br0
bridge fdb show br br0
```

### Docker 容器场景

看网络定义：

```bash
docker network ls
docker network inspect bridge
```

看容器网络配置：

```bash
docker inspect <容器名或ID>
docker inspect <容器名> --format '{{json .NetworkSettings.Networks}}'
```

进入容器网络命名空间：

```bash
pid=$(docker inspect -f '{{.State.Pid}}' <容器名>)
nsenter -t $pid -n ip addr
```

看宿主机侧 veth 是否挂到 bridge：

```bash
ip -d link show
bridge link
brctl show docker0
```

### Kubernetes / CNI 场景

看 CNI 配置：

```bash
ls /etc/cni/net.d/
cat /etc/cni/net.d/*.conf
cat /etc/cni/net.d/*.conflist
```

看 Pod 落在哪个节点：

```bash
kubectl get pod -o wide
kubectl describe pod <pod名> -n <namespace>
```

看节点上的接口：

```bash
ip link
bridge link
ovs-vsctl show
ovs-vsctl list-ports br-int
```

这些命令组合起来，才能把“定义关系”和“实际生效状态”对上。

## 我的理解

以前容易把 Linux Bridge、OVS Bridge、虚拟机网络、容器网络都叫成“虚拟网络”，但这样太粗了。更准确的理解是：

```text
Linux Bridge / OVS Bridge 是转发平面。
虚拟机 / 容器是工作负载。
tap / vnet / veth 是接入接口。
libvirt / Docker / CNI 是定义和创建这些连接关系的控制层。
```

排查网络时，只看 `ip a` 或只看一个 bridge 名字是不够的。要从工作负载一路查到宿主机接口，再查到 bridge，再查到上联口和物理网络。

## 一句话总结

Linux Bridge 和 OVS Bridge 都可以作为虚拟交换机；虚拟机通常通过 tap/vnet 接入，容器通常通过 veth pair 接入；具体接到哪个网桥、端口叫什么、VLAN 怎么打，是由虚拟化层、容器运行时或 CNI 插件定义并创建的。