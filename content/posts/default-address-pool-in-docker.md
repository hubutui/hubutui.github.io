---
title: "关于 Docker 默认地址池选项的权威指南"
date: 2025-06-25T00:00:00+08:00
draft: false
toc: false
images:
tags:
  - docker
---

## 前言

如果你创建的 docker compose 项目足够多，在默认的 docker 配置下，你很快就会遇到这个错误：

```bash
ERROR: could not find an available, non-overlapping IPv4 address pool among the defaults to assign to the network
```

具体地说，这是在我们已经创建了 31 个 docker 桥接网络的情况下，继续使用 `docker compose up` 启动新的服务的时候会遇到的。

TLDR，这个问题的根源在于 docker 默认的子网太大，以致于我们只能创建 31 个子网。我们可以通过修改子网的大小，来划分更多的子网。你可以修改 `daemon.json` 配置如下：

```json
{
  "default-address-pools": [
    {
      "base": "172.16.0.0/12",
      "size": 20
    },
    {
      "base": "192.168.0.0/16",
      "size": 24
    }
  ]
}
```

按照这个新的配置，第一个地址池可以创建 256 个子网，每个子网可以容纳 4096 个 IP；第二个地址池可以创建 256 个子网，每个子网容纳 256 个 IP。完全足够我们使用了。

如果你想要了解更多细节，可以继续往后读。

## 一些前提知识

### IPv4 地址与子网划分基础（仅讨论 IPv4）

**IP地址表示**
IPv4 地址是一个32位的二进制数，通常采用点分十进制表示法以提高可读性。具体格式是将32位地址划分为4个8位字节，每个字节转换为十进制数值（0-255），中间用点号分隔。例如：`192.168.1.1`。

**子网掩码的作用**
仅凭IP地址无法确定设备在网络中的具体位置，需要结合子网掩码（Subnet Mask）来区分地址中的网络部分和主机部分。子网掩码是一个与IP地址等长的32位二进制数，其特点是前N位为连续的1，其余位为0：

- **网络部分**：子网掩码中"1"对应的IP地址位，标识设备所属的网络
- **主机部分**：子网掩码中"0"对应的IP地址位，标识设备在该网络中的具体位置

**CIDR表示法**
为简化子网表示，业界普遍采用无类别域间路由（CIDR）表示法，格式为：`X.X.X.X/N`  
其中：

- `X.X.X.X` 为网络地址
- `N` 为网络前缀长度（即子网掩码中连续"1"的位数）

例如：`192.168.1.0/24` 表示前 24 位为网络地址，剩余 8 位用于主机寻址，可支持 256个 主机地址（实际去除网络地址和广播地址，可用的地址更少，但是本文中暂时忽略不考虑）。

对于 docker 的地址池设置参数：

```json
{
  "default-address-pools": [
    {
      "base": "172.16.0.0/12",
      "size": 20
    }
  ]
}
```

- `base` 表示基地址，也就是整个可以划分的地址池。
- `size` 表示每个子网的掩码长度，决定了每个从 base 中划分的子网的大小。

docker 在创建桥接网络的时候，会从基地址中划分出来 `size` 大小的子网。根据以上参数，我们可以计算：

- 可划分的子网数量为：$$ 2^(szie - base) $$，这里 `base` 指的是基地址的掩码长度。
- 每个子网容纳的 IP 数量为：$$ 2^(32 - size) $$

### 使用 python 计算

我们也可以用 python 来计算，例如：

```python
from ipaddress import IPv4Network

net = IPv4Network("172.16.0.0/12")
subnets = list(net.subnets(new_prefix=20))
# 可划分子网数量
num_subnets = len(subnets)
# 子网容纳的地址数量
num_address_per_subnet = subnets[0].num_addresses
```

## docker 的原生地址池配置

在我们不写任何配置的时候，docker 自己原生的地址池配置是如何的呢？[The definitive guide to docker's default-address-pools option](https://straz.to/2021-09-08-docker-address-pools) 给出的是：

```json
{
  "default-address-pools": [
    {
      "base": "172.17.0.0/12",
      "size": 16
    },
    {
      "base": "192.168.0.0/16",
      "size": 20
    }
  ]
}
```

实际上这并不准确。你用 python 代码检查一下：

```python
net = IPv4Network("172.17.0.0/12")
```

它会报错：

```text
ValueError: 172.17.0.0/12 has host bits set
```

这是因为 `172.17.0.0` 的并不是这个网段的网络地址，正确的网络地址应该为 `172.16.0.0`，我们只需要：

```python
net = IPv4Network("172.17.0.0/12", strict=False)
print(net.network_address)
```

即可知道其网络地址，正确的网段表示应该是 `172.16.0.0/12`。那是否说明 docker 的默认地址池配置如下呢？

```json
{
  "default-address-pools": [
    {
      "base": "172.16.0.0/12",
      "size": 16
    },
    {
      "base": "192.168.0.0/16",
      "size": 20
    }
  ]
}
```

也不是，如果按照上述配置，我们应该可以创建 32 个 docker 桥接网络，而不是 31 个。

一切都得回到源码，根据 [issue #8863](https://github.com/docker/docs/issues/8663)，可以知道对应的源码在 [ipamutils/utils.go#L10-L22](https://github.com/moby/libnetwork/blob/master/ipamutils/utils.go#L10-L22)。不过这仓库已经不再维护，代码合并到了 [moby](https://github.com/moby/moby) 仓库，对应的代码是 [libnetwork/ipamutils/utils.go#L12-L25](https://github.com/moby/moby/blob/v28.2.2/libnetwork/ipamutils/utils.go#L12-L25)。从源码中我们可以知道，Docker 的原生地址池配置如下：

```json
{
  "default-address-pools": [
    {
      "base": "172.17.0.0/16",
      "size": 16
    },
    {
      "base": "172.18.0.0/16",
      "size": 16
    },
    {
      "base": "172.18.0.0/16",
      "size": 16
    },
    {
      "base": "172.19.0.0/16",
      "size": 16
    },
    {
      "base": "172.20.0.0/14",
      "size": 16
    },
    {
      "base": "172.24.0.0/14",
      "size": 16
    },
    {
      "base": "172.28.0.0/14",
      "size": 16
    },
    {
      "base": "192.168.0.0/16",
      "size": 20
    }
  ]
}
```

经过计算即可知道，按照这个配置，我们可以创建 31 个 docker 桥接网络，子网分别可以容纳 65536 和 4096 个 IP。这正好符合我们的实际情况。此外，注意到还有一个 `10.0.0.0/8` 的网段用于 docker swarm，我们这里暂不讨论。

由此，我们给出了文章一开始的解决方案，把子网的大小减小，从而可以创建更多的子网。

## 避免 IP 冲突

假设你的宿主机网络设备为 eth0，使用命令 `ip addr show eth0` 输出：

```text
inet 172.20.2.32/20 metric 100 brd 172.20.15.255 scope global dynamic eth
```

直接使用：

```python
net = IPv4Network('172.20.2.32/20', strict=False)
print(net)
```

可以知道所在的子网为 `172.20.0.0/20`。

如果 docker 使用的网络跟这个子网有重叠，可能会有网络冲突的可能。为此，我们考虑避开这个网段，设置新的默认地址池。或者，考虑到宿主机所在的网络可能会使用 `172.16.0.0/12` 整个私有网段，我们也可以完全不用此网段。我们可以直接仅用 `192.168.0.0/16`，配置如下：

```json
{
  "default-address-pools": [
    {
      "base": "192.168.0.0/16",
      "size": 24
    }
  ]
}
```

这样我们可以创建 256 个容纳 256 个 IP 的子网，一般也足够使用。如果你确认自己不用 docker swarm，可以把 `10.0.0.0/8` 给用上。例如配置：

```json
{
  "default-address-pools": [
    {
      "base": "10.0.0.0/8",
      "size": 20
    }
  ]
}
```

这样就可以创建 4096 个可以容纳 4096 个 IP 的子网了，应该也是足够使用的。

## 参考

1. [issue #8863](https://github.com/docker/docs/issues/8663)
2. [The definitive guide to docker's default-address-pools option](https://straz.to/2021-09-08-docker-address-pools)
