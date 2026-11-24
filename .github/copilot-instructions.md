# VS Code Copilot Instructions

## Project Overview

Visual Studio Code is built with a layered architecture using TypeScript, web APIs and Electron, combining web technologies with native app capabilities. The codebase is organized into key architectural layers:

### Root Folders
- `src/`: Main TypeScript source code with unit tests in `src/vs/*/test/` folders
- `build/`: Build scripts and CI/CD tools
- `extensions/`: Built-in extensions that ship with VS Code
- `test/`: Integration tests and test infrastructure
- `scripts/`: Development and build scripts
- `resources/`: Static resources (icons, themes, etc.)
- `out/`: Compiled JavaScript output (generated during build)

### Core Architecture (`src/` folder)
- `src/vs/base/` - Foundation utilities and cross-platform abstractions
- `src/vs/platform/` - Platform services and dependency injection infrastructure
- `src/vs/editor/` - Text editor implementation with language services, syntax highlighting, and editing features
- `src/vs/workbench/` - Main application workbench for web and desktop
  - `workbench/browser/` - Core workbench UI components (parts, layout, actions)
  - `workbench/services/` - Service implementations
  - `workbench/contrib/` - Feature contributions (git, debug, search, terminal, etc.)
  - `workbench/api/` - Extension host and VS Code API implementation
- `src/vs/code/` - Electron main process specific implementation
- `src/vs/server/` - Server specific implementation
- `src/vs/sessions/` - Agent sessions window, a dedicated workbench layer for agentic workflows (sits alongside `vs/workbench`, may import from it but not vice versa)

The core architecture follows these principles:
- **Layered architecture** - from `base`, `platform`, `editor`, to `workbench`
- **Dependency injection** - Services are injected through constructor parameters
    - If non-service parameters are needed, they need to come before the service parameters
- **Contribution model** - Features contribute to registries and extension points
- **Cross-platform compatibility** - Abstractions separate platform-specific code

### Built-in Extensions (`extensions/` folder)
The `extensions/` directory contains first-party extensions that ship with VS Code:
- **Language support** - `typescript-language-features/`, `html-language-features/`, `css-language-features/`, etc.
- **Core features** - `git/`, `debug-auto-launch/`, `emmet/`, `markdown-language-features/`
- **Themes** - `theme-*` folders for default color themes
- **Development tools** - `extension-editing/`, `vscode-api-tests/`

Each extension follows the standard VS Code extension structure with `package.json`, TypeScript sources, and contribution points to extend the workbench through the Extension API.

### Finding Related Code
1. **Semantic search first**: Use file search for general concepts
2. **Grep for exact strings**: Use grep for error messages or specific function names
3. **Follow imports**: Check what files import the problematic module
4. **Check test files**: Often reveal usage patterns and expected behavior

## 语言约定（Hestia 分支）

本分支由中文团队维护。以下规则覆盖上游「一律英文」的默认约定，适用于所有改动，与改动落在哪个文件无关。

**新增内容用中文**

- 新增文档：`PATCHES.md`、`docs/**`、扩展 README。
- 新增代码注释。保留 ASCII 标记以便检索，例如 `// Hestia：……`；若属于产品分歧，同时在 `PATCHES.md` 登记。
- 提交信息的正文。主题行保持英文，便于与上游历史一起 `git log --grep`。
- 中文段落不做固定宽度硬换行，一个自然段占一行，保证 diff 可读。

**必须英文（硬约束，不是偏好）**

- 所有经 `nls.localize()` / `localize2()` 外部化的用户可见字符串。翻译流水线从源码英文提取词条，生成 zh-cn、ja、de 等 12 种语言；写成中文会让 zh-cn 变成源语言，其他语言全部失去源文本。中文界面应通过「内置 zh-cn 语言包 + 默认 locale」实现，绝不通过改写源码文本实现。
- 标识符：setting id、command id、配置键、API 名、context key。这些会被外部引用，翻译等于破坏性变更。
- 被测试断言、构建脚本消费或 ESLint 规则匹配的字符串。
- 不翻译上游已有注释。只在本 fork 新增或改写的行使用中文；翻译上游正文会让每次同步产生与功能无关的冲突。

**编码**

- 提交进仓库的内容为 UTF-8 无 BOM、LF 换行。Windows 工作区出现 CRLF 是本仓库 `text=auto` 与 `core.autocrlf` 的既有行为；必须避免的是 CRLF 或 BOM 进入索引。可用 `git ls-files --eol` 核对。

同样的规则与已登记的分歧见 `PATCHES.md`。

## Validating TypeScript changes

Choose validation based on the scope and risk of the change. Large-scale builds and typechecking can be slow, and consume significant resources, so minimize their use. Prefer existing editor or watch-task diagnostics and the smallest targeted tests that cover the changed behavior. Do not start build or watch tasks, run broad type checks, or make type checking a prerequisite for targeted tests solely as a completion ritual.

When running in a VS Code editor window with a workspace folder, use the VS Code task tools for build and watch workflows: inspect the existing task output first, and run an existing task instead of invoking its equivalent shell command. Do not start a duplicate build or watch process when the workspace task already provides current diagnostics. Agents window chats and isolated worktree sessions may not have access to the editor's workspace tasks; use the repository commands directly in those contexts.

Run a targeted type check or build when you are not fully confident in the change, and the change is broad or cross-cutting, it affects build or type configuration, or another validation step reports a compilation problem. When task tools are unavailable or no suitable task exists, useful commands include:

- `pnpm run typecheck-client` for the main sources under `src/`
- `pnpm run gulp compile-extensions` for built-in extensions
- `pnpm run typecheck` from the `build` folder for build tooling

Development compile tasks already type-check their inputs. Do not run `pnpm run typecheck-client` immediately before `pnpm run compile` or `pnpm run compile-client`; choose the command that covers the required validation. When tests only need fresh client output, use `pnpm run build-fast --client-only`. It supports cold builds and file-granular incremental updates without starting a watcher or building extensions. Use `pnpm run build-fast` when extension and Copilot outputs are also needed. The client and extension JavaScript lanes transpile rather than type-check; extension media retains its existing type checks. Run explicit type checks when the change warrants them. Reserve `pnpm run transpile-client` for an intentional clean client rebuild.

Reuse successful validation while its relevant inputs remain unchanged, including when moving into review, commit, or PR creation. Batch related test selectors in one invocation. Prefer `pnpm run oxlint <changed files>` for targeted linting. The repository installs dependencies with pnpm (the `packageManager` field of `package.json` pins the version); never install them with npm or yarn. Install dependencies only when dependency inputs change or validation identifies missing dependencies; run `pnpm install` for a needed refresh, which takes a few seconds when nothing changed. Never share or symlink `node_modules` between worktrees.

Use `scripts/test.sh` (or `scripts\test.bat` on Windows) for unit tests and `scripts/test-integration.sh` (or `scripts\test-integration.bat` on Windows) for integration tests. Add a targeted selector such as `--grep` whenever possible. Run `pnpm run valid-layers-check` only when a change may affect module layering.

## Coding Guidelines

### Markdown

- Do not hard-wrap Markdown prose to a fixed line width. Keep each paragraph on one source line and let the editor or renderer wrap it naturally. Preserve structural line breaks in lists, tables, code blocks, and intentional Markdown hard breaks.

### Indentation

We use tabs, not spaces.

### Naming Conventions

- Use PascalCase for `type` names
- Use PascalCase for `enum` values
- Use camelCase for `function` and `method` names
- Use camelCase for `property` names and `local variables`
- Use whole words in names when possible

### Types

- Do not export `types` or `functions` unless you need to share it across multiple components
- Do not introduce new `types` or `values` to the global namespace

### Comments

- Use JSDoc style comments for `functions`, `interfaces`, `enums`, and `classes`

### Strings

- Use "double quotes" for strings shown to the user that need to be externalized (localized)
- Use 'single quotes' otherwise
- All strings visible to the user need to be externalized using the `vs/nls` module
- Externalized strings must not use string concatenation. Use placeholders instead (`{0}`).

### UI labels
- Use title-style capitalization for command labels, buttons and menu items (each word is capitalized).
- Don't capitalize prepositions of four or fewer letters unless it's the first or last word (e.g. "in", "with", "for").

### Designing UI
- When creating, editing, or reviewing any visual surface, reason in **design terms, not pixels**: name the **feeling** (Calm, Focused, Consistent, Delightful), find the **principle** it breaks, then reach for the **move** (token/tier/ramp) that restores it. Describe a bug by its role/tier/ramp (e.g. "this overlay is rounded at the control tier"), not its number.
- See the [`design-philosophy` skill](skills/design-philosophy/SKILL.md) for the full Values→Principles→Moves vocabulary, worked examples, and feedback guidance, and [design-tokens.instructions.md](instructions/design-tokens.instructions.md) for the token reference.

### Style

- Use arrow functions `=>` over anonymous function expressions
- Only surround arrow function parameters when necessary. For example, `(x) => x + x` is wrong but the following are correct:

```typescript
x => x + x
(x, y) => x + y
<T>(x: T, y: T) => x === y
```

- Always surround loop and conditional bodies with curly braces
- Open curly braces always go on the same line as whatever necessitates them
- Parenthesized constructs should have no surrounding whitespace. A single space follows commas, colons, and semicolons in those constructs. For example:

```typescript
for (let i = 0, n = str.length; i < 10; i++) {
    if (x < 10) {
        foo();
    }
}
function f(x: number, y: string): void { }
```

- Whenever possible, in top-level scopes, use `export function x(…) {…}` instead of `export const x = (…) => {…}`. One advantage of using the `function` keyword is that the stack trace shows a good name when debugging.

### Code Quality

- All files must include Microsoft copyright header
- Prefer `async` and `await` over `Promise` and `then` calls
- All user facing messages must be localized using the applicable localization framework (for example `nls.localize()` method)
- Don't add tests to the wrong test suite (e.g., adding to end of file instead of inside relevant suite)
- Look for existing test patterns before creating new structures
- Use `describe` and `test` consistently with existing patterns
- Prefer regex capture groups with names over numbered capture groups.
- If you create any temporary new files, scripts, or helper files for iteration, clean up these files by removing them at the end of the task
- Never duplicate imports. Always reuse existing imports if they are present.
- When removing an import, do not leave behind blank lines where the import was. Ensure the surrounding code remains compact.
- Do not use `any` or `unknown` as the type for variables, parameters, or return values unless absolutely necessary. If they need type annotations, they should have proper types or interfaces defined.
- When adding file watching, prefer correlated file watchers (via fileService.createWatcher) to shared ones.
- When adding tooltips to UI elements, prefer the use of IHoverService service.
- Do not duplicate code. Always look for existing utility functions, helpers, or patterns in the codebase before implementing new functionality. Reuse and extend existing code whenever possible.
- You MUST deal with disposables by registering them immediately after creation for later disposal. Use helpers such as `DisposableStore`, `MutableDisposable` or `DisposableMap`. Do NOT register a disposable to the containing class if the object is created within a method that is called repeatedly to avoid leaks. Instead, return an `IDisposable` from such method and let the caller register it.
- You MUST NOT use storage keys of another component only to make changes to that component. You MUST come up with proper API to change another component.
- Use `IEditorService` to open editors instead of `IEditorGroupsService.activeGroup.openEditor` to ensure that the editor opening logic is properly followed and to avoid bypassing important features such as `revealIfOpened` or `preserveFocus`.
- Avoid using `bind()`, `call()` and `apply()` solely to control `this` or partially apply arguments; prefer arrow functions or closures to capture the necessary context, and use these methods only when required by an API or interoperability.
- Avoid using events to drive control flow between components. Instead, prefer direct method calls or service interactions to ensure clearer dependencies and easier traceability of logic. Events should be reserved for broadcasting state changes or notifications rather than orchestrating behavior across components.
- Service dependencies MUST be declared in constructors and MUST NOT be accessed through the `IInstantiationService` at any other point in time.

## Public Content

This repository is public, and so is everything written in it: pull requests, issues, comments, reviews, commit messages, code, tests and documentation. The instructions above cover the code; this section covers the text around it.

- Don't include confidential information: credentials or tokens, personal or customer data, links to internal-only tools and dashboards, or data from internal sources such as telemetry queries, experiment scorecards and cost reports.
- Anything derived from internal data is confidential too: counts of users, devices, sessions or requests; rates, shares and deltas; statistical results; cost, token usage and capacity figures; and screenshots of queries or dashboards. Telemetry event and property names, and the shape of a query, are fine.
- When internal data informs a change, describe the conclusion without numbers, for example "the error is common on Windows".
- A pull request, commit or issue that ships, changes, graduates or removes an experiment describes only the code change. Don't explain the change with the experiment, its rollout or its results, even without numbers, such as "the experiment showed no regression" or "the experiment is shipping". Link its tracking issue instead.
- Numbers you measured yourself, for example in a local run, a test or a benchmark, are fine as long as they aren't cost figures. So are planned settings such as a rollout percentage.
- Check the text of a pull request, issue, comment or commit message against these rules before you publish it. Commit messages can't be changed after they are merged.

## Learnings
- Minimize the amount of assertions in tests. Prefer one snapshot-style `assert.deepStrictEqual` over multiple precise assertions, as they are much more difficult to understand and to update.
- Do not stub a global object (e.g. `(mainWindow as any).ResizeObserver = ...`) or use `any` casts to install fakes in tests. Instead, make the dependency injectable: add an optional constructor parameter on the production class that defaults to the real implementation (e.g. `dom.DisposableResizeObserver`'s `resizeObserverCtor` option, which defaults to `targetWindow.ResizeObserver`), and have the test pass a fake that implements the real interface.
