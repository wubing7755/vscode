# Hestia 分支补丁记录

Hestia 是基于 `microsoft/vscode` 发布标签重新命名的 Code OSS 发行版。本文件记录**所有**与上游的有意分歧，使「同步一个新上游标签」成为机械工作，而不是考古。

保持本文件有用的规则：

- 每个逻辑改动一条记录，并写明涉及的文件。
- 变更行为时，在同一个提交里更新本文件。
- 改动被废弃时删除该条，而不是留下过期说明。

## 语言约定

团队以中文为主，因此：

- **新增的文档**（本文件、`docs/**`、扩展 README）用中文撰写。
- **新增的代码注释**用中文撰写。保留 ASCII 前缀 `Hestia` 便于检索，统一写法为 `// Hestia：……`。
- **用户可见字符串必须保持英文**，并通过 `nls.localize()` 外部化。原因：`build/lib/i18n.ts` 从源码英文原文提取词条，翻译成 zh-cn、ja、de 等 12 种语言。把中文写进 `localize()` 会让 zh-cn 变成源语言，其他语言全部失去源文本。界面默认中文应通过「内置 zh-cn 语言包 + 默认 locale」实现，这属于产品配置，不是改源码文本。
- **标识符必须保持 ASCII**：setting id、command id、配置键、API 名称会被外部引用，翻译它们等于破坏性变更。
- **不翻译上游已有的注释**。只在我们新增或改写的行里使用中文；翻译既有注释会让每次上游同步产生大量与功能无关的冲突。

仓库内提交内容统一 UTF-8 无 BOM、LF 换行；Windows 工作区 checkout 成 CRLF 属于本仓库既有的 `text=auto` + `core.autocrlf` 行为，不算问题。需要避免的是**提交进仓库的内容**出现 CRLF 或 BOM。

## 分支模型

| 分支 | 用途 |
| --- | --- |
| `main` | `microsoft/vscode` `main` 的只读镜像，禁止提交 |
| `hestia` | 产品主线，所有 Hestia 改动落在这里 |
| `sync/<tag>` | 临时分支，负责把新的上游标签合并进 `hestia` |

`git rerere` 已开启，保持开启。

## 补丁清单

### 1. 产品标识（`product.json`）

- 产品改名为 `Hestia`（`nameShort`/`nameLong`），`applicationName` 为 `hestia`，数据目录改为 `.hestia` / `.hestia-shared`，`win32DirName`/`win32NameVersion` 为 `Hestia`，`win32RegValueName` 为 `Hestia`，`win32AppUserModelId` 为 `HestiaLab.Hestia`，`win32ShellNameShort` 为 `H&estia`，tunnel/server 应用名同步改名，`urlProtocol` 为 `hestia`。
- 为 Hestia 分配独立的 Inno Setup 应用 ID（`win32x64AppId`、`win32x64UserAppId`），使 Hestia 可以与 VS Code、Code OSS 并存安装。
- `licenseUrl`/`serverLicenseUrl`/`reportIssueUrl` 指向本 fork。
- 删除 `defaultChatAgent`、`trustedExtensionAuthAccess`、`builtInExtensionsEnabledWithAutoUpdates`、`voiceWsUrl`、`webviewContentExternalBaseUrlTemplate`。这些字段都引用了微软服务、GitHub Copilot 扩展或微软 CDN。

原因：重新命名的产品不能继承其他厂商的身份、遥测端点和 CDN。刻意不提供 `webviewContentExternalBaseUrlTemplate`，让 workbench 回退到本地 `vscode-webview://` 端点，而不是 `*.vscode-cdn.net`。

### 2. Windows 品牌资源（`resources/win32/**`）

- `code.ico`、`default.ico`、`sessions.ico`、`code_150x150.png`、`code_70x70.png`。
- Inno Setup 向导位图（`inno-big-*.bmp`、`inno-small-*.bmp`）。
- `VisualElementsManifest.xml`：`ShortDisplayName` 改为 `Hestia`。
- `appx/AppxManifest.xml`：`PublisherDisplayName` 改为 `HestiaLab`。

沿用上游文件名，因此打包脚本与 `code.iss` 无需改动，未来上游新增资源时也能直接覆盖生效。

### 3. 安装器文案与厂商元数据

- `build/win32/code.iss`：`AppPublisher` 改为 `HestiaLab`，发布/支持/更新 URL 指向本 fork，「请改用 System 安装包」的提示不再链接到 `code.visualstudio.com`。
- `build/gulpfile.vscode.ts`、`build/gulpfile.reh.ts`：原生依赖的 Windows 二进制元数据中 `CompanyName` 改为 `HestiaLab`，`LegalCopyright` 改为 `Copyright (C) 2026 HestiaLab`。

### 4. 产品构建不再打包 Copilot

- `build/gulpfile.vscode.ts`：仅当 `product.defaultChatAgent` 存在时，才执行 `prepareCopilotRipgrepShimTask` 与 `compileCopilotExtensionBuildTask`。

原因：Hestia 不配置默认 Chat Agent，Copilot VSIX 及其原生运行时 shim 没有消费者。构建它们会让打包依赖下载 `@github/copilot-*` 压缩包，而随产品分发则等于转发了 Hestia 并不使用的专有扩展。

### 5. workbench 在缺少默认 Chat Agent 时不再崩溃

以下位置原本都假定 `product.defaultChatAgent` 一定存在；Hestia 删除了该字段，因此逐一改为降级处理。

- `src/vs/workbench/contrib/welcomeOnboarding/browser/onboardingVariationA.ts`：删除模块级的 `assertDefined`（它把一处产品配置差异变成加载 workbench 包时的未捕获异常）；`show()` 在未配置 Agent 时直接返回，避免内部命令渲染出半初始化的向导。
- `src/vs/workbench/contrib/welcomeOnboarding/test/browser/onboardingVariationA.test.ts`：跳过该套件（原因见下）。
- `src/vs/workbench/contrib/welcomeGettingStarted/browser/startupPage.ts`：`tryShowOnboarding` 在未配置默认 Chat Agent 时提前返回，不再展示以 AI 为中心的引导向导。
- `src/vs/workbench/services/accounts/browser/defaultAccount.ts`：`toDefaultAccountConfig` 接受 `undefined` 并返回休眠配置，而不是在服务构造期解引用空值。

后续排查：其他与 Copilot 相邻的界面可能存在同类假设。任何提到 `defaultChatAgent` 的新报错（`Not Found`、`Cannot read properties of undefined`）都归入本节。

对上游测试的影响：`onboardingVariationA.test.ts` 验证的是 AI 登录向导，缺少默认 Chat Agent 时该界面根本无法渲染，因此本 fork 跳过该套件。若将来 Hestia 配置了自己的 Agent，应重新启用。

### 6. 文档

- `README.md`：整篇替换为 Hestia 说明，含项目状态、与上游关系、分支模型、构建步骤、已知限制与许可致谢。上游原文保留在 fork 之前的 git 历史中。保留了 Microsoft Corporation 的 MIT 版权声明，并显式声明 Hestia 与 Microsoft 无隶属关系。
- `.github/copilot-instructions.md`：新增「语言约定（Hestia 分支）」一节。这是上游文件，改动位置在「Project Overview」与「Validating TypeScript changes」之间，上游若同时改这两节附近可能产生冲突。

原因：这两份文件是对外与对内的事实入口。若继续沿用上游内容，会让人误以为本项目就是 VS Code，或误以为可以用 `npm` 而不是 `pnpm` 构建。

## 已知缺口

- 打包任务仍会把 CLI 的 `copilot-shim` 复制到 `bin/`。无 Chat Agent 时它是惰性的，但应从打包任务中移除。
- 词典与扩展推荐仍使用上游默认 URL，公开发布前需要复核。
- 欢迎页里读取产品名的字符串来自 `product.json`，会自动更新；其他地方硬编码的 "Code OSS" 属于上游缺陷，不是 Hestia 补丁。
