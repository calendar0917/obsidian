---
title: "御林服务器的初步认识"
subtitle: "御林服务器的初步认识"
summary: "记录相关知识的学习、服务器初始概况"
description: "记录相关知识的学习、服务器初始概况"
image: ""
date: 2025-11-09
lastmod: 2025-11-09
draft: false
toc:
 enable: true
hiddenFromHomePage: false
weight: false
categories: ["DevOps"]
tags: ["技术杂项"]
---

## 知识补充

### PVE SDN 的核心概念

[LXC使用tailscale注意事项](https://tailscale.com/kb/1130/lxc-unprivileged#providing-access-for-proxmox-6-and-earlier)

> - 非特权 LXC 容器的 root 用户（uid 0）会映射到宿主机的普通用户，安全性更高，但默认没有 Tailscale 所需的网络资源权限。
> - Tailscale 通过 UDP 数据包封装隧道，无需内核模块，但必须访问 `/dev/tun` 设备 —— 这是非特权容器默认不提供的。所以需要给 LXC 容器开放 /dev/tun 访问
>
> **具体操作**
>
> 根据 Proxmox 版本不同，配置方式略有差异，但核心命令一致：
>
> 适用 Proxmox 7 及以上（cgroup2 环境）
>
> - 找到目标 LXC 容器的配置文件：路径为 `/etc/pve/lxc/[容器ID].conf`（比如容器 ID 是 112，文件就是 112.conf）。
>
> - 在配置文件中添加以下 2 行：
>
> ```plaintext
> lxc.cgroup2.devices.allow: c 10:200 rwm
> lxc.mount.entry: /dev/net/tun dev/net/tun none bind,create=file
> ```
>
> - 重启容器：已运行的容器需先关机再启动，配置才会生效。
>
> - 后续操作：容器内安装 Tailscale Linux 包，即可正常使用。
>
> 适用 Proxmox 6 及更早（cgroup 环境）
>
> 配置命令与 Proxmox 7 类似，仅需将 `cgroup2` 改为 `cgroup`：
>
> ```plaintext
> lxc.cgroup.devices.allow: c 10:200 rwm
> lxc.mount.entry: /dev/net/tun dev/net/tun none bind,create=file
> ```
>
> 其余步骤（重启容器、安装 Tailscale）一致。

Proxmox VE 的 SDN 功能通过 “Zone（区域）- VNet（虚拟网络）- Subnet（子网）” 三层模型实现网络虚拟化，核心目标是为虚拟机 / 容器提供隔离的网络环境，同时支持灵活的地址管理、DHCP 服务和地址转换（SNAT）。

- **Zone**：最高级别的隔离单元，定义网络的基础类型（此处均为`simple`类型，即简单隔离模式，适用于单机或小规模集群）。
- **VNet**：绑定到 Zone 的虚拟网络，作为子网的载体，实现不同网络的逻辑隔离。
- **Subnet**：VNet 下的 IP 地址段，包含网关、DHCP 地址池、SNAT 开关等配置，直接为虚拟机分配网络资源。

#### 子网

子网就像一个 “小区”，里面有很多 “房子”（IP 地址）。每个子网都有一个固定的 “地址范围”，比如配置中的 `10.22.0.0/24` 或 `10.33.0.0/16`，这串数字定义了这个 “小区” 里有多少个可用的 IP 地址。

- 例子：`10.22.0.0/24` 表示这个子网的 IP 地址从 `10.22.0.1` 到 `10.22.0.254`（共 254 个地址），适合设备数量较少的场景。
- 例子：`10.33.0.0/16` 表示从 `10.33.0.1` 到 `10.33.255.254`（共 65534 个地址），适合设备数量多的场景。
- 常用的 IP 地址是 **32 位二进制数**,0/16 表示前十六位，即`10.33`不变

**为什么需要子网？**

如果所有设备都用同一个大网段，就像一个超级大的小区，找设备、管理设备会很麻烦，还容易冲突。子网通过 “分区” 让网络更有序，比如 R730 的 `vnet1` 和 `vnet2` 分别用两个子网，避免了 IP 地址混乱。

#### 网关

网关是子网的 “大门”，是一个特殊的 IP 地址（比如 `10.22.0.1`），负责子网内的设备和外部网络（其他子网或互联网）之间的通信。

- 比如：`vnet1` 子网的设备（IP 可能是 `10.22.0.100`）要访问 `vnet2` 子网的设备（IP 可能是 `10.33.1.100`），必须通过各自的网关（`10.22.0.1` 和 `10.33.0.1`）转发数据。
- 如果设备要访问互联网，也必须通过网关 “出门”。

**网关的作用：**

没有网关，子网就是一个 “孤岛”，里面的设备只能互相通信，无法连接外部。配置中的每个子网都指定了网关（如 `G: 10.22.0.1`），就是为了让子网能和外界互通。

#### DCHP

DHCP（动态主机配置协议）就像一个 “门牌号管理员”，当子网里新增设备（比如虚拟机）时，它会自动分配一个 IP 地址（从预设的地址池里选），不用人工手动设置。

- 例子：R730 的 `vnet1` 子网中，DHCP 地址池是 `10.22.0.100 - 10.22.0.200`，表示新设备接入时，会自动拿到这个范围内的 IP（比如 `10.22.0.101`、`10.22.0.150` 等）。
- 好处：避免人工分配 IP 时的冲突（比如两个人同时用 `10.22.0.10`），减少管理成本。

#### SNAT

SNAT（源网络地址转换）可以理解为子网的 “共用身份证”。子网内的设备没有直接连接互联网的 “公网 IP”，当它们要访问互联网时，网关会把设备的 “私网 IP”（如 `10.22.0.100`）转换成网关自己的 “公网 IP”，相当于用网关的身份去访问外部，回来的数据再转回去。

- 例子：`vnet1` 启用 SNAT 后，子网内的虚拟机（IP `10.22.0.100`）访问百度时，百度看到的不是 `10.22.0.100`，而是服务器 R730 的公网 IP（比如 `192.168.31.45` 对应的公网地址）。
- 好处：不需要给每个设备分配公网 IP（公网 IP 有限且贵），通过网关 “共用” 一个公网 IP 即可访问外部。

**为什么配置中都启用 SNAT？**

因为这些子网用的是 “私网 IP”（`10.x.x.x` 是内网专用地址，不能直接访问互联网），必须通过 SNAT 转换才能和互联网通信，所以 R730 和 R510 的所有子网都开启了 SNAT。

#### 

#### VNet

VNet（虚拟网络）是一个 “逻辑上的独立网络”，就像在物理服务器上划分出的 “虚拟网线”，不同 VNet 之间默认是隔离的（不能直接通信），即使它们在同一个物理服务器上。

作用：隔离不同的服务（比如 “新服务” 用 `vnet1`，“测试服务” 用 `vnet2`），提高安全性，避免互相干扰。

#### Zone

Zone 是 VNet 的 “容器”，可以包含多个 VNet，定义了这些 VNet 的基础网络规则（比如用什么技术隔离、用什么工具分配 IP 等）。配置中的 Zone 类型都是 `simple`（简单模式），适合单机或小规模场景。

## PVE SDN布局

### R730

- Zone: zone111 (simple)
  - VNet 1: vnet1 ‑> zone111
    - Subnet: 10.22.0.0/24 (G: 10.22.0.1)
    - SNAT: 启用
    - DHCP: 10.22.0.100 ‑ 10.22.0.200
  - VNet 2: vnet2 ‑> zone111
    - Subnet: 10.33.0.0/16 (G: 10.33.0.1)
    - SNAT: 启用
    - DHCP: 10.33.1.100 ‑ 10.33.1.200

```json
# /etc/pve/sdn/.running-config
{
  "controllers": {
    "ids": {}
  },
  "zones": {
    "ids": {
      "zone111": {
        "dhcp": "dnsmasq",
        "type": "simple",
        "ipam": "pve"
      }
    }
  },
  "subnets": {
    "ids": {
      "zone111-10.33.0.0-16": {
        "type": "subnet",
        "vnet": "vnet2",
        "dhcp-range": ["start-address=10.33.1.100,end-address=10.33.1.200"],
        "gateway": "10.33.0.1",
        "snat": 1
      },
      "zone111-10.22.0.0-24": {
        "dhcp-range": ["start-address=10.22.0.100,end-address=10.22.0.200"],
        "gateway": "10.22.0.1",
        "vnet": "vnet1",
        "type": "subnet",
        "snat": 1
      }
    }
  },
  "version": 7,
  "vnets": {
    "ids": {
      "vnet2": {
        "zone": "zone111",
        "type": "vnet"
      },
      "vnet1": {
        "zone": "zone111",
        "type": "vnet"
      }
    }
  }
}
```

### R510

- **Zone 1: recuit24 (simple)**
  - VNet: recuit24 → 10.222.0.0/16（网关：10.222.0.1），SNAT: 启用
  - DHCP地址池：10.222.0.1 ~ 10.222.0.124
  
- **Zone 2: SIMP (simple)**
  - VNet: VNSimp24 → 10.10.77.0/24（网关：10.10.77.1），SNAT: 启用
  - DHCP地址池：10.10.77.100 ~ 10.10.77.200、10.10.77.202 ~ 10.10.77.234

- **Zone 3: SimpNew (simple)**
  - VNet: SIMPSIMP → 10.10.33.0/24（网关：10.10.33.1），SNAT: 启用
  - DHCP地址池：10.10.33.100 ~ 10.10.33.200

```json
# /etc/pve/sdn/.running-config
{
  "vnets": {
    "ids": {
      "SIMPSIMP": {
        "zone": "SimpNew",
        "type": "vnet"
      },
      "recuit24": {
        "type": "vnet",
        "zone": "recuit24"
      },
      "VNSimp24": {
        "type": "vnet",
        "zone": "SIMP"
      }
    }
  },
  "subnets": {
    "ids": {
      "SIMP-10.10.77.0-24": {
        "type": "subnet",
        "dhcp-range": [
          "start-address=10.10.77.100,end-address=10.10.77.200",
          "start-address=10.10.77.202,end-address=10.10.77.234"
        ],
        "gateway": "10.10.77.1",
        "vnet": "VNSimp24",
        "snat": 1
      },
      "SimpNew-10.10.33.0-24": {
        "vnet": "SIMPSIMP",
        "gateway": "10.10.33.1",
        "type": "subnet",
        "dhcp-range": [
          "start-address=10.10.33.100,end-address=10.10.33.200"
        ],
        "snat": 1
      },
      "recuit24-10.222.0.0-16": {
        "type": "subnet",
        "dhcp-range": [
          "start-address=10.222.0.1,end-address=10.222.0.124"
        ],
        "vnet": "recuit24",
        "gateway": "10.222.0.1",
        "snat": 1
      }
    }
  },
  "controllers": {
    "ids": {}
  },
  "version": 24,
  "zones": {
    "ids": {
      "SIMP": {
        "type": "simple",
        "dhcp": "dnsmasq",
        "ipam": "pve"
      },
      "recuit24": {
        "ipam": "pve",
        "dhcp": "dnsmasq",
        "type": "simple"
      },
      "SimpNew": {
        "ipam": "pve",
        "dhcp": "dnsmasq",
        "type": "simple"
      }
    }
  }
}
```

## 24 年的招新平台架构

### 核心架构（PVE角色映射）
| 架构分层     | PVE角色              | 核心功能                        | 网络配置                  |
| ------------ | -------------------- | ------------------------------- | ------------------------- |
| 公网入口     | 公网VM（有公网IP）   | 流量转发+反向代理打通内网访问   | 公网IP，开放80端口        |
| 核心调度     | 内网核心VM（内网IP） | 流量分发+容器管理+数据库存储    | 内网IP（如192.168.1.100） |
| 动态题目运行 | Docker VM（内网IP）  | 运行Docker容器+接收容器管理指令 | 内网IP（如192.168.1.101） |
| 静态题目运行 | 静态题目VM/LXC       | 运行静态题目服务（Web/PWN等）   | 内网IP（如192.168.1.103） |

### 部署步骤
#### 公网VM
- 安装：nginx + autossh
- nginx配置：监听80端口，转发所有请求到本地9000端口（传递Host/用户IP）
- autossh配置：`autossh -M 23333 -NR 9000:192.168.1.100:9000 root@内网核心VMIP`（反向代理到内网核心VM:9000）
- 关键：免密SSH登录内网核心VM，设置autossh开机自启

#### 内网核心VM（核心部署）
- 依赖：Python3 + requests + flask + sqlite3
- 部署文件：`listener.py`（9000端口）、`docker-manager.py`（9001端口）、`db.py`、`ctf.db`（初始化3表：prob/docker/container）
- 配置要点：
  - listener.py：监听0.0.0.0:9000，正则匹配子域名（静态3位数字/动态随机字符）
  - docker-manager.py：监听0.0.0.0:9001，关联数据库+调用Docker VM接口
- 启动：创建systemd服务，设置开机自启（确保9000/9001端口监听）

#### Docker VM
- 安装：Docker引擎 + `docker-listener.py`（9000端口）
- 配置：`docker-listener.py`监听0.0.0.0:9000，接收容器启停指令
- 测试：`docker run hello-world`验证Docker可用性，设置服务开机自启

#### 静态题目VM/LXC
- 部署：运行静态题目服务（如Web开80端口、PWN开9999端口）
- 配置：内网核心VM的`ctf.db`中插入题目记录（id+IP+端口）

### 核心流量链路
1. 用户请求→公网VM:80→nginx转发到本地9000端口
2. autossh反向代理→内网核心VM:9000（静态）/9001（容器管理）
3. 静态题目：listener.py查询`prob`表→转发到静态题目VM端口
4. 动态题目：docker-manager.py调用Docker VM:9000→启动容器→listener.py查询`container`表→转发到容器映射端口

### 关键配置/验证要点
- 网络：所有内网VM在同一网段，开放必要端口（9000/9001/容器端口）
- 自启：所有服务（nginx/autossh/listener/docker-manager/docker-listener）均设置systemd开机自启
- 数据库：`ctf.db`需初始化3表，静态题目/动态模板需手动插入记录
- 排查：用`systemctl status 服务名`查状态，`netstat -tulpn`查端口监听，确保免密SSH/容器端口可访问
