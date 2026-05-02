# Secure Boot 自定义密钥管理工具

## 介绍

这是一套专为 Linux 用户设计的 bash 脚本工具，用于 **创建和管理自定义 Secure Boot 密钥**。

**主要功能**：

- 生成一套完整的自定义 Secure Boot 密钥（PK、KEK、db 和 MOK）
- 实现完全独立于微软密钥的 Secure Boot 环境
- 安全地签名自己的 bootloader（shim/grub）、内核和内核模块

## 前置要求

**1. 系统要求**

- 支持 Secure Boot 的 UEFI 主板
- 系统必须处于 **Setup Mode**（见下方检查命令）
- 本文档中的大部分脚本和命令都需要使用 root 权限执行

**2. 所需软件包**

**Debian / Ubuntu / Linux Mint：**

```bash
root@localhost:~# apt update
root@localhost:~# apt install mokutil sbsigntool efitools openssl uuid-runtime
```

**Fedora / RHEL：**

```bash
root@localhost:~# dnf install mokutil sbsigntool efitools openssl util-linux
```

**Arch Linux：**

```bash
root@localhost:~# pacman -S mokutil sbsigntool efitools openssl
```

**3. 检查是否处于 Setup Mode**

在运行任何脚本之前，请先确认 Secure Boot 状态：

```bash
root@localhost:~# mokutil --sb-state
```

**预期输出**：

```
SecureBoot disabled
Platform is in Setup Mode
```

如果不是 **Setup Mode**，请进入 BIOS/UEFI 设置中清除 Secure Boot 密钥。

---

## 生成密钥

```bash
root@localhost:~# ./1.gen_keys
```

**脚本功能**：

- 生成 PK、KEK、db 和 MOK 密钥及证书
- 创建 EFI 签名列表（.esl）和已签名的 .auth 文件
- 创建 `secureboot/` 目录并进行完整性校验

运行时会提示输入你的名字（用于证书标识）。

**生成后的重要目录**：

- `secureboot/cert/` —— 存放私钥（`.key`）和证书（`.crt`）
- `secureboot/esl/` —— 存放efi签名列表（`.esl`）
- `secureboot/keystore/` —— 存放用于 UEFI 导入的 `.auth` 文件

> **提示**：请备份整个 `secureboot/` 文件夹，尤其是私钥文件。

---

## 文件签名

生成密钥后，需要对以下关键文件进行签名：

### 1. 使用 **db** 密钥签名 **shim**

```bash
root@localhost:~# sbsign --key secureboot/cert/db.key \
            --cert secureboot/cert/db.crt \
            --output 输出文件 \
            输入文件
```

### 2. 使用 **MOK** 密钥签名以下文件：

- **grub**
- **Linux 内核**（`vmlinuz`）
- **内核模块**（自行编译的驱动模块）

```bash
root@localhost:~# sbsign --key secureboot/cert/MOK.key \
            --cert secureboot/cert/MOK.crt \
            --output 输出文件 \
            输入文件
```

---

## 导入 MOK

```bash
root@localhost:~# ./2.import_mok
```

### 为什么需要先导入 MOK？

MOK（**Machine Owner Key**）是 **shim** 额外信任的密钥数据库。  
即使 db 密钥已导入 UEFI，shim 仍然需要通过 MOK 来信任你签名的 grub 和内核。因此必须先导入 MOK，才能让自定义签名的系统正常启动。

### 导入流程

1. 运行脚本并 **设置 MOK 密码**（请牢记）。
2. 脚本会自动使用 `mokutil` 导入 `MOK.der`。
3. **立即重启**电脑。
4. 重启后会进入蓝色 **MOK Management** 界面：
   - 选择 **Enroll MOK**
   - 选择 **Continue**
   - 输入刚才设置的密码
   - 确认 **Yes** 进行导入
   - 再次重启

导入完成后，MOK 将被固件永久信任。

---

## 导入 UEFI 证书

```bash
root@localhost:~# ./3.import_uefi_cert
```

此步骤会将你的自定义 **PK/KEK/db** 密钥写入 UEFI 固件。

导入完成后，执行以下命令验证密钥是否成功加载：

```bash
root@localhost:~# mokutil -la
```

你应该能看到自己的 PK、KEK 和 db 证书。

重新启动后，再次检查 Secure Boot 状态：

```bash
root@localhost:~# mokutil --sb-state
```

**预期输出**：

```
SecureBoot enabled
```

---

## **祝你自定义 Secure Boot 设置成功！** 🔐
