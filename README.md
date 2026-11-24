# Hestia

<p align="center">
  <img alt="Hestia" src="resources/win32/code_150x150.png" width="150">
</p>

Hestia 是一款基于 [Visual Studio Code 开源代码](https://github.com/microsoft/vscode)重新构建的桌面代码编辑器，面向 AI 辅助编程场景。它使用上游的 Code OSS 源码，替换产品标识、安装器与品牌资源，并逐步加入自有的 AI 能力。

**Hestia 不是 Microsoft 产品，与 Microsoft 无隶属或背书关系。** 上游版权与许可声明见文末。

## 项目状态

当前处于里程碑 1：完成产品化底座，尚未加入 AI 功能。

已在 Windows x64 上实际验证：

| 验证项 | 命令 | 结果 |
| --- | --- | --- |
| 依赖安装 | `pnpm install` | 通过 |
| 客户端编译 | `pnpm run compile-client` | 通过 |
| 开发启动 | `.\scripts\code.bat` | 通过，窗口标题为 `Welcome - Hestia Dev` |
| 应用打包 | `pnpm run gulp vscode-win32-x64` | 通过，产出 `Hestia.exe` |
| 安装包 | `pnpm run gulp vscode-win32-x64-user-setup` | 通过，产出 `VSCodeSetup.exe` |
| 静默安装 / 卸载 | `/VERYSILENT` | 通过，安装后可正常启动 |

当前不提供：扩展市场、自动更新、代码签名、账号体系。以上均在后续里程碑中规划。

## 与上游的关系

- 基线为 `microsoft/vscode` 发布标签 `1.147.0`，不是移动中的 `main`。
- 所有与上游的有意分歧记录在 [PATCHES.md](PATCHES.md)，同步新版本前应先读该文件。
- 代码风格与语言约定见 [.github/copilot-instructions.md](.github/copilot-instructions.md)。

上游 `README.md` 已由本文件替换，原文可从 git 历史中取回。

## 分支模型

| 分支 | 用途 |
| --- | --- |
| `main` | `microsoft/vscode` 的只读镜像，禁止提交 |
| `hestia` | 产品主线，所有 Hestia 改动落在这里 |
| `sync/<tag>` | 临时分支，用于把新的上游标签合并进 `hestia` |

## 构建

### 环境要求

| 项 | 版本 | 说明 |
| --- | --- | --- |
| Node.js | **24.22.1** | 必须与 [.nvmrc](.nvmrc) 完全一致，否则原生模块编译会失败 |
| pnpm | 12.16.0 | 由 `package.json` 的 `packageManager` 固定 |
| Python | 3.8+ | node-gyp 需要 |
| Visual C++ 生成工具 | VS 2019 或更高 | 安装「使用 C++ 的桌面开发」工作负载；VS 2026 已可用 |
| Inno Setup | 6.x | 仅打包安装包时需要 |

安装 pnpm 时可显式指定版本：`npm i -g pnpm@12.16.0`。

### 首次构建

```bat
nvm use 24.22.1
pnpm install
pnpm run compile-client
```

### 开发运行

```bat
.\scripts\code.bat
```

首次运行会下载 Electron。若网络受限，先设置镜像：

```bat
set ELECTRON_MIRROR=https://npmmirror.com/mirrors/electron/
```

### 打包

`patch-win32-dependencies` 这一步需要 `signtool.exe` 在 `PATH` 中（来自 Windows SDK），否则会报 `spawn signtool.exe ENOENT`。

```bat
set PATH=C:\Program Files (x86)\Windows Kits\10\bin\10.0.22621.0\x64;%PATH%
pnpm run gulp vscode-win32-x64
```

生成安装包前必须先产出 `tools` 目录，否则 Inno Setup 会因缺少 `tools\*` 而中止：

```bat
pnpm run gulp vscode-win32-x64-inno-updater
pnpm run gulp vscode-win32-x64-user-setup
```

产物：

- 应用目录：`..\VSCode-win32-x64`
- 安装包：`.build\win32-x64\user-setup\VSCodeSetup.exe`

## 已知限制

- 安装包未做代码签名，安装时会出现 SmartScreen 提示。
- 无自动更新，升级需手动安装新版本。
- 无扩展市场，仅内置固定扩展集。
- 受网络限制，本地仓库目前是浅克隆，尚不能稳定拉取上游完整历史。

## 许可与致谢

本项目基于 [microsoft/vscode](https://github.com/microsoft/vscode) 派生，遵循 [MIT 许可](LICENSE.txt)。

上游代码版权归 Microsoft Corporation 所有：

```
Copyright (c) 2015 - present Microsoft Corporation
```

Hestia 自身的改动同样以 MIT 许可发布。使用本产品时请同时遵守上游许可条款。
