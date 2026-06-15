---
title: tampermonkey油猴脚本扩展使用教程
abbrlink: b67c451a
date: 2026-06-16 03:43:36
updated: 2026-06-16 03:43:36
tags:
  - 油猴脚本
  - 浏览器扩展
categories: 工具
---

Tampermonkey（油猴插件）安装使用完整教程

> 本文档基于 2026-06-16 的最新信息汇总，整理自 Tampermonkey 官方网站、Greasy Fork、OpenUserJS 等权威来源，经多源交叉验证。

<!-- more -->


---

## 📑 目录

1. [什么是 Tampermonkey](#一什么是-tampermonkey)
2. [Tampermonkey 安装方法](#二tampermonkey-安装方法)
   - [2.1 Chrome 浏览器](#21-chrome-浏览器)
   - [2.2 Microsoft Edge 浏览器](#22-microsoft-edge-浏览器)
   - [2.3 Firefox 浏览器](#23-firefox-浏览器)
   - [2.4 Safari 浏览器](#24-safari-浏览器)
   - [2.5 Opera 浏览器](#25-opera-浏览器)
3. [油猴脚本获取渠道](#三油猴脚本获取渠道)
   - [3.1 Greasy Fork（首选）](#31-greasy-fork首选)
   - [3.2 OpenUserJS（备选）](#32-openuserjs备选)
   - [3.3 ScriptCat 脚本猫](#33-scriptcat-脚本猫)
4. [脚本的基本操作](#四脚本的基本操作)
   - [4.1 安装脚本](#41-安装脚本)
   - [4.2 启用与禁用脚本](#42-启用与禁用脚本)
   - [4.3 编辑脚本](#43-编辑脚本)
   - [4.4 删除脚本](#44-删除脚本)
   - [4.5 更新检查](#45-更新检查)
   - [4.6 仪表盘（Dashboard）](#46-仪表盘dashboard)
   - [4.7 Tampermonkey 全局设置](#47-tampermonkey-全局设置)
5. [用户脚本元数据（Metadata Block）详解](#五用户脚本元数据metadata-block-详解)
6. [常用脚本推荐](#六常用脚本推荐)
   - [6.1 综合 TOP 6](#61-综合-top-6)
   - [6.2 视频类脚本](#62-视频类脚本)
   - [6.3 网盘类脚本](#63-网盘类脚本)
   - [6.4 知乎类脚本](#64-知乎类脚本)
   - [6.5 GitHub 类脚本](#65-github-类脚本)
7. [常见问题与故障排查](#七常见问题与故障排查)
8. [进阶技巧与安全建议](#八进阶技巧与安全建议)
9. [参考来源](#九参考来源)

---

## 一、什么是 Tampermonkey

**Tampermonkey**（俗称"油猴"）是最受欢迎的**用户脚本管理器**（Userscript Manager）浏览器扩展，它本身并不直接提供功能，而是作为一个"容器"，让你能在任意网页上加载和运行用户脚本（Userscript），从而实现：

- 去除网页广告
- 解除网页复制/右键限制
- 增强视频播放体验
- 自动翻页
- 重新美化网页布局
- 自动化数据抓取
- …等等任何用 JavaScript 能做到的事

### 当前版本

- **最新稳定版**：v5.5.0（发布日期：2026-05-08；截至本文档更新时）
- **多语言支持**：包含简体中文、繁体中文、英文等
- **完全免费**

### v5.5.0 重要变更

> **"注入用户脚本现在需要特殊的扩展权限"**（Injecting a userscript now requires a special extension permission.）

这意味着在新版 Chrome 上安装/注入脚本时，需要授予 `userScripts` 权限，并可能需要开启"允许用户脚本"开关或"开发者模式"。具体操作见下文故障排查部分。

---

## 二、Tampermonkey 安装方法

Tampermonkey 官方支持以下 5 大主流浏览器：

| 浏览器 | 安装方式 |
|--------|---------|
| Chrome | Chrome Web Store（3 个版本）/ 官网直接下载 .crx |
| Microsoft Edge | Chrome Web Store（共用）/ Edge 加载项 |
| Firefox | Mozilla Add-ons（addons.mozilla.org） |
| Safari | App Store / 官方 Safari 扩展 |
| Opera Next | Chrome Web Store（共用） |

### 2.1 Chrome 浏览器

Chrome 提供 **3 个发行版本**，请根据你的 Chrome 版本选择：

| 版本 | 适用 Chrome 版本 | 用途 |
|------|----------------|------|
| **Stable（稳定版）** | Chrome 120+ | 推荐日常使用 |
| **Legacy MV2（遗留版）** | Chrome 71–119 | 老版本 Chrome 使用 |
| **BETA（测试版）** | Chrome 120+ | 尝鲜最新功能 |

#### 方式一：通过 Chrome Web Store 安装（推荐）

1. 打开 Chrome 浏览器，访问以下任一链接：
   - **稳定版**：搜索 `Tampermonkey` 或访问 [Chrome Web Store 页面](https://chromewebstore.google.com/detail/dhdgffkkebhmkfjojejmpbldmpobfkfo)
   - **遗留版（MV2）**：扩展 ID `lcmhijbkigalmkeommnijlpobloojgfn`
   - **测试版（BETA）**：扩展 ID `gcalenpjmijncebpfijmoaglllgpjagf`
2. 点击"添加至 Chrome"（Add to Chrome）
3. 在弹出的权限确认框中点击"添加扩展程序"
4. 安装完成后，浏览器右上角（地址栏右侧）会出现黑色猴子图标 🐒

#### 方式二：从官网直接下载 .crx 安装

适合无法访问 Chrome Web Store 的情况：

1. 访问 [https://www.tampermonkey.net/index.php?browser=chrome](https://www.tampermonkey.net/index.php?browser=chrome)
2. 下载对应版本的 .crx 文件：
   - `tampermonkey_stable.crx`
   - `tampermonkey_legacy.crx`
   - `tampermonkey_beta_current.crx`
3. 打开 Chrome，访问 `chrome://extensions/`
4. 打开右上角"**开发者模式**"（Developer mode）
5. 将 .crx 文件拖入扩展页面完成安装

> ⚠️ **注意**：Chrome 120+ 上通过 .crx 直接安装的稳定版（MV2）会自动更新到 MV3 版本。如不希望更新，可禁用自动更新。

#### 方式三：Chrome 安装后的必要设置（v5.5.0+）

由于 v5.5.0 引入了 userScripts 权限要求，安装后还需：

1. 访问 `chrome://extensions/`
2. 找到 Tampermonkey，点击"**详细信息**"
3. 确保"**允许访问用户脚本**"或"**允许用户脚本**"开关已打开
4. 某些版本需要同时开启"**开发者模式**"

### 2.2 Microsoft Edge 浏览器

Edge 基于 Chromium，方法与 Chrome 类似：

1. 打开 Edge 扩展商店（[microsoftedge.microsoft.com/addons](https://microsoftedge.microsoft.com/addons)）
2. 搜索 "Tampermonkey" 并安装
3. 或者：直接复用 Chrome Web Store 的链接，Edge 也能识别并提示安装
4. 安装后地址栏右侧出现 Tampermonkey 图标

### 2.3 Firefox 浏览器

Firefox 上有多个用户脚本管理器可选：

| 脚本管理器 | 特性 | 推荐场景 |
|-----------|------|---------|
| **Tampermonkey** | 功能最全面，闭源免费 | 普通用户首选 |
| **Violentmonkey** | 开源（MIT 协议），轻量简洁 | 注重隐私/开源的进阶用户 |
| **Greasemonkey** | Firefox 原生，老牌项目 | 怀旧用户；功能更新较慢 |

> 💡 **Tampermonkey vs Violentmonkey**：Tampermonkey 功能最丰富（云同步、高级编辑器等），但**闭源**；Violentmonkey 完全**开源**、体积更小，大部分脚本可直接兼容。如果你注重开源和隐私，Violentmonkey 是很好的替代方案。

#### 安装 Tampermonkey（推荐）

1. 打开 Firefox，访问 [https://addons.mozilla.org/](https://addons.mozilla.org/)
2. 搜索 "Tampermonkey"
3. 点击"添加到 Firefox"（Add to Firefox）
4. 在弹窗中确认权限，点击"添加"
5. 安装完成后浏览器工具栏会出现 Tampermonkey 图标

### 2.4 Safari 浏览器

Safari 提供两个版本：

| 版本 | 安装渠道 |
|------|---------|
| **现代版 Safari**（macOS 11+/iOS 15+） | Mac App Store |
| **Safari Classic**（旧版） | 官方提供的 Safari 扩展包 |

#### 现代版 Safari 安装

1. 打开 Mac App Store
2. 搜索 "Tampermonkey" 或 "Userscripts"
3. 点击"获取"安装
4. 安装后需要在 Safari 的「设置 → 扩展」中启用 Tampermonkey 并授予「所有网站」权限

> ⚠️ 注意：Safari Classic 渠道已多年未更新，建议现代用户使用现代版或考虑更换为 Chrome/Edge/Firefox。

### 2.5 Opera 浏览器

Opera Next（基于 Chromium）使用与 Chrome 相同的方式：

- 通过 Chrome Web Store 安装（Opera 兼容）
- 或从 Tampermonkey 官网下载 .crx 后开发者模式安装

Opera 上也可选：
- **Tampermonkey**
- **Violentmonkey**

> ✅ 安装完扩展后，接下来你需要获取脚本才能在网站上生效——下一章将介绍脚本获取渠道。

---

## 三、油猴脚本获取渠道

### 3.1 Greasy Fork（首选）

🌐 官网：[https://greasyfork.org/](https://greasyfork.org/) | 中文：[https://greasyfork.org/zh-CN](https://greasyfork.org/zh-CN)

**Greasy Fork** 是目前**最受欢迎、规模最大**的油猴脚本仓库，描述自己是：

> "欢迎来到 Greasy Fork，一个专为用户脚本打造的网站。"

**特点**：
- ✅ 所有脚本**免费**安装
- ✅ 数量多达数十万个脚本
- ✅ 支持**中文界面**（zh-CN）
- ✅ 提供**高级搜索**功能（按安装量、评分、日期、语言等筛选）
- ✅ 提供**按站点浏览**功能（例如直接浏览 YouTube、知乎、B 站、百度网盘等站点的脚本）
- ✅ 官方推荐渠道（Tampermonkey 文档中明确推荐）

#### Greasy Fork 安装脚本的 3 步流程

1. **安装用户脚本管理器**
   - Chrome/Edge → Tampermonkey
   - Firefox → Tampermonkey / Greasemonkey / Violentmonkey
   - Safari → Tampermonkey / Userscripts
   - Opera → Tampermonkey / Violentmonkey

2. **找到想要的用户脚本**
   - 在脚本页面点击绿色的"**安装**"按钮
   - 用户脚本管理器会弹出确认对话框，核对脚本信息和权限

3. **访问目标网站**
   - 脚本会自动启动并生效，无需额外操作

### 3.2 OpenUserJS（备选）

🌐 官网：[https://openuserjs.org/](https://openuserjs.org/)

**OpenUserJS** 自称是 "A Presentational Userscripts Repository"（一个用户脚本展示仓库）。

**特点**：
- 收录脚本和库（libraries）
- 提供 4 种排序方式：Name、Installs、Rating、Updated
- 脚本 URL 格式：`/scripts/{author}/{script-name}`
- 收录大量脚本（数据截至 2026-06-16）
- 热门脚本示例：
  - **Bypass All Shortlinks Manual Captcha**（v96.5）
  - **Manga OnlineViewer**（v2026.06.08.build-2156）

### 3.3 ScriptCat 脚本猫

🌐 官网：[https://scriptcat.org/](https://scriptcat.org/)

**ScriptCat**（脚本猫）是一个新兴的开源脚本平台，主要面向中文用户，对国内网站（如 B 站、知乎、百度等）的脚本收录较为完善，可作为 Greasy Fork 的补充。

---

## 四、脚本的基本操作

### 4.1 安装脚本

**方式一：从 Greasy Fork / OpenUserJS 安装**

1. 打开 Greasy Fork（如 `https://greasyfork.org/zh-CN`）
2. 浏览或搜索你需要的脚本
3. 进入脚本详情页
4. 点击绿色的"**安装此脚本**"按钮
5. Tampermonkey 会弹出脚本预览窗口，显示脚本的元数据（@name、@match、@grant 等）
6. 检查 `@match` 是否合理（决定脚本在哪些网站上运行）
7. 点击"**安装**"确认

**方式二：安装本地 .user.js 文件**

1. 将 `.user.js` 脚本文件保存到本地
2. 直接将文件**拖入浏览器任意网页窗口**（不是扩展管理页面），Tampermonkey 会拦截并弹出安装确认对话框
3. 或者：打开 Tampermonkey 仪表盘 → 点击右上角"**+**" → 将脚本代码粘贴进去 → 保存
4. 也可以在浏览器地址栏输入 `file:///C:/path/to/script.user.js` 打开本地文件触发安装

**方式三：通过远程 URL 安装**

有些脚本作者直接提供 `.user.js` 文件的原始链接（形如 `https://example.com/script.user.js`），在浏览器地址栏打开该链接即可触发 Tampermonkey 安装对话框。

### 4.2 启用与禁用脚本

#### 临时启用/禁用（推荐用于排查）

1. 点击浏览器工具栏的 **Tampermonkey 图标** 🐒
2. 在弹出的弹出窗口中：
   - 看到当前页面上**正在运行**的脚本列表
   - 每个脚本旁边有开关按钮
   - 灰色 = 已禁用，绿色/亮色 = 已启用
   - 关闭开关即可临时禁用某个脚本

#### 永久启用/禁用

1. 点击 Tampermonkey 图标
2. 点击"**仪表盘**"（Dashboard）
3. 在脚本列表中点击某个脚本
4. 切换"**已启用**"开关

> 📌 弹出窗口顶部还会显示一个**总开关**，一键启用/禁用所有脚本。

> 📌 浏览器扩展图标上**显示的数字** = 当前页面上正在运行的脚本数量。

### 4.3 编辑脚本

你可以修改已安装脚本的代码（但修改会在脚本更新时被覆盖）：

1. 打开 Tampermonkey 仪表盘
2. 点击要编辑的脚本名
3. 切换到"**编辑器**"（Editor）标签
4. 修改代码
5. 点击"**保存**"（Ctrl/Cmd + S）

> ⚠️ **建议**：如果想长期自定义某个脚本，应先复制代码并通过本地 .user.js 文件安装，或切换为 `@updateURL` 指向自己的镜像。

### 4.4 删除脚本

#### 单个删除

1. 打开 Tampermonkey 仪表盘
2. 找到要删除的脚本
3. 点击该脚本右侧的"**垃圾桶**"图标
4. 确认删除

#### 批量删除

1. 在仪表盘列表中**多选**脚本（勾选复选框）
2. 点击底部的"**删除**"按钮

#### 卸载 Tampermonkey 本身

如果想完全卸载：

1. 打开浏览器的扩展管理页面：
   - Chrome: `chrome://extensions/`
   - Edge: `edge://extensions/`
   - Firefox: `about:addons`
2. 找到 Tampermonkey
3. 点击"**移除**"（Chrome/Edge）或"**卸载**"（Firefox）
4. 确认后 Tampermonkey 及其管理的所有脚本会被一并清除

### 4.5 更新检查

#### 自动更新

Tampermonkey 默认**每天自动检查一次**脚本更新。脚本元数据中的 `@updateURL` 或 `@version` 决定如何检查。

#### 手动检查更新

1. 点击 Tampermonkey 图标
2. 点击"**检查用户脚本更新**"（Check for userscript updates）
3. 等待扫描完成
4. 在列表中点击"**更新**"按钮升级单个脚本，或点击"**全部更新**"

#### 在仪表盘查看更新历史

仪表盘 → 选择脚本 → "**历史**"（History）标签可看到该脚本的所有版本变更。

### 4.6 仪表盘（Dashboard）

仪表盘是 Tampermonkey 的"控制中心"：

- **概览**：所有已安装脚本的列表
- **排序**：可按 Name、Installs、Rating、Updated 等排序
- **筛选**：按启用/禁用状态、来源筛选
- **新建脚本**：右上角 "+" 号可创建空白脚本
- **导入**：从本地文件导入
- **设置**：全局配置

#### 仪表盘快速打开方式

- 方式 1：点击 Tampermonkey 图标 → "仪表盘"
- 方式 2：地址栏输入 `chrome-extension://dhdgffkkebhmkfjojejmpbldmpobfkfo/options.html`
- 方式 3：右键 Tampermonkey 图标 → "选项"

### 4.7 Tampermonkey 全局设置

除了管理单个脚本外，Tampermonkey 本身也有全局配置，进入方式：仪表盘 → 右上角"**设置**"（Settings）标签。

#### 常用设置项

| 设置项 | 作用 | 建议值 |
|--------|------|--------|
| **脚本更新频率** | 控制多久检查一次脚本更新 | 默认「每天」，频繁使用的脚本可改为「每小时」 |
| **注入模式** | 脚本何时注入页面 | 默认「自动」即可；如遇兼容问题可尝试「即时注入」 |
| **全局排除** | 在所有脚本中排除特定 URL | 可添加不想让任何脚本运行的网站（如银行网站） |
| **黑名单** | 指定网站完全不加载任何脚本 | 与全局排除类似，适用于安全敏感网站 |
| **脚本存储位置** | 脚本数据存储路径 | 默认即可；高级用户可配合 OneDrive/Google Drive 做云同步 |
| **调试模式** | 输出更详细的日志 | 仅在排查问题或开发脚本时开启 |

#### 云同步配置

Tampermonkey 支持将脚本和设置同步到云端（Google Drive、OneDrive 等）：

1. 仪表盘 → 设置 → **同步**
2. 选择云服务提供商（Google Drive / OneDrive）
3. 登录授权后，脚本将自动备份到云端
4. 换设备/重装后可在新浏览器上恢复

> 💡 同步功能需要授予额外的 OAuth 权限，如果担心隐私，建议使用「导出/导入」手动迁移。

---

## 五、用户脚本元数据（Metadata Block）详解

每个用户脚本顶部都有一个 `// ==UserScript== ... // ==/UserScript==` 块，称为**元数据块**，用来告诉脚本管理器这个脚本的基本信息。Tampermonkey 支持以下常用指令：

### 5.1 常用元数据指令

| 指令 | 作用 | 示例 |
|------|------|------|
| `@name` | 脚本名称 | `// @name          My Script` |
| `@namespace` | 命名空间（避免冲突） | `// @namespace     https://example.com/` |
| `@version` | 版本号 | `// @version       1.0.0` |
| `@description` | 脚本描述 | `// @description   自动翻页+去广告` |
| `@author` | 作者 | `// @author        YourName` |
| `@match` | **匹配模式**（决定脚本在哪些 URL 运行） | `// @match         https://example.com/*` |
| `@include` | 类似 @match 但使用通配符 | `// @include       *://*.example.com/*` |
| `@exclude` | 排除匹配的 URL | `// @exclude       https://example.com/admin*` |
| `@exclude-match` | 排除模式（同 @match 语法） | `// @exclude-match https://example.com/login*` |
| `@noframes` | 不在 iframe 中运行 | `// @noframes` |
| `@icon` | 脚本图标 URL | `// @icon          https://.../icon.png` |
| `@run-at` | 脚本执行时机 | `// @run-at        document-start` |
| `@grant` | 授予 GM_* API 权限 | `// @grant         GM_setValue` |
| `@connect` | 声明 GM_xmlhttpRequest 可访问的域名 | `// @connect       example.com` |
| `@require` | 加载外部 JS 依赖 | `// @require       https://.../jquery.min.js` |
| `@resource` | 声明外部资源 | `// @resource      logo https://.../logo.png` |
| `@updateURL` | 更新检查 URL | `// @updateURL     https://.../script.meta.js` |
| `@downloadURL` | 下载 URL | `// @downloadURL   https://.../script.user.js` |
| `@supportURL` | 反馈/支持链接 | `// @supportURL    https://github.com/.../issues` |
| `@homepageURL` | 主页链接 | `// @homepageURL   https://github.com/...` |
| `@license` | 许可证 | `// @license       MIT` |

> ⚠️ **重要**：如果脚本使用了 `@grant GM_xmlhttpRequest`（跨域请求），务必同时声明 `@connect` 来指定允许访问的域名。例如 `// @connect api.example.com`。未声明 `@connect` 时，`GM_xmlhttpRequest` 会直接失败。

### 5.2 元数据示例

```javascript
// ==UserScript==
// @name            百度网盘直链下载助手
// @namespace       https://github.com/syhyz1990/baidupcs-web
// @version         3.0.0
// @description     解析百度网盘直链，突破限速下载
// @author          syhyz
// @match           https://pan.baidu.com/*
// @match           https://yun.baidu.com/*
// @icon            https://pan.baidu.com/favicon.ico
// @grant           GM_setValue
// @grant           GM_getValue
// @grant           GM_xmlhttpRequest
// @grant           GM_notification
// @grant           unsafeWindow
// @connect         pan.baidu.com
// @run-at          document-idle
// @updateURL       https://greasyfork.org/scripts/xxx/code/xxx.meta.js
// @license         MIT
// ==/UserScript==

(function() {
    'use strict';
    // 你的脚本代码
})();
```

### 5.3 @match 匹配模式语法

`@match` 使用 Chrome 扩展的 [match pattern](https://developer.chrome.com/docs/extensions/develop/concepts/match-patterns) 语法：

```
<scheme>://<host>/<path>
```

- `<scheme>`：`*`（任意）、`http`、`https`、`file`、`ftp`
- `<host>`：完整域名（如 `example.com`）或以 `.` 开头的子域
- `<path>`：路径，可用 `*` 通配

**示例**：

| 匹配模式 | 匹配的 URL |
|---------|-----------|
| `https://example.com/*` | `https://example.com/任何/路径` |
| `*://*.google.com/*` | 所有 Google 子域 |
| `https://*/*` | 所有 https 页面 |
| `http://localhost/*` | 仅本地 |

---

## 六、常用脚本推荐

> 📊 数据截至 2026-06-16，安装量会持续变动。

### 6.1 综合 TOP 6

按 Greasy Fork 总安装量排序的明星脚本：

| 排名 | 脚本名 | 安装量 | 作用 |
|------|--------|--------|------|
| 🥇 | **AC-Baidu 净化搜索**（全名见下方备注） | **3,426,616** | 清理百度/谷歌/搜狗/必应搜索结果中的重定向链接、屏蔽广告、净化搜索 |
| 🥈 | Remove web limits(modified) | **2,228,757** | 万能网页限制解除，突破禁止复制/右键/选择/拖动等限制 |
| 🥉 | HTML5 Video Playing Tools | **1,279,789** | HTML5 视频万能播放工具，支持倍速、截图、画中画、下载等 |
| 4 | Github Enhancement - High Speed Download | **896,023** | GitHub 增强：高速下载 Release/Code/Zip、单文件/文件夹下载加速 |
| 5 | AutoPager | **576,725** | 通用自动翻页脚本，聚合多页内容 |
| 6 | Kill Baidu AD | **426,250** | 彻底清理百度搜索结果中的广告 |

> 📌 **备注**：第 1 名脚本全名为 `AC-baidu-google_sogou_bing_RedirectRemove_favicon_adaway_TwoLine`，在 Greasy Fork 搜索 "AC-baidu" 即可找到。

### 6.2 视频类脚本

| 脚本名 | 适用平台 | 功能 |
|--------|---------|------|
| **HTML5 Video Playing Tools** | 通用 HTML5 视频 | 倍速、画中画、截图、调节亮度/对比度/饱和度 |
| **B站(bilibili)综合增强** | bilibili.com | 视频下载、弹幕过滤、宽屏模式 |
| **YouTube 一键下载** | youtube.com | 解析 YouTube 视频并下载（注意版权风险） |
| **视频倍速控制** | 通用 | 持久化保存倍速设置 |
| **Global Speed** | HTML5 视频 | 统一控制所有视频倍速 |

> 🎬 **B 站脚本**专题：[https://greasyfork.org/en/scripts?site=bilibili.com&sort=total_installs](https://greasyfork.org/en/scripts?site=bilibili.com&sort=total_installs)

### 6.3 网盘类脚本

| 脚本名 | 适用平台 | 功能 |
|--------|---------|------|
| **百度网盘直链下载助手** | pan.baidu.com | 解析真实下载链接，突破限速 |
| **阿里云盘** | aliyundrive.com | 增强分享/下载体验 |
| **OneDrive 直链** | onedrive.live.com | 解析 OneDrive 文件直链 |

> 📁 **百度网盘**专题：[https://greasyfork.org/en/scripts?site=pan.baidu.com&sort=total_installs](https://greasyfork.org/en/scripts?site=pan.baidu.com&sort=total_installs)

### 6.4 知乎类脚本

| 脚本名 | 功能 |
|--------|------|
| **知乎增强** | 一键收起/展开答案、移除广告、净化标题 |
| **知乎去除登录弹窗** | 关闭强制登录弹窗 |
| **知乎直链** | 绕过外链跳转 |

> 💡 **知乎**专题：[https://greasyfork.org/zh-CN/scripts/by-site/zhihu.com](https://greasyfork.org/zh-CN/scripts/by-site/zhihu.com)

### 6.5 GitHub 类脚本

| 脚本名 | 功能 |
|--------|------|
| **Github Enhancement - High Speed Download** | 高速下载 Release、Code 压缩包、单个文件 |
| **GitHub File Tree** | 在文件页显示仓库目录树 |
| **Octotree**（浏览器扩展） | 左侧显示仓库代码树 |
| **Refined GitHub** | GitHub 体验增强：PR 优化、文件查看器等 |
| **GitHub 镜像加速** | 自动跳转到镜像站 |

### 6.6 其他实用脚本推荐

| 类别 | 推荐脚本 |
|------|---------|
| 🔍 **搜索** | AC-baidu-redirect 净化搜索、Search By Image、SearchJumper |
| 📖 **阅读** | 简书/掘金/SegmentFault 净化、强制启用复制 |
| 🛒 **购物** | 京东/淘宝历史价格、隐藏已购买 |
| 🎓 **学习** | 网易云课堂/MOOC 视频下载、智慧树网课助手 |
| 🤖 **AI 辅助** | 网页内容 AI 摘要、ChatGPT 侧边栏、AI 翻译增强、搜索引擎 AI 集成 |
| 🌐 **翻译** | 划词翻译、谷歌翻译助手 |
| 📊 **社交** | 微博/豆瓣净化、Twitter 视频下载 |

---

## 七、常见问题与故障排查

### 7.1 v5.5.0 userScripts 权限问题（重要）

**现象**：升级到 v5.5.0 后，脚本无法注入或 Tampermonkey 弹出"需要特殊权限"提示。

**原因**：Chrome 引入的新安全机制要求扩展必须显式声明 `userScripts` 权限。

**解决方法（FAQ Q209）**：

1. 打开 `chrome://extensions/`
2. 找到 Tampermonkey
3. 点击"**详细信息**"
4. 开启"**允许用户脚本**"或"**Allow User Scripts**"
5. 某些版本需要同时开启 Chrome 顶部的"**开发者模式**"
6. 重启浏览器

### 7.2 脚本不生效

**排查步骤**：

1. ✅ 确认 Tampermonkey 已启用（图标不是灰色的）
2. ✅ 确认脚本已启用（在仪表盘或弹出窗口中检查）
3. ✅ 确认 `@match` 包含当前页面的 URL（脚本列表旁应显示绿色圆点 = 在当前页运行）
4. ✅ 打开浏览器开发者工具（F12）→ Console，查看是否有报错
5. ✅ （高级）检查页面 CSP（内容安全策略）是否阻止了脚本——在 DevTools Console 中查找以 `Content Security Policy` 开头的报错
6. ✅ 尝试禁用其他扩展（如 AdBlock、uBlock 等）排除冲突
7. ✅ 清除浏览器缓存后重试

### 7.3 脚本显示已运行但实际无效果

**可能原因**：

- 脚本的 `@run-at` 时机不对，DOM 还没加载完
- 目标网站改版，脚本的 `document.querySelector` 找不到元素
- 网站启用了 CSP/Shadow DOM 导致脚本失效

**解决方法**：

1. 编辑脚本，把 `@run-at document-idle` 改为 `@run-at document-end` 或 `@run-at document-start`
2. 查看脚本的"反馈/支持"页，可能有更新版本
3. 自己用 DevTools 调试：Console 报错是定位问题的关键

### 7.4 多个脚本在同一页面冲突

**现象**：安装了多个针对同一网站的脚本，其中一个或几个表现异常。

**原因**：
- 多个脚本修改同一 DOM 元素产生竞争
- 脚本执行顺序不确定
- 不同脚本使用的库/依赖版本不一致

**排查方法**：
1. 在弹出窗口中**逐个禁用**脚本，找出冲突源
2. 使用 `@run-at` 控制执行顺序（`document-start` > `document-end` > `document-idle`）
3. 使用 `@exclude-match` 让脚本 A 排除脚本 B 的作用页面，避免重叠
4. 检查 Console 是否有 JS 报错，确定冲突位置
5. 联系脚本作者告知冲突情况

### 7.5 Tampermonkey 弹出窗口看不到

**可能原因**：

- 图标被隐藏在浏览器菜单（…）里
- 浏览器设置为在隐身模式下禁用扩展

**解决方法**：

1. **Chrome**：地址栏右侧的"扩展"拼图图标 🧩 → 找到 Tampermonkey → 点击"图钉"📌固定到工具栏
2. **Firefox**：菜单 → 更多工具 → 自定义工具栏 → 拖动 Tampermonkey 到工具栏

### 7.6 安装脚本时弹出"未通过审核"提示

**现象**：在 Greasy Fork 上安装某些脚本时显示警告"该脚本未通过审核"。

**原因**：

- 脚本是最近新创建的
- 脚本被举报可能含有可疑代码
- 脚本使用了过多的 `GM_*` 权限

**建议**：

- 仔细阅读脚本的代码（安装前 Tampermonkey 会显示源码）
- 只安装**安装量大、评价高、来源可信**的脚本
- 警惕要求 `@grant unsafeWindow` 和 `@grant GM_xmlhttpRequest` 同时存在的脚本

### 7.7 与广告拦截扩展冲突

**现象**：安装去广告脚本后，uBlock/AdBlock 显示页面仍有广告。

**解决方法**：

1. 关闭 Tampermonkey 内的"去广告脚本"（可能功能重复）
2. 或者在 uBlock 中将目标网站加入白名单，让 Tampermonkey 脚本处理
3. 检查脚本是否针对该网站指定了 `@match`

### 7.8 移动端（手机）能用 Tampermonkey 吗？

- **Android**：推荐使用 **Firefox for Android** + Tampermonkey（Android 版可在 Mozilla Add-ons 安装）
- **Android**（备选）：**Microsoft Edge for Android** 同样支持 Chrome 扩展，可直接安装 Tampermonkey
- **iOS**：iOS 15+ 的 Safari 扩展支持有限，**推荐使用 Userscripts** 扩展（App Store）
- **Kiwi Browser**（Android）：自带 Chrome 内核并支持扩展
- **Yandex Browser**（Android）：内置用户脚本支持

### 7.9 Chrome 升级后 Tampermonkey 失效

**背景：Chrome Manifest V2 → V3 迁移**

Google 从 2024 年开始逐步淘汰 Chrome 扩展的 Manifest V2（MV2）规范，要求所有扩展迁移到 Manifest V3（MV3）。这对用户脚本管理器的影响是：

- MV3 限制后台脚本的持久运行，`unsafeWindow` 的访问方式发生改变
- 部分老旧脚本可能依赖 MV2 特性而无法正常工作
- Tampermonkey 于 v5.5.0 完成了 MV3 迁移，同时提供了 Legacy MV2 版本供过渡

**解决方法**：

1. **首选**：升级到 Tampermonkey 稳定版（MV3，原生支持 v5.5.0+）
2. 如果是 MV2 版本被 Chrome 禁用，需要**卸载旧版**后重新安装稳定版
3. **临时过渡**：在 `chrome://flags/` 中搜索并启用 `ExtensionManifestV2Availability` 标志（仅为过渡方案，Google 随时可能完全移除）
4. 如果特定脚本在 MV3 下不工作，切换到 Firefox + Tampermonkey（Firefox 仍完整支持 MV2 等效能力）

> 💡 大多数主流脚本已经适配 MV3，如果你常用的脚本失效，建议检查 Greasy Fork 上是否有更新版本。

### 7.10 脚本更新后功能异常

**解决方法**：

1. 在仪表盘找到该脚本 → "**历史**" → 选择旧版本 → "**回滚**"
2. 或者先**禁用**该脚本，等作者修复
3. 在脚本的 @supportURL 提交 Issue 反馈

### 7.11 如何备份/迁移脚本

**导出**：

1. 打开 Tampermonkey 仪表盘
2. 点击右上角"**实用工具**"（Utilities）→ "**导出**"（Zip 压缩包）

**导入**：

1. 在新浏览器安装 Tampermonkey
2. 打开仪表盘 → "**实用工具**" → "**导入**"（选择之前的 .zip 文件）

**同步（高级）**：可以配合 OneDrive/坚果云/Google Drive 把脚本文件夹同步到云端。

---

## 八、进阶技巧与安全建议

### 8.1 自建脚本入门

#### 最简示例：Hello World

让我们从零开始编写一个最简单的油猴脚本——在百度首页弹出一个"Hello World"提示。

1. 打开 Tampermonkey 仪表盘
2. 右上角点"**+**"新建脚本
3. 清空默认内容，粘贴以下代码：

```javascript
// ==UserScript==
// @name         Hello World
// @namespace    https://example.com/
// @version      0.1
// @description  我的第一个油猴脚本
// @author       你的名字
// @match        https://www.baidu.com/*
// @grant        none
// ==/UserScript==

(function() {
    'use strict';
    alert('Hello World! 我的第一个脚本运行成功！');
})();
```

4. 按 `Ctrl+S` 保存
5. 打开 [www.baidu.com](https://www.baidu.com)，你会看到弹出一个"Hello World"提示！

#### 关键概念解析

- **`@match`**：决定脚本在哪个网站上运行。上面例子中只在 `www.baidu.com` 生效。你可以改为 `https://*/*` 让它在所有 HTTPS 页面上运行
- **`@grant none`**：声明不使用任何 GM_* 特权 API。如果脚本需要调用 `GM_setValue` 等 API，需要明确声明
- **`'use strict'`**：JavaScript 严格模式，推荐保留
- **`(function() { ... })()`**：IIFE（立即执行函数），避免变量污染全局作用域

#### 常用 GM_* API 快速参考

| API | 作用 | 示例 |
|-----|------|------|
| `GM_setValue(key, value)` | 持久化存储数据 | `GM_setValue('lastVisit', Date.now())` |
| `GM_getValue(key, defaultValue)` | 读取持久化数据 | `GM_getValue('lastVisit', 0)` |
| `GM_xmlhttpRequest(details)` | 跨域 HTTP 请求 | `GM_xmlhttpRequest({url, onload})` |
| `GM_notification(text, title, image, onclick)` | 桌面通知 | `GM_notification('下载完成！')` |
| `GM_addStyle(css)` | 注入自定义 CSS | `GM_addStyle('.ad {display:none}')` |
| `GM_setClipboard(text)` | 写入剪贴板 | `GM_setClipboard('已复制')` |

> ⚠️ 使用 `GM_*` API 前，必须在元数据块的 `@grant` 中声明对应的权限，如 `// @grant GM_setValue`。另外，如果使用 `GM_xmlhttpRequest`，还需要 `@connect` 声明目标域名。

#### 进阶建议

1. 从 Greasy Fork 找一个简单脚本研究源码
2. 善用 `console.log()` 在 DevTools Console 中输出调试信息
3. 设置 `@run-at document-end` 可确保 DOM 已加载完毕再执行脚本

### 8.2 调试技巧

- 按 F12 打开 DevTools
- 在 Sources 面板找到 Tampermonkey 注入的脚本
- 设置断点调试
- 使用 `console.log` 输出调试信息

### 8.3 安全建议 ⚠️

1. **只安装可信来源的脚本**（Greasy Fork 评分 + 安装量 + 作者声誉）
2. **安装前审查代码**：Tampermonkey 安装时会显示源码，至少应扫一眼
3. **谨慎授予权限**：警惕 `@grant GM_xmlhttpRequest`（可绕过同源策略访问任意网站）
4. **定期审计**：仪表盘里清理不再使用的脚本
5. **避免泄露密码**：切勿将密码、API Key 等敏感信息硬编码在脚本中。如需存储认证信息，优先使用 `GM_setValue` 加密存储（注意：`GM_setValue` 数据为明文存储在浏览器本地，安全性有限，仅供非敏感用途）
6. **关注更新**：脚本更新可能引入恶意代码，关注作者动态

### 8.4 配合其他工具使用

- **AdBlock/uBlock Origin**：处理广告 + 跟踪器
- **Tampermonkey**：处理功能性增强
- **Decentraleyes**：本地化 CDN 资源
- **ClearURLs**：清理 URL 中的跟踪参数

> 💡 油猴脚本和广告拦截器**不是替代关系**，而是**互补关系**。

### 8.5 性能优化

- 减少安装脚本数量（经验上控制在 20 个以内最佳；脚本越多浏览器越慢）
- 关闭不需要的 `@match`
- 对不常用的脚本**临时禁用**而非卸载
- 定期更新到最新版以获得性能优化

---

## 九、参考来源

本文档基于以下权威来源整理（按引用频次）：

### 9.1 官方来源

- 🌐 **Tampermonkey 官网**：https://www.tampermonkey.net/
- 📜 **更新日志**：https://www.tampermonkey.net/changelog.php
- 📚 **元数据文档**：https://www.tampermonkey.net/documentation.php
- ❓ **FAQ**：https://www.tampermonkey.net/faq.php
- ❓ **FAQ Q209（userScripts 权限）**：https://www.tampermonkey.net/faq.php?q=Q209
- ❓ **FAQ Q105（安装）**：https://www.tampermonkey.net/faq.php?q=Q105
- ❓ **FAQ Q405 / Q408（脚本管理）**：https://www.tampermonkey.net/faq.php?q=Q405
- 🔌 **Chrome 版本下载页**：https://www.tampermonkey.net/index.php?browser=chrome

### 9.2 脚本仓库

- 🛢️ **Greasy Fork（英文）**：https://greasyfork.org/
- 🇨🇳 **Greasy Fork（中文）**：https://greasyfork.org/zh-CN
- 📋 **TOP 脚本列表**：https://greasyfork.org/en/scripts?sort=total_installs
- 🔍 **高级搜索**：https://greasyfork.org/en/search
- 📂 **OpenUserJS**：https://openuserjs.org/
- 🐱 **ScriptCat 脚本猫**：https://scriptcat.org/

### 9.3 元数据规范参考

- 📖 **Violentmonkey API 文档**：https://violentmonkey.github.io/api/metadata-block/
- 📖 **Greasespot Wiki - Metadata Block**：https://wiki.greasespot.net/Metadata_Block

### 9.4 浏览器扩展商店

- 🌐 **Chrome Web Store**：https://chromewebstore.google.com/
- 🦊 **Firefox Add-ons**：https://addons.mozilla.org/
- 🟦 **Edge Add-ons**：https://microsoftedge.microsoft.com/addons

---

> ⚠️ **免责声明**：本教程仅供学习交流使用，部分脚本可能涉及版权或违反目标网站使用条款，请用户自行评估风险并承担相应责任。
