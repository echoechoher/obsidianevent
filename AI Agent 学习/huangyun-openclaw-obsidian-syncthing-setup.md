# OpenClaw + Obsidian 打造个人数字资产生产线

> **来源**：X @huangyun_122（黄赟）  
> **发布时间**：2026-03-01  
> **原文链接**：https://x.com/huangyun_122/status/2027802599836332264  
> **整理时间**：2026-03-03

---

## 核心理念

我们每天和大模型对话产生大量有价值的思维发散，但关掉对话窗口那瞬间，这些就都消失了。

**解决方案：** 把对话落到 Obsidian 形成 RAW 文档，及时安排 Agent 复盘，发现"不知道自己不知道"的知识盲区。

> 一旦开启复盘模式，某个下午我将收到：Hey, 你上周爆款迁移这个想法不错，其实宝玉老师有个公众号 Skill 可以帮你自动发文哦，要不要我帮你试试……

---

## 系统架构

```
远程 VPS（OpenClaw）
    ↓ Syncthing 实时同步
本地 Mac（Obsidian 编辑）
    ↑ TailScale 加密（可选）
```

**所需软件：**
- VPS 端：OpenClaw、Syncthing、TailScale（可选）
- 本地 Mac：Obsidian、VS Code + Remote-SSH、Syncthing、TailScale（可选）、Telegram

---

## 部署环境

| 项目 | 配置 |
|------|------|
| VPS 系统 | OpenCloudOS（RHEL/CentOS 系），腾讯海外服务器 199元/年 |
| VPS 登录 | SSH 密钥登录 |
| 本地系统 | macOS |
| 本地已安装 | Obsidian、VS Code + Remote SSH |

---

## 第一步：VPS 上安装 Syncthing

```bash
# 下载最新版（Linux amd64）
wget https://github.com/syncthing/syncthing/releases/download/v2.0.14/syncthing-linux-amd64-v2.0.14.tar.gz

# 解压
tar -xzf syncthing-linux-amd64-v2.0.14.tar.gz

# 复制到系统路径
cp syncthing-linux-amd64-v2.0.14/syncthing /usr/local/bin/
chmod +x /usr/local/bin/syncthing

# 验证
/usr/local/bin/syncthing --version
```

---

## 第二步：创建 systemd 服务文件

> ⚠️ 二进制安装不会自动创建服务文件，必须手动创建。

```bash
nano /etc/systemd/system/syncthing@.service
```

粘贴以下内容：

```ini
[Unit]
Description=Syncthing - Open Source Continuous File Synchronization for %I
Documentation=man:syncthing(1)
After=network.target

[Service]
User=%i
ExecStart=/usr/local/bin/syncthing serve --no-browser --no-restart --logflags=0
Restart=on-failure
RestartSec=5
SuccessExitStatus=3 4
RestartForceExitStatus=3 4

[Install]
WantedBy=multi-user.target
```

| 参数 | 含义 |
|------|------|
| `--no-browser` | 不自动打开浏览器（服务器无桌面，必须加） |
| `--no-restart` | 不自动重启，交给 systemd 管理 |
| `--logflags=0` | 简化日志格式 |

---

## 第三步：启动并验证服务

```bash
systemctl daemon-reload
systemctl enable syncthing@root.service
systemctl start syncthing@root.service
systemctl status syncthing@root.service
```

看到 `Active: active (running)` 即成功。

---

## 第四步：Mac 上安装 Syncthing

```bash
brew install syncthing
brew services start syncthing
```

Mac 端管理界面：http://localhost:8384

---

## 第五步：SSH 隧道访问 VPS 管理界面

VPS 的 Syncthing 默认只监听 127.0.0.1:8384，需要 SSH 隧道转发：

```bash
# 本地 Mac 执行（替换为你的 VPS 信息）
ssh -L 9384:127.0.0.1:8384 your_name@your_vps_ip -N
```

然后浏览器打开 http://localhost:9384 即可访问 VPS 端管理界面。

> `-N` 参数：只建立隧道，不执行命令，保持终端开着即可。

---

## 第六步：两端互相添加设备

**获取设备 ID：**
- Mac 端：http://localhost:8384 → 右上角操作 → 显示ID
- VPS 端：http://localhost:9384 → 同样操作

**Mac 端添加 VPS：** 点击右下角"添加远程设备"，粘贴 VPS 设备 ID。

**VPS 端确认连接请求：** 在管理界面确认 Mac 发来的请求。

---

## 第七步：配置同步文件夹

**VPS 端：** 添加文件夹 → 填写 vault 路径 → 共享给 Mac 设备

**Mac 端：** 接受共享请求 → 本地路径填 `/Users/your_username/obsidian-vault`

---

## 第八步：Obsidian 打开同步目录

Obsidian → 打开另一个库 → 打开本地文件夹 → 选择 `obsidian-vault`

**可选：创建 `.stignore` 忽略文件**

```
// Obsidian 工作区状态，不同设备各自维护
.obsidian/workspace.json
.obsidian/workspace-mobile.json
.DS_Store
```

---

## 可选：TailScale 加固安全

**不加固的安全隐患：**
- Syncthing 22000 端口暴露在公网
- 流量可能经过第三方中继服务器
- 设备 IP 上报给全球发现服务器

**加固后架构：**
```
# 加固前
Mac ──── 公网 ──── VPS:22000（端口暴露）

# 加固后
Mac ──── Tailscale 私有网络（WireGuard 加密）──── VPS
         100.x.x.x 私有 IP，公网完全不可见
```

### VPS 安装 Tailscale

```bash
curl -fsSL https://tailscale.com/install.sh | sh
systemctl enable --now tailscaled
tailscale up  # 输出授权 URL，浏览器打开登录
tailscale ip -4  # 查看分配的 100.x.x.x 地址
```

### Mac 安装 Tailscale

```bash
brew install --cask tailscale
```

> ⚠️ Mac 必须安装 GUI 版本，不能只装 CLI 版。安装后在系统设置中允许系统扩展和 VPN 配置。

### 配置 Syncthing 只走 Tailscale

**VPS 端（http://localhost:9384）：**
- 设置 → 连接 → 同步协议监听地址改为：`tcp://100.x.x.x:22000`
- 取消勾选"全球发现"和"启用中继"

**Mac 端：**
- 编辑 VPS 设备 → 地址改为：`tcp://100.x.x.x:22000`
- 同样关闭全球发现和中继

**关闭公网端口：**
```bash
firewall-cmd --permanent --remove-port=22000/tcp
firewall-cmd --reload
```

**可选：直接用 Tailscale IP 访问 VPS 管理界面（无需 SSH 隧道）：**
VPS 端 Syncthing → 设置 → GUI → 监听地址改为 `100.x.x.x:8384`，之后浏览器直接访问。
