# sing-box 3in1（hy2/tuic/Anytls）管理脚本

[![Shell](https://img.shields.io/badge/Shell-Bash-4EAA25?logo=gnu-bash&logoColor=white)](https://www.gnu.org/software/bash/)[![Platform](https://img.shields.io/badge/Platform-Linux-blue)](https://www.kernel.org/)[![License](https://img.shields.io/badge/License-MIT-lightgrey)](./LICENSE)

一个用于快速部署和管理 **sing-box** 三合一服务端的脚本，集成：

- **Hysteria2**
- **TUIC**
- **AnyTLS**

适用于 VPS、家宽、NAT 机器、单栈 / 双栈环境。

---

## 目录

- [功能特性](#功能特性)
- [支持环境](#支持环境)
- [快速开始](#快速开始)
- [菜单功能](#菜单功能)
- [配置说明](#配置说明)
- [环境变量](#环境变量)
- [节点输出](#节点输出)
- [常见问题](#常见问题)
- [卸载](#卸载)
- [免责声明](#免责声明)

---

## 功能特性

- 一键安装 `sing-box`
- 自动生成证书
- 自动生成三协议配置
- 自动检测公网 IPv4 / IPv6
- 支持单栈 / 双栈 / NAT 环境
- 自动生成节点链接
- 支持菜单管理
  - 重装
  - 卸载
  - 重置端口
  - 重启服务
  - 查看节点
- 支持 `systemd` / `OpenRC`
- 支持常见 Linux 发行版
- 自动处理基础依赖安装
- 可自定义限速、端口、SNI 等参数

---

## 支持环境

### 操作系统

- Alpine Linux 3.x
- Debian 11+
- Ubuntu 20.x+
- CentOS / Rocky / AlmaLinux
- 其他支持 `systemd` 或 `OpenRC` 的 Linux

### 架构

- `amd64` / `x86_64`
- `arm64` / `aarch64`
- `armv7`

### 网络环境

- IPv4 only
- IPv6 only
- 双栈
- NAT 出口环境

---

## 快速开始

### 1. 下载脚本

```bash
bash <(curl -fsSL https://raw.githubusercontent.com/hakd-code/sbox-easy/main/sbox.sh)
```

或者直接上传脚本到服务器后执行：

```bash
# 若vps遥服务器没有安装bash请先安装它
apk add bash
chmod +x sbox.sh
bash sbox.sh
```

### 2. 首次运行

脚本会自动完成以下操作：

1. 检查并安装依赖
2. 下载并安装 `sing-box`
3. 生成证书
4. 生成配置文件
5. 启动服务
6. 输出节点信息

------

## 菜单功能

脚本运行后会进入管理菜单：

```reStructuredText
===== sing-box 3in1（hy2/tuic/Anytls）管理脚本 =====
1) 重装
2) 卸载
3) 重置端口
4) 重启服务
5) 查看节点
0) 退出
```

### 1. 重装

重新安装整套环境，包括：

- `sing-box`
- 配置文件
- 服务文件
- 证书
- 节点信息

### 2. 卸载

删除：

- `sing-box` 程序
- 配置文件
- 证书文件
- 服务文件

### 3. 重置端口

重新生成各协议端口，并自动重启服务。

### 4. 重启服务

仅重启服务，不修改配置。

### 5. 查看节点

重新检测公网 IPv4 / IPv6，并输出可用节点。

------

## 配置说明

脚本默认会生成以下三种入站：

- **Hysteria2**
- **TUIC**
- **AnyTLS**

默认使用同一套证书，并自动生成对应所需参数。

### 默认路径

```reStructuredText
/usr/local/bin/sing-box
/etc/sing-box/config.json
/etc/sing-box/state.env
/etc/sing-box/cert.pem
/etc/sing-box/key.pem
/etc/sing-box/info.log
```

------

## 环境变量

你可以在运行脚本前自定义部分参数：

### 自定义 SNI / 证书域名

```bash
export SNI_HOST=www.bing.com
```

### 自定义 Hysteria2 限速

```Bash
export HY2_UP_MBPS=100
export HY2_DOWN_MBPS=100
```

### 自定义端口范围

如果脚本支持端口范围，也可以预先设置，例如：

```Bash
export PORT_MIN=20000
export PORT_MAX=60000
```

> 具体支持哪些变量，以脚本实现为准。

------

## 节点输出

安装完成后，脚本会根据当前网络环境输出节点信息，包括：

- `HY2_V4`
- `HY2_V6`
- `TUIC_V4`
- `TUIC_V6`
- `AnyTLS_V4`
- `AnyTLS_V6`

如果机器只有单栈，脚本只会输出对应栈的节点。

------

## 常见问题

### 1. 为什么没有输出 IPv6 节点？

可能原因：

- 服务器本身没有可用 IPv6 出口
- 云厂商未分配 IPv6
- 防火墙或安全组未放行对应端口
- IPv6 探测源访问失败

------

### 2. 为什么客户端无法连接？

请检查：

- 云服务器安全组是否放行端口
- 本机防火墙是否放行端口
- 节点地址是否正确
- 客户端是否开启 `allowInsecure` / `insecure`
- SNI 是否与证书一致

------

### 3. NAT 机器可以用吗？

可以。
 脚本适用于只要能访问外网的 NAT 环境，并会尝试自动探测公网 IPv4 / IPv6。

------

### 4. 如何查看服务状态？

#### systemd

```bash
systemctl status sing-box
journalctl -u sing-box -e --no-pager
```

#### OpenRC

```bash
rc-service sing-box status
```

------

## 卸载

你可以在菜单中选择：

```reStructuredText
2) 卸载
```

也可以手动删除：

```bash
rm -f /usr/local/bin/sing-box
rm -rf /etc/sing-box
rm -f /etc/systemd/system/sing-box.service
rm -f /etc/init.d/sing-box
systemctl daemon-reload 2>/dev/null || true
```

------

## 项目结构

如果你的仓库中包含多个文件，可以参考类似结构：

```reStructuredText
.
├── README.md
├── sbox.sh
└── LICENSE
```

------

## 开发与贡献

欢迎提交 Issue 和 Pull Request：

- 功能建议
- Bug 修复
- 文档优化
- 多平台兼容性改进

如果你希望贡献代码，请尽量保持：

- Bash 兼容性
- 变量命名清晰
- 函数职责单一
- 日志输出简洁明确

------

## 免责声明

本项目仅供学习、测试和合法用途使用。
 使用者需确保自己的部署行为符合当地法律法规以及网络服务条款。

作者不对因使用本脚本造成的任何后果负责。



## License

MIT License
