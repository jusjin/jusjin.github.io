---
layout: post
title: Windows/WSL2 上部署 OpenClaw 完整指南
date: 2026-05-18 20:45 +0800
category: [AI, OpenClaw]
tags: [Windows, WSL2, Docker, OpenClaw]
---

## 简介

本文档将带你一步步在 WSL2（Windows Subsystem for Linux 2）上安装和配置 OpenClaw AI 智能体。

### 为什么选择 WSL2 运行 OpenClaw

OpenClaw 是使用 Node.js 构建的应用，专门在 Linux 环境下开发和测试。在 WSL2 中运行它意味着：

- ✅ 与 Linux 服务器完全相同的运行环境
- ✅ 无需兼容性修补或 Windows 路径特殊处理
- ✅ 原生模块完整支持

### 不同部署方案对比

| 方案                            | 兼容性            | 难度 |
| ------------------------------- | ----------------- | ---- |
| **WSL2 + npm**                  | 完整 Linux 兼容性 | 中等 |
| **Docker Desktop（WSL2 后端）** | 优秀，容器化      | 简单 |

## 部署步骤

### 步骤 1：安装 WSL2

1. 以**管理员身份**打开 PowerShell（右键开始菜单 → "Windows PowerShell (管理员)"）

2. 运行以下命令：

```bash
wsl --install
```

3. 提示时**重启计算机**

4. 重启后，Ubuntu 将自动启动并完成设置。按照提示创建 Linux 用户名和密码（这些与 Windows 凭据不同）

5. 验证 WSL2 是否安装成功：

```bash
wsl --list --verbose
```

你应该看到 `Ubuntu` 列出为 `VERSION 2`。如显示版本 1，升级它：

```bash
wsl --set-version Ubuntu 2
```

### 步骤 2：在 WSL2 中安装 Node.js

1. 从开始菜单打开 **Ubuntu**（或在 PowerShell 中运行 `wsl`）

2. 在 Ubuntu 终端中安装 Node.js 24：

```bash
curl -fsSL https://deb.nodesource.com/setup_24.x | sudo -E bash -
sudo apt-get install -y nodejs
```

3. 验证安装：

```bash
node --version
npm --version
```

你应看到 Node.js 24.x 和 npm 10.x（或更高版本）。OpenClaw 需要 Node.js 22 或更高版本。

### 步骤 3：通过 npm 安装 OpenClaw

在同一体 Ubuntu 终端中，全局安装 OpenClaw：

```bash
npm install -g openclaw
```

这将在系统范围内安装 `openclaw` 命令。

验证安装：

```bash
openclaw --version
```

如果看到版本号，安装成功。如果找不到 `openclaw`，可能需要将 npm 全局 bin 目录添加到 PATH：

```bash
echo 'export PATH="$PATH:$(npm root -g)/../bin"' >> ~/.bashrc
source ~/.bashrc
```

### 步骤 4：配置 OpenClaw

1. 在 WSL2 文件系统（而非 `/mnt/c/`）中创建 OpenClaw 配置目录：

```bash
mkdir -p ~/.openclaw
```

**重要**：保持配置在 Linux 文件系统中以获得最佳性能！

1. 创建配置文件。以下示例使用 OpenRouter 作为 AI 模型提供商：

```bash
cat > ~/.openclaw/openclaw.json << EOF
{
  "models": {
    "providers": {
      "openrouter": {
        "apiKey": "YOUR_OPENROUTER_API_KEY"
      }
    },
    "default": "openrouter/anthropic/claude-sonnet-4-5"
  },
  "gateway": {
    "auth": {
      "token": "YOUR_GATEWAY_TOKEN"
    }
  }
}
EOF
```

### 步骤 5：启动网关并连接消息平台

1. 启动 OpenClaw 网关：

```bash
openclaw gateway
```

2. 默认在端口 **18789** 启动。在 Windows 浏览器中打开：

```
http://localhost:18789
```

## Docker Desktop 与 WSL2 集成

如果你更喜欢以 Docker 容器而不是 npm 的方式运行 OpenClaw，Docker Desktop 直接与 WSL2 集成：

1. 从 [docker.com](https://www.docker.com/products/docker-desktop/) 安装 Docker Desktop
2. 在安装期间启用 **WSL2 后端** 选项
3. 在 Docker Desktop 设置中，转到 **资源 → WSL 集成**，为你的 Ubuntu 发行版启用集成
4. 现在可以直接在 Ubuntu WSL2 终端中使用 `docker` 命令

5. 拉取并运行 OpenClaw：

```bash
docker run -d --name openclaw \
  -v ~/.openclaw:/home/node/.openclaw \
  -p 18789:18789 \
  ghcr.io/openclaw/openclaw:latest
```

6. Docker 容器也可在 Windows 浏览器中的 `http://localhost:18789` 访问

### 设置内存限制

默认情况下，WSL2 最多可以使用总 RAM 的 50% 或 8 GB（以较小者为准）。要设置更低的限制，创建或编辑 `C:\Users\<YourName>\.wslconfig`：

```ini
[wsl2]
memory=4GB
processors=2
```

保存后重启 WSL2：

```bash
wsl --shutdown
```

然后在 PowerShell 中重新打开 Ubuntu。

### 启用 systemd

较新版本的 Ubuntu on WSL2 支持 systemd，可以以后台服务方式运行 OpenClaw。在 WSL2 中的 `/etc/wsl.conf` 中添加：

```ini
[boot]
systemd=true
```

然后重启 WSL2（`wsl --shutdown`）并创建 systemd 服务：

```bash
sudo nano /etc/systemd/system/openclaw.service
```

内容：

```ini
[Unit]
Description=OpenClaw Gateway
After=network.target

[Service]
Type=simple
User=YOUR_WSL_USERNAME
ExecStart=/usr/bin/openclaw gateway
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

启用服务：

```bash
sudo systemctl enable --now openclaw
```

## 常见 WSL2 问题

### npm 安装失败

使用 nvm（Node 版本管理器）进行用户拥有的 Node.js 安装：

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash
source ~/.bashrc
nvm install 24
nvm use 24
npm install -g openclaw
```

### WSL2 会话中途关闭

默认情况下，Windows 11 在无活动（没有打开的终端窗口，没有运行的进程）后会自动关闭 WSL2。要保持 OpenClaw 持续运行，请：

- 使用上面的 systemd 方法
- 或通过 `.wslconfig` 防止空闲关闭：

```ini
[wsl2]
# 即使没有打开终端窗口也保持 WSL2 运行
kernelCommandLine = quiet
```

## 总结

在 WSL2 上部署 OpenClaw 让你获得：

- ✅ 接近原生的 Linux 性能
- ✅ 完整的 Node.js 和 npm 支持
- ✅ 便捷的 Windows 浏览器访问（自动端口转发）
- ✅ 灵活的选择（npm 或直接 Docker）
- ✅ 强大的调试和问题解决能力

记住，如果以后遇到任何网络或性能问题，检查文件是否仍在 Linux 文件系统，并确保虚拟化和网络连接配置正确。

🎉 祝你在 WSL2 上成功运行 OpenClaw！
