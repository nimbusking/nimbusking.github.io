---
title: vmware安装教程
abbrlink: 65cd2699
date: 2026-06-15 11:34:45
updated: 2026-06-15 11:34:45
tags:
  - vmware
  - 安装教程
  - 虚拟化
categories: 工具
---

# VMware Workstation Pro 26H1 下载、安装与创建虚拟机完整教程

> **最后更新：2026 年 6 月 15 日**
>
> 适用版本：VMware Workstation Pro **26H1** (Build 25388281)，发布于 2026 年 5 月 14 日


<!-- more -->


---

## 目录

1. [前言](#1-前言)
2. [26H1 新特性一览](#2-26h1-新特性一览)
3. [系统要求](#3-系统要求)
4. [第一步：注册 Broadcom 账号](#4-第一步注册-broadcom-账号)
5. [第二步：下载 VMware Workstation Pro 26H1](#5-第二步下载-vmware-workstation-pro-26h1)
6. [第三步：安装前的系统准备](#6-第三步安装前的系统准备)
7. [第四步：安装 VMware Workstation Pro](#7-第四步安装-vmware-workstation-pro)
8. [第五步：创建你的第一台虚拟机](#8-第五步创建你的第一台虚拟机)
9. [第六步：安装客户操作系统](#9-第六步安装客户操作系统)
10. [第七步：安装 VMware Tools](#10-第七步安装-vmware-tools)
11. [虚拟机管理基础](#11-虚拟机管理基础)
12. [常见问题与排查](#12-常见问题与排查)
13. [总结](#13-总结)

---

## 1. 前言

VMware Workstation Pro 是全球最流行的桌面虚拟化软件之一，它允许你在一台物理电脑上同时运行多个操作系统（如 Windows、Linux、FreeBSD 等）。

自 Broadcom（博通）收购 VMware 以来，**VMware Workstation Pro 已面向个人、教育和商业用户全面免费**，无需额外购买许可证即可享用全部功能。26H1 是该产品历史上一次重大的架构升级——Windows 版本全面转向 64 位架构，带来了显著的性能提升。

无论你是开发者需要在隔离环境中测试代码，还是运维人员想搭建实验环境学习 Linux，抑或是普通用户想安全地体验不同操作系统，VMware Workstation Pro 都是你的理想选择。

---

## 2. 26H1 新特性一览

### 🌟 核心变更：Windows 全面 64 位化

这是 26H1 最重大的变化。所有二进制文件、库、安装程序组件和服务进程现已全部采用 64 位架构：

- 安装路径从 `C:\Program Files (x86)\` 迁移至 `C:\Program Files\`
- 单台虚拟机最大内存支持提升至 **256 GB**
- 虚拟机启动速度提升约 **50%**
- 多虚拟机并行运行时的内存利用率显著优化

> ⚠️ **注意**：此版本彻底移除了 32 位支持，无法再运行 Windows XP、Windows 7 32 位、CentOS 6 等 32 位操作系统。如有这类需求，请保留 17.x 版本。

### 🕐 虚拟机生命周期时间戳

每台虚拟机现在会记录并显示**创建时间**和**最后开机时间**，方便你追踪长期项目或识别不再需要的测试环境，这在管理大量虚拟机时格外实用。

### 📝 文件夹备注集成

虚拟机的备注信息直接显示在 Workstation 主界面的**文件夹标签页**中，无需进入设置菜单即可一目了然地查看每台虚拟机的用途说明。

### 🖥️ ARM ESXi 远程连接

新增对 **ARM 架构 ESXi 主机**的远程连接支持，可执行基本的虚拟机操作。不过此项功能目前仍处于**技术预览（Tech Preview）**阶段，不建议用于生产环境。

### 🔑 其他改进

- 凭据管理器格式优化：加密虚拟机和远程服务器的凭据在 Windows 凭据管理器中更易识别
- 帮助链接修复：应用内帮助按钮现在正确跳转至 Broadcom 技术文档页面
- 新增客户操作系统支持：Ubuntu 26.04 LTS、Fedora 43/44、SUSE Linux Enterprise 16、openSUSE 16.0、FreeBSD 15.0
- 新增主机操作系统支持：Ubuntu 26.04 LTS、Fedora 43/44、SUSE Linux Enterprise 16、openSUSE 16.0

---

## 3. 系统要求

### 硬件要求

| 组件 | 最低要求 | 推荐配置 |
|------|---------|----------|
| **CPU** | 2011 年后发布的 64 位 x86/AMD64 处理器，必须支持 **Intel VT-x** 或 **AMD-V** | 多核 CPU（Intel 第 13/14 代或 AMD Ryzen 7000/8000 系列） |
| **内存** | 最低 4 GB | **16 GB 及以上**（根据虚拟机数量与负载） |
| **硬盘** | 至少 2 GB 可用空间（不含虚拟机文件） | **SSD 固态硬盘**，留有充足的额外空间给虚拟机磁盘 |
| **显示器** | 1024×768 分辨率 | 1920×1080 或更高 |

### 主机操作系统要求

**Windows：**
- Windows 10（64 位，版本 21H2 或更高）
- Windows 11（64 位，版本 21H2 或更高）
- Windows Server 2019 / 2022 / 2025（64 位）

**Linux：**
- Ubuntu 22.04 / 24.04 / 26.04 LTS（64 位）
- Fedora 41 / 42 / 43 / 44
- Red Hat Enterprise Linux 9.x / 10.x
- SUSE Linux Enterprise 15 / 16
- openSUSE 15.x / 16.0
- Debian 12.x

### BIOS 设置确认

在开始安装之前，请确认以下设置已在主板的 BIOS/UEFI 中**启用**：

1. **Intel VT-x** 或 **AMD-V（SVM Mode）**——硬件虚拟化支持
2. **VT-d** 或 **AMD IOMMU**（可选，但建议启用）

> 💡 如何确认虚拟化是否已启用？
>
> 在 Windows 中打开**任务管理器**（`Ctrl + Shift + Esc`），切换到"**性能**"标签页，选中"**CPU**"，查看右下角的"**虚拟化**"状态。如果显示"已启用"，则说明 BIOS 设置正确。

---

## 4. 第一步：注册 Broadcom 账号

VMware Workstation Pro 现在通过 Broadcom 支持门户分发。下载前需要先注册一个免费的 Broadcom 账号。

### 4.1 访问注册页面

在浏览器中访问：

```
https://profile.broadcom.com/web/registration
```

### 4.2 填写注册信息

1. 在注册页面输入你的**邮箱地址**和图形验证码，点击 **"NEXT"**
2. 稍后你的邮箱会收到一封来自 Broadcom 的邮件，内含 **6 位验证码**
3. 回到注册页面输入验证码，点击 **"Verify & Continue"**
4. 填写基本信息：国家/地区、姓名、密码等
5. 勾选同意服务条款，点击 **"Create Account"**
6. 系统可能会弹出"完善个人资料"页面，可以点击 **"I'll do it later"** 跳过，不影响后续下载

> ✅ **支持的邮箱**：QQ 邮箱、Gmail、Outlook 均可正常注册。
>
> ⚠️ **注意**：部分国内邮箱（如 163.com、126.com）可能被限制，建议优先使用 QQ 邮箱或 Gmail。

---

## 5. 第二步：下载 VMware Workstation Pro 26H1

### 5.1 登录 Broadcom 支持门户

1. 访问 Broadcom 支持门户：
   ```
   https://support.broadcom.com/group/ecx/productdownloads
   ```
2. 点击右上角 **"LOGIN"** 按钮
3. 输入你的注册邮箱 → 点击 **"NEXT"**
4. 输入密码 → 点击 **"Sign In"**

### 5.2 进入免费下载页面

登录成功后，有两种方式找到下载入口：

**方式一（推荐快捷方式）：**
直接访问以下链接（需已登录）：
```
https://support.broadcom.com/group/ecx/productdownloads?subfamily=VMware%20Workstation%20Pro&freeDownloads=true
```

**方式二（标准路径）：**
1. 点击左侧菜单中的 **"My Downloads"**
2. 页面上方会显示提示：*"Free Software Downloads available HERE"*
3. 点击 **"HERE"** 链接跳转到免费产品页面

### 5.3 选择版本并下载

1. 在免费下载页面找到 **"VMware Workstation Pro"** 并点击
2. 选择最新版本 **"VMware Workstation Pro 26H1 for Windows"**（Windows 用户）或 **"for Linux"**（Linux 用户）
3. 选择具体小版本号（通常选择最新的 Build 即可）
4. 进入下载详情页后，**先点击蓝色的 "Terms and Conditions" 超链接**阅读条款
5. 阅读完毕后，勾选 **"I agree to the Terms and Conditions"** 复选框
6. 点击下载按钮（云朵图标 ☁️）
7. 首次下载会弹出基本信息收集页面（国家、城市、邮政编码等），如实填写后点击 **"Submit"**
8. 返回下载页面，再次点击下载按钮，浏览器将开始下载安装包

> 📦 **下载文件名示例**：
> - Windows：`VMware-Workstation-Full-26H1-25388281.exe`（约 600–700 MB）
> - Linux：`VMware-Workstation-Full-26H1-25388281.x86_64.bundle`（约 500–600 MB）

### 5.4 下载的时候提示错误
如果你下载的时候，提示：*Account verification is Pending. Please try after some time.*
用下面的油猴脚本，截止笔者测试日期（2026-06-15）目前能解决：
```javascript
// ==UserScript==
// @name         VMware Download Status Modifier
// @namespace    http://tampermonkey.net/
// @version      1.0
// @description  解决博通官网VMware下载的Account verification is Pending问题
// @author       自定义
// @match        https://support.broadcom.com/group/ecx/productfiles*
// @grant        none
// @run-at       document-start
// ==/UserScript==

(function() {
    'use strict';

    // 要拦截的目标请求核心路径
    const TARGET_REQUEST_URL = '/group/ecx/productfiles/-/productFiles/getDownloadableFiles';
    // 替换exportControlStatus字段值
    const REPLACE_FIELD = 'exportControlStatus';
    const OLD_VALUE = 'SCREENING_REQUIRED';
    const NEW_VALUE = 'SCREENING_NOT_REQUIRED';

    // 深度遍历替换字段值
    function deepReplaceValue(obj, targetKey, oldVal, newVal) {
        if (Array.isArray(obj)) {
            obj.forEach(item => deepReplaceValue(item, targetKey, oldVal, newVal));
        } else if (typeof obj === 'object' && obj !== null) {
            Object.entries(obj).forEach(([key, value]) => {
                if (key === targetKey && value === oldVal) {
                    obj[key] = newVal;
                    console.log(`[脚本] 替换成功：${key} = ${newVal}`);
                }
                deepReplaceValue(value, targetKey, oldVal, newVal);
            });
        }
    }

    // 重写XHR拦截请求
    const originalOpen = XMLHttpRequest.prototype.open;
    const originalSend = XMLHttpRequest.prototype.send;

    XMLHttpRequest.prototype.open = function(method, url) {
        this._requestMethod = method.toUpperCase();
        this._requestUrl = url;
        return originalOpen.apply(this, arguments);
    };

    XMLHttpRequest.prototype.send = function(body) {
        const xhr = this;
        const originalOnReadyStateChange = xhr.onreadystatechange;

        xhr.onreadystatechange = function() {
            if (
                xhr.readyState === 4 &&
                xhr.status === 200 &&
                xhr._requestMethod === 'POST' &&
                xhr._requestUrl.includes(TARGET_REQUEST_URL)
            ) {
                try {
                    const originalResponse = JSON.parse(xhr.responseText);
                    deepReplaceValue(originalResponse, REPLACE_FIELD, OLD_VALUE, NEW_VALUE);
                    Object.defineProperty(xhr, 'responseText', {
                        value: JSON.stringify(originalResponse),
                        writable: false,
                        configurable: true
                    });
                } catch (e) {
                    console.error('[脚本] 处理失败：', e);
                }
            }
            if (typeof originalOnReadyStateChange === 'function') {
                originalOnReadyStateChange.apply(xhr, arguments);
            }
        };

        return originalSend.call(xhr, body);
    };

    console.log('[VMware下载助手] 脚本已加载');
})();
```

---

## 6. 第三步：安装前的系统准备

### 6.1 关闭 Hyper-V（Windows 主机必做）

VMware Workstation 与 Windows 自带的 Hyper-V 存在冲突。如果 Hyper-V 处于启用状态，VMware 虚拟机在启动时将会报错或运行极度缓慢。建议在安装前将其关闭。

#### 方法一：通过 Windows 功能关闭（推荐）

1. 按 `Win + R`，输入 `optionalfeatures`，回车
2. 在弹出的 "Windows 功能" 窗口中，**取消勾选**以下项目：
   - **Hyper-V**
   - **虚拟机平台**
   - **Windows 沙盒**
   - **Windows 虚拟机监控程序平台**
3. 点击"确定"，等待系统完成更改
4. 根据提示**重启电脑**

#### 方法二：通过命令行关闭

以**管理员身份**打开 PowerShell 或命令提示符，执行：

```powershell
# 关闭 Hyper-V
dism.exe /Online /Disable-Feature:Microsoft-Hyper-V

# 关闭虚拟机平台
dism.exe /Online /Disable-Feature:VirtualMachinePlatform
```

完成后重启电脑。

### 6.2 关闭 Device Guard / Credential Guard（如适用）

如果你的系统启用了 Device Guard 或 Credential Guard，也需要将其关闭。Microsoft 提供了专门的硬件准备工具：

1. 下载 [Device Guard and Credential Guard Hardware Readiness Tool](https://www.microsoft.com/en-us/download/details.aspx?id=53337)
2. 解压下载的 ZIP 文件
3. 以**管理员身份**打开 PowerShell，进入解压目录，执行：
   ```powershell
   Set-ExecutionPolicy Unrestricted -Scope Process
   DG_Readiness_Tool_v3.6.ps1 -Disable
   ```
4. **重启电脑**，启动时按 **F3** 确认禁用更改

### 6.3 关于 Windows 安全功能说明

你可能听说过 VMware Workstation 可以配合 Windows 的 **VBS（基于虚拟化的安全）** 使用。自 Workstation Pro 17.5 版本后，VMware 引入了对 **Windows Hypervisor Platform (WHP)** 的支持，使得在一定程度上可以与 Hyper-V 共存。

但在实际使用中，**关闭 Hyper-V 仍然能获得最佳的性能和兼容性**。如果你是初学者，建议先按上述步骤关闭 Hyper-V。如果你确实需要同时运行 WSL2、Docker Desktop 等依赖 Hyper-V 的应用，可以安装完成后再尝试开启并观察兼容性。

### 6.4 卸载旧版本（如适用）

如果你之前安装过旧版 VMware Workstation Pro（17.x 系列），建议先卸载：

1. 打开"控制面板" → "程序和功能"
2. 找到 **VMware Workstation**，右键选择"卸载"
3. 在卸载向导中，选择 **"保留配置和虚拟机信息"**（重要！这样你的虚拟机数据不会丢失）
4. 完成卸载后，**重启电脑**，再安装新版本

> ⚠️ 26H1 的安装路径从 `C:\Program Files (x86)\VMware\` 变更为 `C:\Program Files\VMware\`。旧安装目录中的部分 ISO 镜像文件不会自动删除，你可以在确认新版本工作正常后手动清理旧的程序文件夹。

---

## 7. 第四步：安装 VMware Workstation Pro

### 7.1 启动安装程序

双击下载的 `VMware-Workstation-Full-26H1-25388281.exe` 文件，安装向导将自动启动。如果出现用户账户控制（UAC）提示，点击"是"允许运行。

### 7.2 安装向导步骤

**步骤 1 —— 欢迎页面**
点击 **"Next >"**

**步骤 2 —— 最终用户许可协议**
- 勾选 **"I accept the terms in the License Agreement"**
- 点击 **"Next >"**

**步骤 3 —— 选择安装目录**
- 默认路径：`C:\Program Files\VMware\VMware Workstation\`
- 如果你希望安装到其他磁盘，点击 **"Change..."** 选择自定义路径
- 点击 **"Next >"**

**步骤 4 —— 用户体验设置（可选）**
- ✅ **Check for product updates on startup**（启动时检查产品更新）：建议保持勾选
- ☐ **Join the VMware Customer Experience Improvement Program**（加入客户体验改进计划）：根据个人意愿选择
- 点击 **"Next >"**

**步骤 5 —— 快捷方式**
- ✅ **Desktop**（桌面快捷方式）：建议勾选
- ✅ **Start Menu Programs folder**（开始菜单程序文件夹）：建议勾选
- 点击 **"Next >"**

**步骤 6 —— 准备安装**
确认所有设置无误后，点击 **"Install"**，安装程序将开始复制文件并进行配置。整个过程大约需要 2–5 分钟。

**步骤 7 —— 完成安装**
- 安装完成后点击 **"Finish"**
- 如果安装程序提示重启，选择 **"Yes"** 重启电脑

### 7.3 首次启动验证

1. 双击桌面上的 **VMware Workstation Pro** 图标启动程序
2. 首次启动会显示许可协议确认页面，选择 **"I accept the terms..."** 并点击 **"OK"**
3. 确认主界面正常加载，可通过菜单 **Help → About VMware Workstation Pro** 查看版本信息

---

## 8. 第五步：创建你的第一台虚拟机

下面我们将详细介绍如何创建一台 Windows 11 虚拟机。其他操作系统的流程大同小异，主要区别在于选择合适的客户操作系统类型和硬件配置。

### 8.1 准备操作系统镜像（ISO 文件）

首先需要获取你要安装的操作系统的 ISO 镜像文件：

| 操作系统 | 官方下载地址 |
|----------|-------------|
| **Windows 11** | [https://www.microsoft.com/software-download/windows11](https://www.microsoft.com/software-download/windows11) |
| **Windows 10** | [https://www.microsoft.com/software-download/windows10](https://www.microsoft.com/software-download/windows10) |
| **Ubuntu 26.04 LTS** | [https://ubuntu.com/download/desktop](https://ubuntu.com/download/desktop) |
| **Ubuntu 24.04 LTS** | [https://releases.ubuntu.com/24.04/](https://releases.ubuntu.com/24.04/) |
| **Fedora** | [https://fedoraproject.org/workstation/download](https://fedoraproject.org/workstation/download) |
| **Debian** | [https://www.debian.org/download](https://www.debian.org/download) |

> 📌 **提示**：建议将下载的 ISO 文件统一存放在一个易于查找的文件夹中，例如 `D:\ISO\`，方便管理。

### 8.2 启动新建虚拟机向导

1. 打开 **VMware Workstation Pro**
2. 在主页上点击 **"Create a New Virtual Machine"**（创建新的虚拟机）
   - 或者通过菜单：**File → New Virtual Machine...**
   - 快捷键：`Ctrl + N`

### 8.3 选择配置类型

向导会询问你选择虚拟机配置方式：

| 选项 | 说明 | 适合人群 |
|------|------|---------|
| **Typical (recommended)** | 典型（推荐），向导帮你完成大部分配置 | 初学者，绝大多数场景 |
| **Custom (advanced)** | 自定义（高级），完全控制所有硬件参数 | 有经验的用户，需要精细控制 |

> 本节以 **Typical（典型）** 模式为例进行讲解。

### 8.4 选择安装来源

1. 选择 **"Installer disc image file (iso)"**（安装程序光盘映像文件）
2. 点击 **"Browse..."** 浏览并找到你下载的 ISO 文件（如 `Win11_24H2_Chinese_Simplified_x64.iso`）
3. VMware 通常会自动检测操作系统类型和版本。如果未能识别，手动从下拉菜单选择：
   - 客户操作系统：**Microsoft Windows**
   - 版本：**Windows 11 x64** 或 **Windows 10 and later x64**

> 💡 VMware 的 **Easy Install（简易安装）** 功能会在读取 ISO 后自动配置客户操作系统。如果你使用的是 Windows ISO，可以在此输入 Windows 产品密钥、用户名和密码，系统安装过程将自动完成这些步骤。你也可以暂时留空，在安装过程中手动输入。

### 8.5 命名虚拟机并选择存储位置

| 配置项 | 建议 |
|--------|------|
| **虚拟机名称** | 取一个有辨识度的名称，如 `Windows11-Lab` 或 `Ubuntu-26.04-Dev` |
| **存储位置** | 建议选择**非系统盘**且空间充足的磁盘，如 `D:\VirtualMachines\` |

> 📁 每个虚拟机都会生成多个文件（`.vmx` 配置文件、`.vmdk` 虚拟磁盘等），建议为每台虚拟机建立独立文件夹。

### 8.6 配置虚拟机加密（可选但推荐）

如果你计划安装 Windows 11，VMware 会提示需要**虚拟可信平台模块 (vTPM)**，这要求对虚拟机进行加密。对于其他不需要 TPM 的操作系统，可以跳过此步骤。

1. 在弹出的加密设置对话框中，设置加密密码
2. 勾选 **"Remember the password on this machine"**（在此计算机上记住密码）以省去每次启动时的输密码步骤
3. 建议将密码**保存在安全的地方**，遗忘密码可能导致虚拟机数据无法恢复

### 8.7 指定磁盘容量

| 配置项 | 推荐值 | 说明 |
|--------|--------|------|
| **最大磁盘大小** | 60–100 GB | 视用途而定，开发环境建议 80GB+ |
| **存储为单个文件** | ✅ 推荐 | 性能更好，便于管理和迁移 |
| **将虚拟磁盘拆分为多个文件** | ☐ 可选 | 便于跨 FAT32 文件系统移动，但略微影响性能 |

> 💡 VMware 默认使用**精简置备（Thin Provisioning）**模式，磁盘文件会随着虚拟机使用逐渐增长，不会立刻占用全部设定空间。你可以在高级选项中关闭"Allocate all disk space now"以获得更灵活的磁盘管理。

### 8.8 自定义硬件（重要步骤）

在最终确认页面，**务必**点击 **"Customize Hardware..."** 按钮调整硬件配置，默认参数通常需要根据实际需求优化：

#### 内存 (Memory)

| 用途 | 建议分配 |
|------|---------|
| 轻量级 Linux（无桌面） | 1–2 GB |
| 桌面 Linux（GNOME/KDE） | 4–8 GB |
| Windows 11（基本使用） | 4–6 GB |
| Windows 11（开发/编译） | 8–16 GB |
| 数据库/容器实验 | 8 GB+ |

> ⚠️ 分配的内存不要超过物理内存的一半，确保宿主机有足够资源保持流畅。

#### 处理器 (Processors)

| 配置 | 建议 |
|------|------|
| **处理器数量** | 1（一般情况不需要多处理器） |
| **每个处理器的内核数** | 2–4 个 |
| **虚拟化引擎** | 勾选"Virtualize Intel VT-x/EPT or AMD-V/RVI" |

> 💡 给虚拟机分配的核心数不宜超过物理 CPU 核心数的一半。

#### 其他硬件建议配置

| 硬件 | 推荐设置 | 说明 |
|------|---------|------|
| **硬盘 (Hard Disk)** | NVMe 控制器 | 比 SATA 更快（新建向导通常默认 NVMe） |
| **网络适配器** | NAT | 最通用，虚拟机通过宿主机网络上网 |
| **USB 控制器** | USB 3.1 | 如果宿主机支持 USB 3.x |
| **声卡** | 按需保留 | 不需要声音可移除以节省资源 |
| **打印机** | 移除 | 一般不需要，可移除 |
| **显示器** | 启用 3D 图形加速 | 提升桌面流畅度（占用额外显存） |
| **CD/DVD 驱动器** | 保持 ISO 挂载 | 安装系统后会自动断开 |

完成所有配置后，点击 **"Close"**，回到确认页面，检查一遍设置摘要，确认无误后点击 **"Finish"**。

---

## 9. 第六步：安装客户操作系统

### 9.1 启动虚拟机

在 VMware Workstation 主界面左侧的虚拟机列表中，选中你刚创建的虚拟机，点击工具栏上的绿色 ▶ **"Power on this virtual machine"** 按钮（或点击虚拟机预览窗口中的"开启虚拟机"）。

### 9.2 从 ISO 镜像引导

虚拟机启动后，会出现类似物理机开机时的画面。注意在启动阶段观察屏幕提示：

- 当看到 **"Press any key to boot from CD or DVD..."** 提示时，**将鼠标点击到虚拟机屏幕内**（让虚拟机捕获光标），然后快速按下键盘上的**空格键**或任意键
- 如果没有及时按下，虚拟机可能跳过 ISO 启动，此时点击菜单 **VM → Power → Reset** 重启虚拟机即可

> 💡 **光标释放提示**：在虚拟机窗口中操作时，你的鼠标和键盘会被虚拟机"捕获"。按 `Ctrl + Alt` 组合键可以释放光标回到宿主机。

### 9.3 Windows 11 安装流程

以下以安装 Windows 11 为示例：

**第一步：语言和区域设置**

选择安装语言、时间格式和键盘布局，点击 **"下一步"**，然后点击 **"现在安装"**。

**第二步：输入产品密钥**

- 如果你有合法的 Windows 产品密钥，在此输入
- 如果没有，点击 **"我没有产品密钥 (I don't have a product key)"**，你仍然可以完成安装并在后续激活（未激活状态下仅部分个性化功能受限）

**第三步：选择版本**

选择你要安装的 Windows 11 版本：
- **Windows 11 Home**（家庭版）
- **Windows 11 Pro**（专业版）——推荐开发者选择
- **Windows 11 Education / Enterprise**（教育版/企业版）

勾选 **"我接受 Microsoft 软件许可条款"**，点击 **"下一步"**。

**第四步：选择安装类型**

⚠️ 选择 **"自定义：仅安装 Windows（高级）"**。

> 不要选择"升级"，因为虚拟机中不存在旧系统。

**第五步：选择安装磁盘**

在磁盘分区界面，你会看到一块"未分配的空间"（大小为你之前设置的虚拟磁盘大小）。选中它，直接点击 **"下一步"**。Windows 安装程序会自动创建必要的分区（EFI 系统分区、MSR 保留分区和主分区）。

**等待安装完成**

Windows 将自动执行以下步骤（约需 10–20 分钟，取决于你的硬件速度）：

1. 复制 Windows 文件
2. 展开文件
3. 安装功能
4. 安装更新
5. 完成安装

安装过程中虚拟机会**自动重启一次**，**此时不要按任何键**，让系统从虚拟硬盘引导。

### 9.4 完成初始设置（OOBE）

系统安装完成后，进入 Out-of-Box Experience（开箱体验）设置：

1. **选择区域**：根据你的实际位置选择
2. **键盘布局**：保留默认（微软拼音 + 美式键盘），或按需调整
3. **网络连接**：
   - 虚拟机使用 NAT 网络模式时会自动获取 IP 地址
   - 可以暂时选择 **"我没有 Internet 连接"**，稍后再配置（这样会创建本地账户而非 Microsoft 账户）
4. **设置用户名和密码**：为你的虚拟机账户起一个名称，并设置登录密码（强烈建议设置）
5. **隐私设置**：根据个人偏好选择是否开启各项隐私选项
6. 等待几分钟完成最终配置，进入桌面

> 🎉 恭喜！你已经在虚拟机中成功安装了 Windows 11！

---

## 10. 第七步：安装 VMware Tools

**VMware Tools 是虚拟机体验的关键组件，务必安装。** 它提供了：

- 🖥️ 优化的显卡驱动（高分辨率、流畅动画）
- 📋 宿主机与虚拟机之间的剪贴板共享
- 📁 文件拖放传输
- 🖱️ 鼠标无缝切换（无需按 Ctrl+Alt 释放光标）
- ⏱️ 时间同步
- 🔊 优化的音频驱动
- 🌐 增强的网卡驱动（VMXNET3）

### 安装步骤

1. 在 VMware Workstation 菜单栏中，点击 **VM → Install VMware Tools...**（安装 VMware Tools）
   - 如果菜单是灰色的，确保虚拟机已开机并进入了操作系统桌面
2. VMware 会在虚拟机内挂载一个虚拟光驱，包含 VMware Tools 安装程序
3. 在 Windows 虚拟机中，打开**文件资源管理器**，进入 **此电脑 → DVD 驱动器 (D:) VMware Tools**
4. 如果自动播放弹窗出现，选择"运行 setup64.exe"；否则手动双击 **setup64.exe**
5. 按照安装向导操作：
   - 选择安装类型：**"典型安装 (Typical)"**（推荐）
   - 点击"安装"，等待进度条走完
6. 安装完成后，选择 **"是，立即重新启动计算机"**，点击"完成"

> ✅ 重启后，你会发现分辨率自动适配了、鼠标可以在虚拟机和宿主机之间自由移动、剪贴板也可以互通了。

---

## 11. 虚拟机管理基础

### 11.1 创建快照（强烈推荐）

**快照（Snapshot）**是虚拟机当前状态的完整记录，包括内存、磁盘、设置等。类似于游戏的"存档"功能，你可以随时恢复到快照记录的状态。

**创建快照的最佳时机：**
- 操作系统刚刚安装完毕，安装 VMware Tools 之后
- 安装关键软件之前
- 进行高风险操作（系统配置修改、注册表编辑、软件测试）之前

**创建方法：**
1. 选中虚拟机，点击菜单 **VM → Snapshot → Take Snapshot...**
2. 输入快照名称（如"Clean Install - 2026-06-15"）和描述（如"干净系统+VMware Tools+系统更新完成"）
3. 点击 **"Take Snapshot"**

**恢复快照：**
- 菜单 **VM → Snapshot → Snapshot Manager**（或按 `Ctrl + M`）
- 选择要恢复的快照，点击 **"Go to"**

> 💡 快照会占用额外的磁盘空间。对于长期使用的虚拟机，建议保持 2–3 个关键快照，不需要的快照及时删除以释放空间。

### 11.2 虚拟机电源管理

| 操作 | 菜单路径 | 说明 |
|------|---------|------|
| **开机** | VM → Power → Power On | 正常启动虚拟机 |
| **关机** | VM → Power → Shut Down Guest | 向客户操作系统发送关机信号（推荐） |
| **强制关机** | VM → Power → Power Off | 相当于拔电源，仅在系统无响应时使用 |
| **挂起** | VM → Power → Suspend | 保存当前状态并暂停（下次恢复时继续，不丢失工作） |
| **重启** | VM → Power → Reset | 强制重启（等同于按物理机重启键） |

### 11.3 网络模式说明

| 模式 | 说明 | 典型场景 |
|------|------|---------|
| **NAT** | 虚拟机通过宿主机 IP 访问外网，宿主机外部无法直接访问虚拟机 | 上网浏览、下载更新（**默认推荐**） |
| **桥接 (Bridged)** | 虚拟机与宿主机在同一局域网中，拥有独立 IP | 需要从其他设备访问虚拟机服务时 |
| **仅主机 (Host-only)** | 虚拟机只能与宿主机通信，不能访问外网 | 隔离的本地实验环境 |
| **自定义** | 可组合多个虚拟网络 | 复杂的网络拓扑实验 |

### 11.4 常用文件操作

- **复制虚拟机**：在 VMware 库中右键虚拟机 → **Clone...**（克隆），或直接复制整个虚拟机文件夹到其他位置，再用 **File → Open** 打开 `.vmx` 文件
- **移动虚拟机**：直接移动虚拟机文件夹，然后 File → Open 打开即可（如果虚拟机处于挂起状态，建议先关机再移动）
- **删除虚拟机**：右键虚拟机 → **Manage → Delete from Disk**（彻底删除所有文件）

### 11.5 备份建议

| 备份方式 | 说明 |
|---------|------|
| **完整文件夹复制** | 关机后复制整个虚拟机文件夹到外部存储（最可靠） |
| **导出为 OVF** | 菜单 File → Export to OVF，可跨平台导入 |
| **快照** | 适合短期回滚，不适合长期备份 |
| **定期自动化** | 可编写脚本定期备份 `.vmx` 和 `.vmdk` 文件 |

---

## 12. 常见问题与排查

### 12.1 虚拟机启动报错："VMware Workstation and Hyper-V are not compatible"

**原因**：Windows Hyper-V 仍在启用状态。

**解决方法**：
1. 按第 6.1 节所述方法关闭 Hyper-V 及相关功能
2. 以管理员身份运行命令提示符，执行：
   ```shell
   bcdedit /set hypervisorlaunchtype off
   ```
3. 重启电脑

### 12.2 虚拟机启动后性能极其缓慢/卡顿

**可能原因与解决方案**：

| 原因 | 解决方案 |
|------|---------|
| Hyper-V 未彻底关闭 | 执行 `bcdedit /set hypervisorlaunchtype off` 并重启 |
| 内存分配不足 | 为虚拟机增加 RAM 分配（关机后修改） |
| 磁盘在机械硬盘上 | 将虚拟机文件夹迁移至 SSD |
| 未安装 VMware Tools | 按第 10 节安装 VMware Tools |
| 宿主机资源紧张 | 关闭宿主机不必要的应用程序 |

### 12.3 Windows 11 安装提示"此电脑不符合安装此版本 Windows 的最低系统要求"

**原因**：未启用 vTPM 或安全启动。

**解决方法**：
1. 关闭虚拟机
2. 编辑虚拟机设置：
   - **Options → Access Control**：点击 **"Encrypt..."** 加密虚拟机
   - **Hardware → Add...**：添加 **"Trusted Platform Module"**
   - **Options → Advanced → Firmware type**：确保选择了 **UEFI**，并勾选 **"Enable Secure Boot"**
3. 重新启动虚拟机并尝试安装

### 12.4 虚拟机无法连接网络

**排查步骤**：

1. 确认虚拟机的网络适配器设置：编辑虚拟机设置 → 网络适配器，确认已勾选 **"Connected"** 和 **"Connect at power on"**
2. 尝试切换网络模式：例如从 NAT 切换到桥接，看是否恢复
3. 在 VMware 菜单中：**Edit → Virtual Network Editor** → 点击 **"Change Settings"** → 选择默认网络 → **"Restore Defaults"**
4. 在客户操作系统内检查网卡是否被禁用

### 12.5 虚拟机随宿主机一起进入睡眠/休眠后无法唤醒

**解决方法**：先通过 `Ctrl + Alt` 释放光标，然后在宿主机重新唤醒后，尝试：
1. 点击虚拟机窗口，按任意键唤醒
2. 如果无响应，选择菜单 **VM → Power → Restart Guest**
3. 如果仍然无响应，选择 **VM → Power → Power Off** 强制关机后重启

### 12.6 安装 VMware Tools 时菜单为灰色

**原因**：虚拟机未完成启动或操作系统中没有可识别的桌面环境。

**解决方法**：
1. 确保虚拟机已完全进入操作系统桌面
2. 如果问题依旧，尝试手动挂载：
   - 在 VMware 安装目录（`C:\Program Files\VMware\VMware Workstation\`）找到 `windows.iso`
   - 在虚拟机设置中，将 CD/DVD 驱动器手动挂载到该 ISO
   - 在客户机内打开光驱手动运行安装程序

### 12.7 26H1 特有：旧版 32 位虚拟机无法启动

**原因**：VMware Workstation Pro 26H1 彻底移除了 32 位架构支持。

**解决方案**：
- 如果你仍需运行 32 位客户操作系统，请保留并使用 **VMware Workstation Pro 17.x**（17.6.x 为最后的稳定版本）
- 可以将 17.x 和 26H1 分别安装在不同电脑上，或将旧虚拟机迁移至其他虚拟化平台（如 VirtualBox）

### 12.8 遇到安装过程中windows蓝屏
如果你装win11，不论是business版本镜像，还是LSTC版本镜像，在vmware workstation pro中，启动启动就蓝屏了，或者如下图所示的样子：
![蓝屏报错](65cd2699/windows_crash_dump_code.png)

具体代码是：**Stop code: WHEA_UNCORRECTABLE_ERROR (0x124)**
这个问题，我目前还没有仔细分析crash的minidump，但是通过我反复比对，基本可以确认，这个就是**底层的兼容性问题**。
如果你跟我遇到一样的问题，虚拟机的虚拟化关闭了，宿主机虚拟化开了，内存完整性什么的都是关的。
那基本上就一个问题，**新版本的vmware创建磁盘格式，默认都是NVME（SSD）**，如果你vmware创建的虚拟磁盘**还在机械硬盘**，此时底层虚拟化**大概率会有兼容问题。**

解决方案应该有两个：
    - 迁移虚拟镜像目录到固态硬盘上
    - 重新安装镜像，不是通过典型（typical）安装，而是通过自定义（custom）安装，到磁盘格式这一个配置的时候，选SATA模式，如下图所示：
        ![默认格式nvme](65cd2699/default_nvme.png)

我没试第一个，我是新建的虚拟机系统，我直接删了重新装了，通过第二个方式解决了，没有蓝屏报错自动重启了。
**如果你遇到了相同的情况，不妨试试看。**

---

## 13. 总结

恭喜你完成了本教程的学习！让我们回顾一下你掌握的内容：

| 章节 | 你学会了 |
|------|---------|
| Broadcom 注册与下载 | 注册 Broadcom 账号并从官方渠道免费获取 VMware Workstation Pro 26H1 |
| 安装前准备 | 关闭 Hyper-V、确认 BIOS 虚拟化已开启、清理旧版本 |
| 软件安装 | 在 Windows 上完成 VMware Workstation Pro 26H1 的安装 |
| 创建虚拟机 | 使用向导创建一台配置合理的虚拟机 |
| 安装操作系统 | 在虚拟机中安装 Windows 11（或其他操作系统） |
| 安装 VMware Tools | 获得流畅的图形、剪贴板共享等完整体验 |
| 虚拟机管理 | 掌握快照、电源管理、备份等基本运维操作 |

### 下一步建议

- 🐧 尝试安装一个 Linux 发行版（如 Ubuntu 26.04 LTS），体验不同操作系统
- 🌐 创建多台虚拟机并配置不同的网络模式，搭建小型虚拟网络实验室
- 📸 养成为重要虚拟机创建快照的好习惯
- 🔧 探索 VM → Settings 中的更多高级选项，如共享文件夹、VMware 虚拟打印机等

### 参考资源

- [VMware Workstation Pro 26H1 官方文档](https://techdocs.broadcom.com/us/en/vmware-cis/desktop-hypervisors/workstation-pro/26H1.html)
- [VMware Workstation Pro 26H1 Release Notes](https://techdocs.broadcom.com/us/en/vmware-cis/desktop-hypervisors/workstation-pro/26H1/release-notes/vmware-workstation-pro-26h1-release-notes.html)
- [VMware 官方博客 - 26H1 发布公告](https://blogs.vmware.com/cloud-foundation/2026/05/14/announcing-vmware-workstation-and-fusion-26h1/)
- [Broadcom 支持门户 - 免费下载](https://support.broadcom.com/group/ecx/productdownloads?subfamily=VMware%20Workstation%20Pro&freeDownloads=true)
- [Microsoft Windows 11 下载](https://www.microsoft.com/software-download/windows11)
- [Ubuntu 官方下载](https://ubuntu.com/download/desktop)

---

> 📝 **本文档将持续更新**，如果在新版本中遇到任何问题或有操作建议，欢迎反馈。

---

*教程编写日期：2026 年 6 月 15 日*
*基于 VMware Workstation Pro 26H1 (Build 25388281)*
