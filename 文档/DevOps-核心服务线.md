---
title: "DevOps核心服务线"
subtitle: "DevOps核心服务线"
summary: "学习记录"
description: "学习记录"
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



参考：[DevOps入门小结](https://calendar0917.github.io/posts/%E5%BE%A1%E6%9E%97-devops%E5%85%A5%E9%97%A8%E5%B0%8F%E7%BB%93/)

## Git版本控制

### 核心流程

#### 四大核心区域定义

1. **工作区（Working Directory）**：即本地电脑中存放项目文件的实际目录，是开发者日常编写代码、修改文件的区域，文件状态分为 “未跟踪（Untracked）” 和 “已跟踪（Tracked）”。
   - 未跟踪：新创建的文件，Git 未记录其任何版本信息；
   - 已跟踪：曾通过 `git add` 纳入版本控制的文件，包括 “已修改（Modified）”“已暂存（Staged）”“未修改（Unmodified）” 三种状态。
2. **暂存区（Staging Area/Index）**：是 Git 中的 “临时缓冲区”，位于 `.git` 目录内，用于存放待提交的文件变更。通过 `git add` 命令可将工作区的修改同步到暂存区，暂存区的内容会在执行 `git commit` 时被提交到本地仓库。
3. **本地仓库（Local Repository）**：位于项目根目录的 `.git` 隐藏文件夹，是 Git 存储版本历史的核心区域。包含所有提交记录、分支信息、对象数据库（存储文件内容、提交对象等），即使断开网络也能完整查看历史和管理版本。
4. **远程仓库（Remote Repository）**：位于服务器（如 GitHub、GitLab、自建 Git 服务器）的共享仓库，用于团队成员同步代码、协同开发。通过 `git push` 将本地仓库的提交推送到远程，通过 `git pull/fetch` 从远程获取他人的提交。

#### 命令与状态转换

| 命令                     | 作用                       | 状态转换                         |
| ------------------------ | -------------------------- | -------------------------------- |
| `git add <文件>`         | 将工作区修改添加到暂存区   | 未跟踪 → 已暂存；已修改 → 已暂存 |
| `git commit -m "备注"`   | 将暂存区内容提交到本地仓库 | 已暂存 → 未修改                  |
| `git push <远程> <分支>` | 本地仓库提交推送到远程仓库 | 本地提交 → 远程提交              |
| `git checkout -- <文件>` | 丢弃工作区未暂存的修改     | 已修改 → 未修改                  |
| `git reset HEAD <文件>`  | 将暂存区的修改撤回到工作区 | 已暂存 → 已修改                  |

### 协同操作

**git fetch：仅获取，不合并**

- 原理：从远程仓库拉取本地尚未有的提交记录，更新本地的远程跟踪分支（如 `origin/main`），但不会修改本地当前工作分支的代码。
- 流程：`git fetch origin main` → 拉取远程 `main` 分支的新提交 → 存储到本地 `origin/main` 分支 → 本地 `main` 分支保持不变。
- 优势：可在合并前通过 `git diff main origin/main` 查看远程提交与本地的差异，评估是否会引发冲突，更安全可控。

**git pull：获取并合并**

- 原理：`git pull` 本质是 `git fetch` + `git merge` 的组合命令，拉取远程提交后会自动将其合并到本地当前工作分支。
- 流程：`git pull origin main` → 先执行 `git fetch origin main` → 再执行 `git merge origin/main into main` → 直接更新本地工作分支代码。
- 优势：操作简洁，一步完成 “获取 + 合并”，适合快速同步代码；劣势：若存在冲突，会直接进入合并冲突解决流程，新手可能难以处理。

#### 最佳实践

团队协作优先使用 `git fetch` + `git merge` 组合

```bash
git fetch origin main  # 获取远程新提交
git diff main origin/main  # 查看差异
git merge origin/main  # 手动合并（冲突可提前预判）
```

若确认远程提交与本地无冲突（如个人分支同步），可直接用 `git pull` 简化操作；

避免在多人协作的公共分支（如 `main`）上直接使用 `git pull`，建议先切换到个人分支，合并后再提交 PR/MR。

### 分支管理

核心分支概念

- **主分支（main/master）**：存放稳定可发布的代码，通常不允许直接提交，仅通过合并其他分支更新；
- **功能分支（feature/\*）**：用于开发新功能，从主分支创建，功能完成后合并回主分支；
- **修复分支（hotfix/\*）**：用于修复生产环境紧急 bug，从主分支创建，修复后同时合并回主分支和开发分支。

核心命令：

- **创建并切换到新分支**：`git checkout -b <new_branch>`

  - 原理：等价于 `git branch <new_branch>`（创建分支） + `git checkout <new_branch>`（切换分支）；

- **合并分支**：`git merge <branch>`

  - 原理：将指定分支的代码合并到当前所在分支；

  - 如功能开发完成后，将 `feature/login` 合并到 `main` 分支：

    ```bash
    git checkout main  # 切换到主分支
    git merge feature/login  # 合并功能分支
    ```

  - 冲突处理：若合并时出现代码冲突（同一文件同一位置被不同修改），Git 会标记冲突文件，需手动编辑冲突内容后，执行 `git add <冲突文件>` + `git commit` 完成合并。

- **删除分支**：`git branch -d <branch>`

  - 作用：删除已合并到主分支的无用分支，清理分支结构；
  - 强制删除：若分支未合并（如废弃的功能分支），需用 `git branch -D <branch>`（大写 D）强制删除；

### 历史管理

**git reset --hard <commit_id>：本地强制回滚**

- 原理：将本地仓库、暂存区、工作区强制重置到指定的提交版本，丢弃该版本之后的所有提交记录；
- 直接修改本地版本历史，操作不可逆（丢弃的提交无法通过常规命令恢复），仅适用于本地未推送的提交。
- 严禁在公共分支使用！！

**git revert <commit_id>：创建反向提交**

- 原理：不删除原有提交记录，而是创建一个新的 “反向提交”，抵消指定提交的修改内容，保持版本历史的完整性；
- 修改会生成新的提交，可正常推送到远程仓库，适合撤销已推送到公共分支的错误提交。
- 公共分支优先使用

## Docker与容器化

之前的文档写得比较详细了。

## 网络服务和转发

### Nginx 反向代理配置

```nginx
server {
    listen 80;
    server_name api.example.com;

    # 匹配 /user 开头的请求，转发到用户服务
    location /user {
        proxy_pass http://user-service:3000;  # 可使用内网域名（需DNS或hosts解析）
        proxy_set_header Host $host;
    }

    # 匹配 /order 开头的请求，转发到订单服务
    location /order {
        proxy_pass http://order-service:4000;
        proxy_set_header Host $host;
    }

    # 匹配静态文件（如图片、CSS），直接返回本地文件
    location ~* \.(jpg|jpeg|png|css|js)$ {
        root /var/www/static;  # 静态文件目录
        expires 1d;  # 缓存1天，减轻后端压力
    }
}
```

### 内网穿透

#### SSH 反向隧道

> **原理**：
>
> 通过 SSH 连接将内网服务的端口 “反向映射” 到公网服务器（有公网 IP），使得公网用户可通过公网服务器的指定端口访问内网服务。

#### 持久化隧道（autossh）：

`autossh` 可自动检测 SSH 连接状态并重启隧道，确保稳定性：

```bash
# 安装 autossh（Debian/Ubuntu）
sudo apt install autossh

# 启动持久化反向隧道：-M 指定监控端口，-f 后台运行
autossh -M 20000 -fN -R 8000:localhost:8080 user@public-server-ip
```

- **适用场景**：轻量需求（如个人博客、小型服务），无需额外服务端配置（仅需公网服务器支持 SSH）。

#### Frp 的 C/S 工作模式

> Frp 是专门的内网穿透工具，采用客户端（frpc）/ 服务端（frps）架构，支持 TCP、UDP、HTTP 等多种协议，功能更强大。
>
> 工作原理：
>
> - **服务端（frps）**：部署在有公网 IP 的服务器，监听客户端连接和用户访问请求。
> - **客户端（frpc）**：部署在内网机器，与服务端建立长连接，将内网服务端口映射到服务端。

配置示例：

1. **服务端（frps.ini）**：

   ```ini
   [common]
   bind_port = 7000  # 与客户端通信的端口
   vhost_http_port = 8080  # HTTP 服务统一端口（可选）
   ```

   启动：`frps -c frps.ini`

2. **客户端（frpc.ini）**：

   ```ini
   [common]
   server_addr = public-server-ip  # 服务端公网 IP
   server_port = 7000  # 服务端 bind_port
   
   # 映射内网 SSH 服务（22 端口）
   [ssh]
   type = tcp
   local_ip = 127.0.0.1
   local_port = 22
   remote_port = 6000  # 访问服务端 6000 端口 → 内网 22 端口
   
   # 映射内网 Web 服务（8080 端口）
   [web]
   type = http
   local_ip = 127.0.0.1
   local_port = 8080
   custom_domains = web.example.com  # 绑定域名（需 DNS 解析到服务端 IP）
   ```

   启动：`frpc -c frpc.ini`

### 零配置网络

具体使用看以前文章，这里分析原理

> Tailscale 是基于 WireGuard 协议的零配置 VPN 工具，可快速构建跨网络的私有网络（如办公室、家庭、云服务器之间），实现设备间直接通信。
>
> > 相当于让分散的网络组成一个私有网络，实现跨网直连
>
> 核心原理：
>
> - **P2P 直连**：通过 STUN 协议检测设备是否可直接通信，优先建立点对点连接（无需中转服务器），速度快、延迟低。
> - **中继 fallback**：若设备位于严格 NAT 后无法直连，自动通过 Tailscale 中继服务器转发（加密传输）。
> - **身份认证**：基于 Tailscale 账号系统（支持 Google、GitHub 等 OAuth 登录），设备加入网络需管理员授权，安全可控。
> - **DNS 与子网路由**：自动分配虚拟 IP（如 `100.x.y.z`），支持自定义 DNS 名称（如 `my-laptop.tailnet-name.ts.net`），可通过子网路由暴露整个内网（如办公室网段）。
>
>  优势场景：
>
> - 远程办公：在家访问公司内网服务器，无需复杂 VPN 配置；
> - 多地域设备组网：云服务器（AWS / 阿里云）、本地服务器、个人电脑组成私有网络，互访无感知；
> - 安全通信：所有流量通过 WireGuard 加密，且无需暴露公网端口。

Q:为什么普通网络下 “跨网直连” 很难？

- **NAT 隔离**：家用 / 公司路由器会给设备分配 “内网 IP”（如 `192.168.x.x`），这些 IP 只在路由器内部有效，互联网上看不见。路由器通过 “NAT 转换” 把内网 IP 映射到唯一的公网 IP 上，但这个公网 IP 通常是动态的，且路由器会阻止外部主动访问内网设备。
- **防火墙限制**：公司 / 家庭网络的防火墙会默认拦截陌生的外部连接，防止被攻击。
- **没有统一身份**：两台设备不知道对方是谁，无法验证 “是否允许通信”。

Q:Tailscale 如何解决这些问题？

- P2P 直连：绕开 “中间商”，设备直接对话

- Tailscale 优先让设备之间 “直接握手”，不用经过第三方服务器转发，速度最快。

- **关键技术：STUN 协议**STUN 协议的作用是让设备 “发现自己在互联网中的真实位置”。比如：

  - 家里的电脑会先问 Tailscale 的 STUN 服务器：“我在互联网上的地址和端口是多少？”
  - STUN 服务器回复：“你当前的公网 IP 是 `203.0.113.5`，端口是 `50000`（路由器给你分配的临时端口）。”
  - 公司的服务器也会做同样的操作，得到自己的公网 IP 和端口。
  - Tailscale 再把这两个 “地址 + 端口” 告诉对方，两台设备就可以直接发送数据了（比如家里电脑直接访问公司服务器的 `203.0.113.10:50001`）。

- **中继 fallback：实在连不上？找 “中介” 帮忙**

  如果设备在 “严格的 NAT 环境” 下（比如某些公司网络、校园网），STUN 也无法让它们直连，这时 Tailscale 会自动切换到 “中继模式”。

  - **原理**：Tailscale 会在全球部署多台 “中继服务器（DERP 服务器）”，两台设备都先连接到同一台中继服务器，数据通过中继转发。
  - **安全性**：即使经过中继，数据也是端到端加密的（中继服务器看不到内容），只有通信的两台设备能解密。

- **身份认证：确保 “连对人”，防止陌生人闯入**

  Tailscale 用一套严格的身份体系确保只有授权设备能加入网络：

  - **账号绑定**：所有设备必须登录同一个 Tailscale 账号（支持 Google、GitHub 等账号登录），相当于 “家庭住址”，只有这个 “家庭” 的设备才能互相看到。
  - **授权机制**：新设备登录后，需要管理员在 Tailscale 后台手动 “批准” 才能加入网络（类似小区门禁，保安确认后才放行）。
  - **加密证书**：每台设备会生成一对加密密钥，由 Tailscale 的 “密钥服务器” 颁发证书，通信时用证书验证对方身份（确保不是伪装的设备）。

- **子网路由：让 “整个内网” 都加入虚拟网络**

  - 如果你想让虚拟网络里的设备访问 “整个公司内网”（比如公司内网有 `192.168.1.x` 网段的多台服务器），不用每台都装 Tailscale，只需：

  - 在内网找一台 “网关设备”（比如一台服务器）安装 Tailscale，并配置 “子网路由”，告诉 Tailscale：“我能访问 `192.168.1.0/24` 这个网段”。
  - 其他设备（如家里的电脑）就可以通过这台网关设备访问公司内网的所有机器（如 `192.168.1.100`）。

Q：为什么能绕过防火墙？

- 因为这个连接是由 “内网设备主动发起” 的，大多数防火墙会允许 “主动发起的连接的返回数据”。

### 文件服务

具体看以前文章，这里做一下整合

> 文件服务用于跨设备、跨系统共享文件，SMB/CIFS 适合 Windows 环境，SFTP 基于 SSH 更安全通用。

#### SMB/CIFS（Samba）：Windows 友好的文件共享

> Samba 实现了 SMB 协议，可让 Linux 与 Windows、macOS 无缝共享文件。

核心配置（`/etc/samba/smb.conf`）：

```ini
[global]
   workgroup = WORKGROUP  # 与 Windows 工作组一致
   security = user        # 用户密码认证
   map to guest = bad user  # 拒绝匿名访问

[public]  # 共享名称（Windows 中显示为 \\server\public）
   path = /var/samba/public  # 共享目录
   browsable = yes           # 允许浏览
   writable = yes            # 可写
   guest ok = no             # 不允许匿名访问
   valid users = alice bob   # 允许访问的用户

[private]
   path = /home/alice/private
   writable = yes
   valid users = alice       # 仅 alice 可访问
   create mask = 0600        # 新建文件权限
   directory mask = 0700     # 新建目录权限
```

用户管理：

- Samba 依赖系统用户，但密码独立管理：

```bash
# 创建系统用户（若不存在）
sudo useradd -m alice

# 设置 Samba 密码（需输入两次）
sudo smbpasswd -a alice  # -a 添加用户，-x 删除用户

# 重启 Samba 服务
sudo systemctl restart smbd
```

访问方式：

- Windows：`Win + R` 输入 `\\samba-server-ip\public`，输入 Samba 用户名密码；
- Linux：`smbclient //samba-server-ip/public -U alice` 或挂载到本地 `mount -t cifs //server/public /mnt -o username=alice`。

#### SFTP（基于 SSH）：安全的文件传输

> SFTP 是 SSH 的子协议，无需额外服务，仅需开启 SSH 服务即可使用，适合跨平台且对安全性要求高的场景。

限制用户目录（ChrootDirectory）：

默认 SFTP 允许用户访问整个系统，通过配置可限制用户仅能访问指定目录（增强安全性）：

1. 修改 SSH 配置（`/etc/ssh/sshd_config`）：

   ```bash
   # 注释原 Subsystem sftp /usr/lib/openssh/sftp-server
   Subsystem sftp internal-sftp  # 使用内置 SFTP 模块
   
   # 针对 sftpuser 组的配置
   Match Group sftpuser
       ChrootDirectory /var/sftp/%u  # 限制根目录（必须为 root 所有，权限 755）
       ForceCommand internal-sftp    # 强制使用 SFTP，禁止 SSH 登录
       AllowTcpForwarding no
       X11Forwarding no
   ```

2. 创建用户和目录：

   ```bash
   # 创建 sftpuser 组
   sudo groupadd sftpuser
   
   # 创建用户（禁止登录 shell）
   sudo useradd -m -g sftpuser -s /usr/sbin/nologin sftp_alice
   
   # 设置密码
   sudo passwd sftp_alice
   
   # 创建 Chroot 目录（权限必须正确，否则登录失败）
   sudo mkdir -p /var/sftp/sftp_alice/upload
   sudo chown root:root /var/sftp/sftp_alice  # 根目录必须为 root 所有
   sudo chmod 755 /var/sftp/sftp_alice
   sudo chown sftp_alice:sftpuser /var/sftp/sftp_alice/upload  # 子目录可写
   ```

3. 重启 SSH 服务：`sudo systemctl restart sshd`

访问方式：

- 命令行：`sftp sftp_alice@server-ip`（默认端口 22）；
- 图形工具：FileZilla、WinSCP（输入主机、端口、用户名密码）。

## 数据与应用

补充一下自动化备份

### 自动化备份：`crontab` 定时任务

通过 Linux 的`crontab`设置定时任务，可实现 MySQL 自动备份，避免人工操作遗漏。

（1）编写备份脚本（`mysql_backup.sh`）

```bash
#!/bin/bash
# 备份目录
BACKUP_DIR="/data/mysql_backups"
# 日期格式（如20251109）
DATE=$(date +%Y%m%d)
# 保留备份天数（删除7天前的备份）
RETENTION_DAYS=7

# 创建备份目录（若不存在）
mkdir -p $BACKUP_DIR

# 执行备份（单库示例，替换为实际库名和密码）
mysqldump -u root -p"your_password" --databases blog --single-transaction | gzip > $BACKUP_DIR/blog_$DATE.sql.gz

# 删除过期备份
find $BACKUP_DIR -name "blog_*.sql.gz" -type f -mtime +$RETENTION_DAYS -delete
```

- 注意：脚本中直接写密码有安全风险，建议通过`~/.my.cnf`配置无密码登录（见下方补充）。

（2）添加执行权限

```bash
chmod +x mysql_backup.sh
```

（3）设置`crontab`定时任务

```bash
# 编辑定时任务
crontab -e

# 添加一行：每天凌晨3点执行备份
0 3 * * * /path/to/mysql_backup.sh
```

- 格式说明：`分 时 日 月 周 命令`，`0 3 * * *` 表示每天 3 点 0 分执行。

（4）补充：无密码登录配置（`~/.my.cnf`）

```ini
[client]
user=root
password=your_password
```

设置权限：`chmod 600 ~/.my.cnf`，避免密码泄露。

### Python 后端

> Python 后端框架基于不同的网关协议，分为同步（WSGI）和异步（ASGI）两类

#### WSGI 与 ASGI：协议本质与区别

**WSGI**（Web Server Gateway Interface）：同步网关协议

- WSGI 是 Python Web 应用与 Web 服务器（如 Nginx、Gunicorn）之间的标准接口，定义了二者的通信规范，**仅支持同步请求处理**。
- **工作原理**：
  - 客户端请求到达 Web 服务器（如 Nginx），转发给 WSGI 服务器（如 Gunicorn）；
  - WSGI 服务器为每个请求创建一个线程 / 进程，调用应用框架（如 Flask）的处理函数；
  - 处理函数同步执行（执行期间线程 / 进程被阻塞），完成后返回响应。
- **代表框架**：Flask、Django（默认）。
- **局限性**：
  - 面对 IO 密集型任务（如数据库查询、网络请求）时，线程 / 进程会被阻塞，无法处理其他请求，并发能力有限；
  - 不支持 WebSocket 等长连接协议。

**ASGI**（Asynchronous Server Gateway Interface）：异步网关协议

- ASGI 是 WSGI 的继任者，**支持同步和异步请求处理**，专为高并发和长连接场景设计。
- **工作原理**：
  - 基于事件循环（Event Loop）机制，用少量线程处理大量请求；
  - 遇到 IO 操作（如等待数据库响应）时，会释放当前线程去处理其他请求，IO 完成后再回调继续执行；
  - 原生支持 WebSocket、HTTP/2 等协议。
- **代表框架**：FastAPI、Starlette、Django 3.0+（异步视图）。
- **优势**：
  - 处理 IO 密集型任务时，并发能力比 WSGI 高 10 倍以上；
  - 支持长连接和异步语法，适合实时应用（如聊天、通知）。

#### FastAPI

> FastAPI 是基于 ASGI 的高性能框架，原生支持异步语法和数据校验，开发效率和运行性能兼具。

`async/await` 异步语法

用于定义异步函数，实现非阻塞 IO 操作，提升并发能力。

- **基础示例**：

  ```python
  from fastapi import FastAPI
  import asyncio
  
  app = FastAPI()
  
  # 异步路由函数（用async def定义）
  @app.get("/async")
  async def async_endpoint():
      # 模拟IO操作（如数据库查询、网络请求）
      await asyncio.sleep(1)  # 非阻塞等待1秒（不占用线程）
      return {"message": "异步响应"}
  
  # 同步路由函数（用def定义，兼容同步代码）
  @app.get("/sync")
  def sync_endpoint():
      # 同步IO操作（会阻塞线程）
      time.sleep(1)
      return {"message": "同步响应"}
  ```

- **关键规则**：
  - 用`async def`定义的函数可使用`await`调用其他异步函数（如异步数据库驱动`asyncpg`）；
  - 若调用同步 IO 函数（如`requests.get`），需用`async def`+ 线程池（避免阻塞事件循环）；
  - 异步函数中**不能有阻塞操作**（如`time.sleep`），必须用`await`调用异步版本（如`asyncio.sleep`）。

Pydantic 数据校验

FastAPI 依赖 Pydantic 库实现请求数据的自动校验和类型转换，确保输入数据符合预期。

- **示例：请求体校验**

  ```python
  from fastapi import FastAPI
  from pydantic import BaseModel, EmailStr, Field
  
  app = FastAPI()
  
  # 定义数据模型（继承BaseModel）
  class User(BaseModel):
      name: str = Field(..., min_length=2, max_length=50)  # 必传，字符串长度限制
      age: int = Field(None, ge=0, le=120)  # 可选，整数范围0-120
      email: EmailStr  # 自动验证邮箱格式
  
  # 接收并校验请求体
  @app.post("/users/")
  async def create_user(user: User):
      return {"user": user.dict(), "message": "用户创建成功"}
  ```

- **校验效果**：

  - 若请求体不符合模型（如`age=-5`、`email="invalid"`），FastAPI 会自动返回 422 错误，包含详细校验信息；
  - 支持自动类型转换（如字符串`"25"`转为整数`25`）；
  - 可通过`Field`定义额外约束（如默认值、描述、正则匹配）。

- **适用场景**：

  - 请求参数（路径参数、查询参数、请求体）校验；
  - 响应数据格式化（确保输出结构一致）；
  - 数据模型文档自动生成（FastAPI 的 Swagger 文档会基于 Pydantic 模型展示参数说明）。

## AI 与 Agent

看以前文章