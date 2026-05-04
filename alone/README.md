# alone单独管理脚本

[![Shell](https://img.shields.io/badge/Shell-Bash-4EAA25?logo=gnu-bash&logoColor=white)](https://www.gnu.org/software/bash/)[![Platform](https://img.shields.io/badge/Platform-Linux-blue)](https://www.kernel.org/)[![License](https://img.shields.io/badge/License-MIT-lightgrey)](./LICENSE)

本单独管理脚本适用于当 **sing-box** 三合一服务端的脚本安装出错时，或系统的RAM或DISK不足出错时的替代选择。可单独选择安装的脚本包含以下内容：

- **Hysteria2管理脚本**
- **TUIC管理脚本**
- **AnyTLS管理脚本**

适用于 VPS、家宽、NAT 机器、单栈 / 双栈环境，特别适用于内存和磁盘特别小的机器例如1c/32M/50M的机器。

---

## 目录

- [支持环境](#支持环境)
- [快速开始](#快速开始)
- [开发与贡献](#开发与贡献)
- [免责声明](#免责声明)

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

### 1. 总体说明

- **Hysteria2管理脚本**
- **TUIC管理脚本**
- **AnyTLS管理脚本**

这三种单独的脚本对于机器消耗的资源不同，具体请看如下表格：

| 脚本名称  | 资源占用 | 主要安装路径        |
| --------- | -------- | ------------------- |
| Hysteria2 | 约19.8M  | /usr/local/hysteria |
| TUIC      | 约7.9M   | /usr/local/tuic     |
| AnyTLS    | 约5.2M   | /usr/local/anytls   |

请依据自身系统的资源情况做出合理的选择。



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

### 2. 快速开始

**Hysteria2管理脚本**

```bash
bash <(curl -fsSL https://raw.githubusercontent.com/hakd-code/sbox-easy/main/alone/hy2.sh)
```



**TUIC管理脚本**

```bash
bash <(curl -fsSL https://raw.githubusercontent.com/hakd-code/sbox-easy/main/alone/tuic.sh)
```



**AnyTLS管理脚本**

```bash
bash <(curl -fsSL https://raw.githubusercontent.com/hakd-code/sbox-easy/main/alone/anytls.sh)
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
