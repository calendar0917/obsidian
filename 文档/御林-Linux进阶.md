---
title: "御林招新题：Linux 进阶"
subtitle: "御林招新题-Linux 进阶"
summary: "学习关于 Linux 的更深层知识"
description: "学习关于 Linux 的更深层知识"
image: ""
date: 2025-10-17
lastmod: 2025-10-17
hiddenFromHomePage: True
draft: false
toc:
 enable: true
weight: false
categories: ["CTF"]
tags: ["CTF"]
---

## 小结

1. ps 命令查看进程
   - losf、netstat 查看端口进程号
   - kill 终结进程
2. Linux 文件结构
3. grep + 正则 + 管道查找指定内容
   - wc 统计字符
4. 自动化脚本 .sh
   - systemctl 控制 service 终止
5. 内核模块
   - lsmod 列出
   - modprobe 控制
6. btrfs 控制磁盘、分卷
   - mount 挂载
   - snapshot 创建快照、回滚
7. journalctl 查看、删除日志文件
8. sysctl 查看、修改内核参数
   - 控制 TCP 最大连接数：`net.core.somaxconn & net.ipv4.tcp_max_syn_backlog`

## 系统与进程管理

- **特殊进程**：描述进程号为 `1` 的特殊进程是什么，以及它在系统中的作用。

> 进程号为 1 的特殊进程在类 Unix 系统（如 Linux）中通常是`init`进程
>
> - **作用**：它是系统启动后创建的第一个用户空间进程，是所有其他进程的祖先进程。负责系统初始化，比如启动各种服务、设置运行级别等，系统关闭时也由它来完成相关收尾工作，确保系统各部分有序启动和停止。
> - 由 0 进程创建，内核启动 -> init启动 -> 启动其他进程

- **进程状态**：使用 `ps` 命令查看所有活跃进程，并分析其父子关系。

```shell
[root@localhost ~]# ps -A -f
UID         PID   PPID  C STIME TTY          TIME CMD
root          1      0  0 23:25 ?        00:00:03 /usr/lib/systemd/systemd --switched-root --syste
root          2      0  0 23:25 ?        00:00:00 [kthreadd]
root          4      2  0 23:25 ?        00:00:00 [kworker/0:0H]
```

> PID：进程 ID
>
> PPID：进程的父 PID

- **端口占用**：当某个程序无法启动并提示端口已被占用时，如何通过命令行找出占用该端口的进程号（PID）并终止它。

> 查看端口进程：
>
> - lsof（List Open Files ）
> - netstat（Network Statistics）
>   - netstat | grep PID
>   - `-t`：显示 **TCP** 协议的连接 / 端口（Transmission Control Protocol，传输控制协议）。
>   - `-u`：显示 **UDP** 协议的连接 / 端口（User Datagram Protocol，用户数据报协议）。
>   - `-l`：仅显示 **处于监听状态** 的端口（即等待外部连接的端口，而非已建立的连接）。
>   - `-p`：显示 **占用端口的进程信息**（包括进程 ID 和进程名），需要 root 权限才能完整显示。
>   - `-n`：以 **数字形式** 显示 IP 地址和端口号（而非域名或服务名，例如直接显示 `127.0.0.1:8080` 而非 `localhost:http-alt`）。
>
> 终止：
>
> - kill -pid PID 强制终止进程

## 网络与文件系统

- **文件系统结构**：描述 Linux 文件系统（例如 ext4）的基本目录结构和作用，并解释根目录 `/` 和 `/home` 的区别。

> 基本目录结构：树形
>
> ![image-20251017150043653](https://raw.githubusercontent.com/calendar0917/images/master/image-20251017150043653.png)
>
> - **`/`（根目录）**：最顶层目录，所有其他目录和文件都从这里开始分支，包含了系统运行所需的所有核心文件和目录，一般情况下普通用户只能读取
>   - **`/home`**：是普通用户的主目录集合，**每个普通用户在系统中都有**一个以自己用户名命名的子目录（如用户 `user1` 的主目录是 `/home/user1`）。用户在自己的主目录下有完全的操作权限，可以存储个人文件、配置个人环境等，不会影响系统的核心部分。
> - **`/bin`**：存放系统的**基本命令**，这些命令是二进制可执行文件，普通用户和超级用户都可以执行，用于完成基本的系统操作，如 `ls`（列出目录内容）、`cp`（复制文件）等。
> - **`/sbin`**：存放**系统管理命令**，通常只有超级用户（root）才能执行，用于系统管理和维护，如 `ifconfig`（配置网络接口）、`shutdown`（关闭系统）等。
> - **`/etc`**：存放系统的**配置文件**，包括系统服务配置、用户配置、网络配置等，如 `passwd`（用户账户信息）、`fstab`（文件系统挂载配置）等。
> - **`/dev`**：存放**设备文件**，Linux 把所有的硬件设备都抽象为文件，通过这些文件可以访问和控制硬件设备，如 `sda`（第一块硬盘）、`tty1`（第一个终端）等。
> - **`/proc`**：是一个**虚拟文件系统**，它不占用实际的磁盘空间，而是反映系统当前的运行状态，包含了进程信息、内存使用情况等，如 `/proc/cpuinfo`（CPU 信息）、`/proc/meminfo`（内存信息）等。
> - **`/usr`**：存放**用户安装的应用程序和文件**，类似于 Windows 系统中的 “Program Files” 目录，包含了大量的应用程序、库文件、文档等。
> - **`/var`**：存放**经常变化的文件**，如日志文件（`/var/log`）、邮件（`/var/mail`）、打印队列（`/var/spool`）等。
> - **`/tmp`**：存放**临时文件**，系统**重启后这里的文件会被清除**，用于程序运行时临时存储数据。

- **GRUB**：什么是 GRUB？它在 Linux 系统启动中扮演了什么角色？

> Grand Unified Bootloader 统一引导加载程序
>
> - **系统启动引导**：计算机开机后，首先由 BIOS（基本输入输出系统）或 UEFI（统一可扩展固件接口）进行硬件检测和初始化，然后会将引导加载程序（GRUB）加载到内存中运行。**GRUB 负责加载 Linux 内核，并将系统控制权转交给内核**，从而启动 Linux 系统。
> - **多系统引导**：如果计算机上安装了多个操作系统（如同时安装了 Linux 和 Windows），GRUB 可以提供一个**启动菜单**，让用户选择要启动的操作系统。
> - **内核参数设置**：在系统启动时，用户可以通过 GRUB 的启动菜单，**临时修改内核的启动参数**，这对于系统调试、故障排除（如单用户模式启动）等非常有用。

## 文件内容处理

- **查找模式**：使用 `grep` 命令在 `/etc/passwd` 文件中，找出所有以 `t` 开头的文本行。

```shell
[root@localhost ~]# grep '^t' /etc/passwd
tss:x:59:59:Account used ......
tcpdump:x:72:72::/:/sbin/nologin
```

- **管道与重定向**：使用管道和重定向，将 `/etc/passwd` 文件中所有包含数字的行数统计出来，并重定向到一个名为 `digit_count.txt` 的文件中。

```shell
[root@localhost ~]# grep -E [0-9] /etc/passwd | wc -l > hello/digit_count.txt

// 统计结果：44
```

## 高级系统管理

- **systemd**：

  - 编写一个简单的 `systemd service` 文件，让一个简单的脚本（例如，一个每分钟向 `/tmp/test_log.txt` 文件写入当前时间的脚本）开机自启动。

  ```sh
  //time.sh
  #!/bin/bash
  while true; do
      echo "当前时间: $(date)" >> /tmp/test_log.txt
      sleep 60
  done
  
  //service
  [Unit]
  Description=定时写入时间到日志的服务
  
  [Service]
  ExecStart=/home/your_username/time_script.sh
  Restart=always
  
  [Install]
  WantedBy=multi-user.target
  ```

  ```shell
  [root@localhost ~]# sudo systemctl enable time-logger.service
  Created symlink from /etc/systemd/system/multi-user.target.wants/time-logger.service to /etc/syste          md/system/time-logger.service.
  
  [root@localhost ~]# systemctl start time-logger.service
  
  [root@localhost ~]# systemctl status time-logger.service
  ● time-logger.service - 定时写入时间到日志的服务
     Loaded: loaded (/etc/systemd/system/time-logger.service; enabled; vendor preset: disabled)
     Active: active (running) since Fri 2025-10-17 00:48:53 PDT; 2s ago
   Main PID: 29493 (time_script.sh)
      Tasks: 2
     Memory: 304.0K
     CGroup: /system.slice/time-logger.service
             ├─29493 /bin/bash /root/time_script.sh
             └─29497 sleep 60
  
  [root@localhost ~]# cat /tmp/test_log.txt
  当前时间: Fri Oct 17 00:48:53 PDT 2025
  ```

  - 使用 `systemctl` 命令启用和启动你的服务，并验证它是否正常工作。

- **内核模块**：

  - 列出系统中所有已加载的内核模块。

  ```shell
  [root@localhost ~]# lsmod
  Module                  Size  Used by
  nf_conntrack_netlink    36396  0
  xt_addrtype            12676  2
  .....
  ```

  - 尝试加载一个你了解的内核模块（例如，一个虚拟网络设备模块），然后卸载它，并验证其状态。

  ```shell
  [root@localhost ~]# modprobe dummy
  [root@localhost ~]# lsmod | grep dummy
  dummy                  12960  0
  [root@localhost ~]# modprobe -r dummy
  [root@localhost ~]# lsmod | grep dummy
  [root@localhost ~]#
  ```

  > 什么是 dummy？
  >
  > 在 Linux 系统中，`dummy`模块是一种虚拟网络设备模块，主要有以下特点和用途：
  >
  > - **虚拟网络接口创建**：加载`dummy`模块后，通过系统命令可以创建出像 `dummy0`、`dummy1` 这样的虚拟网络接口。这些接口和真实的物理网络接口类似， 可以配置 IP 地址、子网掩码等网络参数 ，也能参与网络数据包的收发模拟。
  > - **网络测试与实验**：在进行网络相关的测试，比如路由策略测试、防火墙规则测试时，使用`dummy`虚拟网络接口非常方便。不需要依赖实际的物理网络设备，就可以构建复杂的网络拓扑结构，模拟各种网络环境。
  > - **服务绑定与隔离**：某些应用场景下，需要将特定的网络服务绑定到指定的网络接口上，使用`dummy`接口可以实现灵活的绑定和隔离。比如，希望某个服务只在特定的 “网络通道” 上提供服务，就可以将该服务绑定到`dummy`接口上，便于管理和安全控制。

## 现代文件系统

**Btrfs/ZFS 文件系统实践**：

- **创建文件系统与子卷**：

  - 假设你有一个未使用的磁盘分区 `/dev/sdb1`。

  - **格式化**该分区为 `Btrfs` 或 `ZFS` 文件系统。

  - **挂载**该文件系统到一个新的目录，例如 `/mnt/data`。

  - 在 `/mnt/data` 下**创建两个子卷**，一个名为 `web_data`，另一个名为 `database`。

  ```shell
  由于没有磁盘分区，故用虚拟磁盘代替
  [root@localhost ~]# dd if=/dev/zero of=~/virtual_disk.img bs=1G count=1
  // 创建 1G * 1 的虚拟磁盘
  [root@localhost ~]# losetup -f ~/virtual_disk.img
  [root@localhost ~]# losetup -a | grep virtual_disk.img
  /dev/loop0: [2051]:67154732 (/root/virtual_disk.img)
  // 自动映射,格式化
  [root@localhost ~]# mkfs.btrfs /dev/loop0 
  btrfs-progs v4.9.1
  See http://btrfs.wiki.kernel.org for more information.
  
  Performing full device TRIM /dev/loop0 (1.00GiB) ...
  ......
  // 创建映射文件系统新目录并挂载
  [root@localhost ~]# mkdir /mnt/data
  [root@localhost ~]# mount /dev/loop0 /mnt/data
  
  // 创建两个子卷
  [root@localhost ~]# btrfs subvolume create /mnt/data/web_data
  Create subvolume '/mnt/data/web_data'
  [root@localhost ~]# btrfs subvolume create /mnt/data/database
  Create subvolume '/mnt/data/database'
  [root@localhost ~]# btrfs subvolume list /mnt/data
  ID 256 gen 7 top level 5 path web_data
  ID 257 gen 8 top level 5 path database
  
  ```

- **快照与回滚**：

  - 在 `web_data` 子卷中创建一个测试文件 `hello.txt`。

  - 为 `web_data` 子卷**创建一个只读快照**，命名为 `snapshot_v1`。

  - **修改** `web_data` 子卷中的 `hello.txt` 文件内容。

  - **验证** `snapshot_v1` 中的文件内容是否保持不变。

  - **回滚** `web_data` 子卷到 `snapshot_v1` 的状态，并验证 `hello.txt` 的内容是否恢复到修改前。

  ```shell
  // 创建文件
  [root@localhost ~]# echo 'helloworld 111' > /mnt/data/web_data/hello.txt
  // 创建只读快照
  [root@localhost ~]# btrfs subvolume snapshot -r /mnt/data/web_data /mnt/data/snapshot_v1
  Create a readonly snapshot of '/mnt/data/web_data' in '/mnt/data/snapshot_v1'
  // 修改文件
  [root@localhost ~]# echo 'helloworld 222' > /mnt/data/web_data/hello.txt
  //读快照，并未改变
  [root@localhost ~]# cat /mnt/data/snapshot_v1/hello.txt
  helloworld 111
  // 删除源文件，快照恢复
  [root@localhost ~]# btrfs subvolume delete /mnt/data/web_data
  Delete subvolume (no-commit): '/mnt/data/web_data'
  [root@localhost ~]# btrfs subvolume snapshot /mnt/data/snapshot_v1/ /mnt/data/web_data
  Create a snapshot of '/mnt/data/snapshot_v1/' in '/mnt/data/web_data'
  [root@localhost ~]# cat /mnt/data/web_data/hello.txt
  helloworld 111
  ```

  

## 系统优化与维护

- **缓存与日志管理**：

  - 使用 `journalctl` 命令查看过去 10 分钟内系统日志中所有包含“error”或“fail”关键字的条目。

  - 通过命令行安全地**清除**超过 7 天的系统日志文件，以释放磁盘空间。

  ```shell
  [root@localhost ~]# journalctl --since "100 minute ago" | grep -E "error|fail"
  Oct 17 00:46:03 localhost.localdomain systemd[1]: Unit time-logger.service entered failed state.
  Oct 17 00:46:03 localhost.localdomain systemd[1]: time-logger.service failed.
  ......
  
  [root@localhost ~]# journalctl --vacuum-time=7d
  Vacuuming done, freed 0B of archived journals on disk.
  ```

- **内核参数调整**：

  - **查看**当前系统中的 `TCP` 最大连接数限制参数。

  - **临时修改**该参数，将最大连接数限制提高到 `50000`。

  - 使用 `sysctl -a` 命令**验证**修改是否生效。

  ```shell
  [root@localhost ~]# sysctl net.core.somaxconn net.ipv4.tcp_max_syn_backlog
  net.core.somaxconn = 128
  net.ipv4.tcp_max_syn_backlog = 256
  [root@localhost ~]# sysctl -w net.core.somaxconn=50000
  net.core.somaxconn = 50000
  
  [root@localhost ~]# sysctl -a | grep -E "50000"
  ......
  net.core.somaxconn = 50000
  ......
  ```

  - **解释**此项修改对一个高并发 Web 服务器可能带来的影响。

> 提高 TCP 最大连接数限制（如 `net.core.somaxconn` 和 `net.ipv4.tcp_max_syn_backlog`），对高并发 Web 服务器有以下影响：
>
> - 优化：
>   - 能够处理更多的并发 TCP 连接请求，减少因连接队列满而导致的连接建立失败情况，**提升 Web 服务器的并发处理能力**，让更多客户端能成功与服务器建立连接并请求资源。
> - 问题：
>   - 会增加服务器的内存等**资源消耗**，因为每个连接都需要一定的内存来维护连接状态等信息。如果服务器内存等资源不足，过度提高连接数限制可能导致系统内存紧张，甚至出现内存溢出等问题，反而影响服务器的稳定运行。