# AGENTS.md

本文件供在 `Thief` 仓库工作的 Coding Agent 使用。它是维护约定，不是产品介绍。

## 开始工作前

1. 先读 `docs/PROJECT.md`，了解窗口、路由、数据键、IPC 和已知限制。
2. 再读 `package.json` 和相关源码；不要只根据 README 推断实现。
3. 用户的问题涉及运行、构建或平台差异时，先确认当前环境和未验证项，不要假装构建成功。

## 项目边界

- 技术栈：Electron 45 + Vue 2 + Webpack 5 + Babel 7 + Dart Sass + lowdb；Node.js `>=22.12.0`。
- 主进程入口：`src/main/index.js`，集中管理窗口、托盘、快捷键、IPC 和业务调度。
- 渲染层：`src/renderer/`，每个窗口通过 hash 路由映射到 `components/`。
- 公共业务：`src/main/utils/`；小说、股票、广告、数据库和系统信息都在这里。
- 本地数据：`app.getPath("userData")/thief_data.json`，默认值和字段见 `docs/PROJECT.md`。
- 渲染进程桥接：`@electron/remote`；主进程在 `createWindow()` 中初始化并逐窗口启用。
- 文档：面向人的完整说明在 `docs/PROJECT.md`；面向 Agent 的维护规则在本文件。

## 操作规则

- 默认最小改动，只修复直接问题；不做无关重构、依赖升级或目录迁移。
- 改业务前先找全调用方，尤其是 `ipcMain`/`ipcRenderer`、`db.get`/`db.set`、窗口创建函数和设置页。
- 修改配置键时，同时更新 `src/main/utils/db.js`、`src/renderer/components/setting.vue` 和 `docs/PROJECT.md`。
- 修改窗口、路由、IPC 或快捷键时，同步更新 `docs/PROJECT.md` 对应表格。
- 不要删除或重命名用户已有的约定、脚本和数据字段，除非明确要求。
- 新增依赖不是默认选项；先使用现有依赖或平台能力。
- 不要把 README 的营销文案当作功能事实，以源码为准。

## 构建与验证

```bash
npm ci
npm run dev
npm run pack
npm run build:dir
npm run build:win64
```

- 仓库没有自动化测试、Lint 或类型检查，代码改动必须做对应场景的手动验证。
- 仓库已有 `package-lock.json`；优先用 `npm ci` 复现依赖。当前 Node.js 22 环境已经验证安装、开发模式、生产编译和本机 arm64 目录打包。
- `npm run build:mac` 固定构建 x64；当前 arm64 机器验证本机产物用 `npm run build:dir`。
- 本机有 Apple Development 证书时，electron-builder 可能停在交互式签名。仅本地未签名验证使用 `CSC_IDENTITY_AUTO_DISCOVERY=false npm run build:dir`。
- Electron 二进制下载慢时，可设置 `ELECTRON_MIRROR=https://npmmirror.com/mirrors/electron/`。
- 只有文档改动时，至少执行 `git diff --check` 并检查相对链接。
- 发布前验证小说、股票、网页、视频、PDF、老板键、快捷键，以及 macOS 三种展示模式。

## 已知高风险点

- `nodeIntegration`、`remote` 和网页窗口的 `webviewTag` 均开启，且可加载任意 URL；不要扩大不可信内容的执行范围。
- Electron 45 默认 `contextIsolation: true`。当前代码在 `createWindow()` 中显式关闭它以维持旧的 `require`/`remote` 行为；新增窗口必须走 `createWindow()`，不要绕过 remote 初始化。
- 渲染进程使用 `@electron/remote` 时必须用 `require('@electron/remote')`；默认 `import` 会得到 `undefined`，导致 `getGlobal` 报错。
- Electron 45 没有 `webFrame.setLayoutZoomLevelLimits`；不要恢复该调用，缩放限制只需保留 `setVisualZoomLevelLimits`。
- 开发模式依赖 `webpack-hot-middleware`，Webpack 配置必须保留 `HotModuleReplacementPlugin`，否则 dev-client 会在 Vue 挂载前抛错。
- `AutoPage()` 的开关状态和定时器语义相反，首次触发不会开始翻页；修改时先验证菜单勾选状态与实际行为。
- `createKey()` 遇到一个格式不正确的快捷键会提前返回，可能跳过后续快捷键和退出快捷键注册。
- `createWindownDesktop()` 使用 `arr_wh.length` 判断位置数组是否有效，只保存过宽高时可能得到 `NaN` 坐标。
- 视频整文件读入内存，PDF 为每页创建 Canvas；不要用大文件验证内存行为。
- 股票和广告接口使用 HTTP，网络失败时多数情况下静默处理；不要用“请求成功”推断 UI 已更新。
- 生产构建当前有 7 条 Vue 样式 `style0` 警告和两条 `electron-debug` 动态 `require` 警告；它们不阻塞运行，但不要把“有警告”当成构建失败，也不要在无关改动中顺手重构样式链。

## 文档维护

以下变化必须同步文档：

- 设置字段、默认值、存储位置或数据格式：更新 `docs/PROJECT.md` 的“本地数据”。
- 窗口尺寸、路由、平台模式或启动流程：更新“进程与窗口架构”。
- IPC、快捷键或菜单行为：更新“IPC 与快捷键”。
- 依赖、脚本、外部接口或安全边界：更新对应章节和本文件的“已知高风险点”。

修改文档后，确认 README、`docs/PROJECT.md` 和本文件中没有互相矛盾的描述。
