# Thief 项目技术文档

> 本文描述当前工作区中已经实现的代码行为。仓库基线为 `codex/develop` 的提交 `d4130c4`，并包含 2026-10-30 的构建链升级。它不是产品宣传文案，也不替官网文档补充未实现能力。

## 1. 项目概览

Thief 是一个基于 Electron + Vue 2 的跨平台桌面应用，核心形态是“常驻托盘，把小说、股票、网页、视频或 PDF 内容以低打扰方式显示在工作窗口之上”。它同时提供桌面悬浮窗、macOS 菜单栏标题和 macOS Touch Bar 三种展示形态。

主要能力：

| 能力 | 当前实现 |
| --- | --- |
| 小说摸鱼 | 读取本地 TXT，按字符数分页，支持 UTF-8/GB2312 解码、上一页/下一页、自动翻页、关键字搜索、鼠标翻页 |
| 股票摸鱼 | 按逗号分隔股票代码，请求新浪行情接口，显示现价和涨跌幅；股票模式下每 5 秒刷新一次 |
| 网页摸鱼 | 在 Electron `webview` 中打开用户输入的 URL |
| 视频摸鱼 | 读取本地视频文件并通过 Blob URL 播放 |
| PDF 摸鱼 | 使用 `pdfjs-dist` 在 Canvas 中渲染本地 PDF 的所有页面 |
| 老板键 | 将当前展示内容切换为配置的“摸鱼文字”或广告文本；再次触发时切换悬浮窗/子窗口的显示状态 |
| 隐蔽显示 | 透明、无边框、置顶、跳过任务栏；网页、视频、PDF 窗口支持透明度调节 |
| 快捷键 | 全局快捷键控制翻页、老板键、自动翻页；退出快捷键为 `CommandOrControl+Alt+X` |

## 2. 技术栈

| 分类 | 技术 |
| --- | --- |
| 桌面运行时 | Electron 45.1.0 |
| 前端 | Vue 2.7、Vue Router 3、Vuex 3、Element UI 2 |
| 构建 | Webpack 5、Babel 7、Dart Sass、electron-vue 脚手架 |
| 本地存储 | lowdb 1 + JSON 文件 |
| 网络请求 | `request`、`request-promise` |
| 文本与文档 | `iconv-lite`、`pdfjs-dist` |
| 渲染进程桥接 | `@electron/remote` 2.1.3 |
| 打包 | electron-builder 26 |
| Node.js | `>=22.12.0` |

项目没有自动化测试、ESLint 配置或提交钩子。仓库现在包含 `package-lock.json`（lockfile v3），依赖安装和构建版本可以被锁定。

已在 macOS 15.7.9 arm64、Node.js `22.23.3`、npm `10.9.9` 下验证：

- `npm install` 可完成安装。
- `npm run pack` 可生成 `dist/electron`。
- `npm run build:dir` 可生成 `build/mac-arm64/thief.app`。
- 打包后的应用可以启动，Vue 页面能够挂载，`@electron/remote` 能读取主进程全局文本，并能在隔离的 `userData` 下生成 `thief_data.json`。
- `npm run dev` 可以启动 Webpack Dev Server 和 Electron，渲染页面能够挂载。

Windows、Linux 以及 macOS 的 x64 安装包尚未在当前机器上验证。

## 3. 目录结构

```text
Thief/
├── .electron-vue/                 # electron-vue 开发、Webpack 和构建脚本
├── build/icons/                   # 应用图标
├── images/                        # README 使用的图片
├── static/                        # 运行时托盘图标
├── src/
│   ├── main/
│   │   ├── index.js               # 主进程入口：窗口、托盘、菜单、快捷键、IPC
│   │   ├── index.dev.js           # 开发模式注入 electron-debug 和 Vue DevTools
│   │   └── utils/
│   │       ├── ad.js              # 广告文本请求
│   │       ├── book.js            # TXT 读取、解码、分页、搜索
│   │       ├── db.js              # lowdb 初始化与读写
│   │       ├── osUtil.js          # 开机时长和内存信息
│   │       └── stock.js           # 新浪股票行情请求
│   ├── renderer/
│   │   ├── components/
│   │   │   ├── desktop.vue        # 小说/股票展示悬浮窗
│   │   │   ├── pdf.vue            # PDF 窗口
│   │   │   ├── setting.vue        # 设置窗口
│   │   │   ├── so.vue             # 小说全文搜索窗口
│   │   │   ├── video.vue          # 视频窗口
│   │   │   └── web.vue            # 网页窗口
│   │   ├── router/index.js        # 每个窗口对应的 hash 路由
│   │   ├── store/                 # Vuex；当前业务未使用 Counter 模块
│   │   ├── utils/dialog.js        # 文件选择对话框
│   │   ├── utils/key.js           # 快捷键录入辅助类，当前设置页未直接使用该类
│   │   ├── App.vue
│   │   └── main.js
│   └── index.ejs                  # HTML 模板
├── README.md
└── package.json
```

## 4. 进程与窗口架构

### 4.1 主进程职责

主进程入口是 `src/main/index.js`，集中承担以下职责：

- 创建托盘、托盘菜单和 Touch Bar。
- 创建设置、搜索、小说/股票悬浮窗、网页、视频、PDF 窗口。
- 管理全局快捷键和页面级快捷键消息。
- 读取小说和股票数据，向渲染进程推送展示文本。
- 读写本地配置，保存窗口位置和尺寸。
- 请求广告、股票行情和版本信息。

这不是多服务架构：业务逻辑几乎全部集中在一个约 1250 行的主进程文件中，渲染层主要负责 UI 和用户输入。

Electron 45 默认开启 `contextIsolation`。为了保持旧代码中渲染层直接使用 `require`、`ipcRenderer` 和 `@electron/remote` 的行为，`createWindow()` 会为所有窗口显式设置 `contextIsolation: false`，并通过 `initializeRemote()`/`enableRemote()` 启用 remote 模块。这是兼容性处理，不代表当前安全边界合理。

### 4.2 窗口与路由

| 窗口 | 路由 | 默认尺寸 | 说明 |
| --- | --- | --- | --- |
| 设置窗口 | `#/setting` | 715 × 630 | 小说路径、分页、颜色、快捷键、股票代码、摸鱼文字 |
| 搜索窗口 | `#/so` | 334 × 540 | 搜索 TXT 内容并跳转到对应页 |
| 桌面悬浮窗 | `#/desktop` | 856 × 47 | 小说或股票的透明无边框展示窗 |
| 菜单栏/Touch Bar 展示窗 | `#/desktop` | 88 × 23 | macOS 菜单栏和 Touch Bar 模式的展示窗 |
| 网页窗口 | `#/web` | 478 × 28 | 选择 URL 后扩展到 715 × 500 |
| 视频窗口 | `#/video` | 478 × 28 | 选择文件后扩展到 715 × 500 |
| PDF 窗口 | `#/pdf` | 478 × 28 | 选择文件后扩展到 715 × 500 |

展示窗口设置 `alwaysOnTop`、`skipTaskbar`，macOS 下网页、视频、PDF 窗口使用无边框模式。桌面悬浮窗会保存窗口宽高与位置；关闭窗口时主进程会把自动翻页和鼠标翻页状态重置。

### 4.3 启动流程

`app.on('ready')` 后执行 `init()`：

1. 禁用应用菜单，初始化本地数据。
2. 每次启动把 `auto_page` 和 `is_mouse` 设为关闭状态。
3. macOS 隐藏 Dock 图标；默认进入任务栏模式。若上次使用桌面模式，则创建桌面悬浮窗；若上次使用 Touch Bar 模式，则启动时回落到任务栏模式。
4. Windows/Linux 默认创建桌面悬浮窗。
5. 注册全局快捷键、创建托盘，并调用老板键逻辑初始化展示文本。

### 4.4 两种内容模式

| 配置 | 值 | 行为 |
| --- | --- | --- |
| `display_model` | `1` | 小说模式，翻页读取 TXT，通过托盘标题或悬浮窗展示 |
| `display_model` | `2` | 股票模式；通过菜单切换时请求行情并启动 5 秒轮询，翻页快捷键也会触发刷新 |

### 4.5 三种展示形态

| 配置 | 值 | 平台 | 行为 |
| --- | --- | --- | --- |
| `curr_model` | `1` | macOS | 菜单栏模式，使用托盘标题展示文本 |
| `curr_model` | `2` | 全平台 | 桌面悬浮模式，使用透明置顶窗口展示文本 |
| `curr_model` | `3` | macOS | Touch Bar 模式，Touch Bar 显示文本，悬浮窗可显示内存信息 |

Windows/Linux 只提供桌面悬浮模式。macOS 的三种模式可以在托盘菜单切换；Touch Bar 模式在下次启动时不会保持。

## 5. 核心业务流程

### 5.1 小说分页与展示

`src/main/utils/book.js` 是小说功能的核心：

1. 从 `current_file_path` 读取 TXT。
2. 勾选“乱码”时用 GB2312 解码，否则用 UTF-8 解码。
3. 将换行符替换为配置的 `line_break`，统一全角空格和普通空格。
4. 用 `page_size` 计算总页数，把文本片段交给主进程展示。
5. `current_page` 表示当前页，上一页/下一页会更新本地 JSON。
6. 展示时可附加 `当前页/总页数`，由 `is_display_page` 控制。
7. 搜索窗口使用正则在全文查找关键字，返回命中位置、附近文本和页码；点击结果后跳转。

路径为空、路径不存在或文件读取失败时，返回中文提示文本，不会弹窗中断流程。

### 5.2 股票行情

`src/main/utils/stock.js` 将逗号分隔的代码逐个请求到：

```text
http://hq.sinajs.cn/list=<code>
```

返回格式使用 `,` 分隔，代码取数组第 3、4 项作为昨收价和现价，再计算涨跌幅。多个代码使用 `||` 拼接。股票模式下，自动刷新定时器固定为 5 秒。

该实现依赖新浪接口的响应格式；接口不可用、响应变更或网络失败时，代码会吞掉异常，界面可能保持旧数据。

### 5.3 老板键

老板键有两种触发方式：

- 快捷键或托盘菜单触发 `BossKey(2)`：切换悬浮窗、网页、视频和 PDF 窗口的显示/隐藏。
- 模式切换或初始化触发 `BossKey(1)`：更新展示内容。

任务栏模式和桌面模式会把内容替换为 `moyu_text`；广告接口成功时使用广告文本，并把时间戳写入 `is_ad`，大约 8 小时内复用缓存。接口失败时回落到 `moyu_text`。

### 5.4 网页、视频与 PDF

这三个窗口初始是 478 × 28 的透明条，选择内容后通过各自 IPC 消息扩展到 715 × 500 并居中。窗口聚焦时支持：

| 按键 | 网页 | 视频 | PDF |
| --- | --- | --- | --- |
| `c` | 显示/隐藏输入条 | 显示/隐藏输入条 | 显示/隐藏输入条 |
| `+` / `-` | 调整透明度 | 调整透明度 | 调整透明度 |
| `x` | 关闭窗口 | 关闭窗口 | 关闭窗口 |
| `z` | 刷新页面 | - | - |

视频通过 `fs.readFileSync` 将整个文件读入内存再创建 Blob；PDF 会为每一页创建 Canvas。对大文件需要特别关注内存占用。

## 6. 本地数据

数据通过 lowdb 保存为 JSON：

```text
<Electron app.getPath("userData")>/thief_data.json
```

`src/main/utils/db.js` 每次 `get`/`set` 都会重新执行初始化检查，默认值如下：

| 键 | 默认值 | 用途 |
| --- | --- | --- |
| `current_page` | `1` | 小说当前页 |
| `page_size` | `20` | 每页字符数 |
| `is_english` | `false` | 英文模式标记；当前分页逻辑中未产生实际差异 |
| `line_break` | `" "` | TXT 换行替换文本 |
| `current_file_path` | `""` | 当前 TXT 路径 |
| `bg_color` | `"rgba(0, 0, 0, 0.5)"` | 悬浮窗背景色 |
| `txt_color` | `"#fff"` | 悬浮窗文字颜色 |
| `font_size` | `"14"` | 悬浮窗字号 |
| `second` | `"5"` | 自动翻页间隔 |
| `auto_page` | `"0"` | 自动翻页开关 |
| `key_previous` / `key_next` | `CmdOrCtrl+Alt+,` / `CmdOrCtrl+Alt+.` | 上一页/下一页快捷键 |
| `key_boss` | `CmdOrCtrl+Alt+M` | 老板键 |
| `key_auto` | `CmdOrCtrl+Alt+P` | 自动翻页快捷键 |
| `errCodeChecked` | `false` | TXT 是否按 GB2312 解码 |
| `is_mouse` | `"0"` | 鼠标翻页模式 |
| `is_display_page` | `true` | 是否显示页码 |
| `display_model` | `"1"` | 小说/股票模式 |
| `display_shares_list` | `""` | 逗号分隔的股票代码 |
| `moyu_text` | `"Hello"` | 老板键显示文本 |
| `desktop_wh` | `""` | 悬浮窗宽高，格式 `width,height` |
| `desktop_wz` | `""` | 悬浮窗位置，格式 `x,y` |
| `is_ad` | `0` | 广告缓存时间戳 |
| `curr_model` | macOS `"1"`，其他平台 `"2"` | 展示形态 |

应用启动以及所有窗口关闭时，会重置 `auto_page` 和 `is_mouse`；macOS 关闭所有窗口时还会把 `curr_model` 重置为 `1`。

## 7. IPC 与快捷键

### 7.1 IPC 消息

| 方向 | 名称 | 参数 | 作用 |
| --- | --- | --- | --- |
| 渲染进程 → 主进程 | `bg_text_color` | `ping` | 保存设置后重建托盘和快捷键，并通知展示窗刷新样式 |
| 渲染进程 → 主进程 | `jump_page` | `ping` | 搜索窗口跳转后触发下一页 |
| 渲染进程 → 主进程 | `MouseAction` | `1`/`2`/`3`/`4` | 左键下一页、右键上一页、移出触发老板键 |
| 渲染进程 → 主进程 | `webOpacity` | `-`/`+`/`change`/`exit` | 网页窗口透明度、尺寸和关闭 |
| 渲染进程 → 主进程 | `pdfOpacity` | `-`/`+`/`change`/`exit` | PDF 窗口透明度、尺寸和关闭 |
| 渲染进程 → 主进程 | `videoOpacity` | `-`/`+`/`change`/`exit` | 视频窗口透明度、尺寸和关闭 |
| 主进程 → 展示窗 | `text` | `boss`/`ping` | 更新全局文本并刷新悬浮窗 |
| 主进程 → 展示窗 | `bg_text_color` | `ping` | 设置保存后刷新展示窗 |

### 7.2 快捷键

默认全局快捷键在设置窗口可配置。主进程还固定注册 `CommandOrControl+Alt+X` 用于退出。

网页、视频、PDF 的 `c`、`+`、`-`、`x` 以及网页的 `z` 是窗口内快捷键，不是全局快捷键，只有对应窗口获得焦点时生效。

## 8. 开发与构建

### 8.1 开发模式

```bash
npm ci
npm run dev
```

也可以使用 `npm install` 代替 `npm ci`。开发模式需要 Node.js `>=22.12.0`。

开发脚本会：

1. 在 `127.0.0.1:9080` 启动渲染进程 Webpack Dev Server。
2. 监听并编译主进程。
3. 启动 Electron，并启用 `electron-debug` 和 Vue DevTools 安装逻辑。
4. 渲染进程使用 `webpack-hot-middleware`，因此开发配置必须保留 `HotModuleReplacementPlugin`；删掉它会导致 dev-client 在 Vue 挂载前抛错。

首次安装 Electron 二进制时，如果 GitHub 下载过慢，可以设置镜像：

```bash
ELECTRON_MIRROR=https://npmmirror.com/mirrors/electron/ npm install
```

### 8.2 生产构建

```bash
npm run pack
npm run build:dir
```

`npm run pack` 只生成 `dist/electron`；`npm run build:dir` 会继续调用 electron-builder，生成未压缩的 `build/mac-arm64/thief.app`（当前 arm64 机器）。

如果本机存在 Apple Development 证书，electron-builder 可能会尝试交互式签名并卡住。仅做本地未签名验证时使用：

```bash
CSC_IDENTITY_AUTO_DISCOVERY=false npm run build:dir
```

`build:mac` 脚本固定构建 x64；在 arm64 机器上验证本机产物应使用 `build:dir` 或显式执行 `electron-builder --mac --arm64 --dir`。

### 8.3 常用命令

| 命令 | 作用 |
| --- | --- |
| `npm run dev` | 开发模式 |
| `npm run pack` | 编译主进程和渲染进程到 `dist/electron` |
| `npm run build` | 编译并交给 electron-builder 打包当前平台 |
| `npm run build:mac` | 构建 macOS x64 |
| `npm run build:dir` | 构建当前架构的未压缩应用目录 |
| `npm run build:win64` / `build:win32` | 构建 Windows |
| `npm run build:linux64` / `build:linux32` | 构建 Linux |
| `npm run build:all` | 构建所有平台 |
| `npm run build:clean` | 清理构建产物 |
| `npm run build:web` | 构建 Web 目标；当前代码仍依赖 Electron 模块，不等同于可部署的纯 Web 版本 |

编译产物位于 `dist/electron`，打包产物默认位于 `build`。`build:web` 脚本和 Webpack 配置存在，但渲染代码直接依赖 Electron 的 `ipcRenderer`、remote、`fs` 等模块，不能把它视为可直接部署的纯 Web 版本。

## 9. 外部依赖与网络

| 地址 | 用途 |
| --- | --- |
| `http://hq.sinajs.cn/list=` | 股票行情 |
| `http://ad.c.team/ad?p=thief-book` | 老板键广告文本 |
| `https://gitee.com/lauix/public_version/raw/master/version.txt` | 检查更新版本号 |
| `https://thief.im`、`https://thief.im/docs` | 官网和使用文档 |
| `https://github.com/cteamx/Thief/releases` | 新版本下载入口 |
| `https://c.team` | 设置页作品链接 |

除广告和股票接口外，其余链接只在用户点击菜单时打开。广告和股票接口使用 HTTP；离线或接口变更时缺少明确错误提示。

## 10. 已知限制与风险

1. **Electron 安全边界仍然很弱**：所有窗口开启 `nodeIntegration` 并显式关闭 `contextIsolation`，网页窗口还开启了 `webviewTag`；渲染层通过 `@electron/remote` 直接访问主进程对象，没有预加载脚本隔离。网页模式可以加载任意 URL，因此不应在不可信网络环境或不信任页面中运行。
2. **工具链已升级，但业务依赖仍然偏旧**：Vue 2、`request`、`request-promise`、lowdb 1 仍在使用；这些依赖不是本次构建修复的目标，升级前需要单独评估行为变化。
3. **没有测试**：仓库没有单元测试、端到端测试、Lint 或类型检查，修改后需要手动验证多个平台和窗口模式。
4. **网络错误处理较弱**：股票请求失败时静默丢弃，广告失败回落到摸鱼文字，更新检查没有统一的错误提示。
5. **大文件内存压力**：视频整文件读入内存，PDF 为所有页面创建 Canvas，大文件可能导致内存暴涨。
6. **平台能力限制**：Touch Bar API 仅适用于 macOS；Touch Bar 模式无法在重启后保持，启动时会回落到菜单栏模式。
7. **窗口状态实现有边界问题**：创建窗口时用 `arr_wh.length` 判断位置数组是否有效；如果只保存过宽高而没有保存过位置，坐标可能变成 `NaN`。正常拖动保存过位置后，最终坐标可以恢复。
8. **快捷键注册提前返回**：`createKey()` 遇到某个格式不合法的快捷键会直接返回，可能导致后续快捷键没有注册；设置页默认值通常可以规避，但自定义快捷键时要注意。
9. **自动翻页开关语义相反**：第一次触发会写入 `auto_page=1` 并清除定时器，第二次触发会写入 `auto_page=0` 却启动定时器；托盘勾选状态也和实际运行状态不一致。
10. **构建仍有非阻塞告警**：Webpack 会报告 7 条 `export 'default' (imported as 'style0') was not found`，来自 Vue 单文件组件样式与 mini-css-extract 的组合；`styles.css` 能正常生成，页面也能正常挂载。开发主进程还会报告 `electron-debug` 的动态 `require` 告警，以及 Electron 45 下 `session.getAllExtensions`/`loadExtension` 的弃用提示。
11. **`@electron/remote` 有兼容性陷阱**：渲染进程应使用 `require('@electron/remote')` 获取 CommonJS 导出；使用默认 `import` 时会得到 `undefined`。新增渲染进程窗口时不要忘记在 `createWindow()` 中启用 remote。

## 11. 维护建议

- 优先补一层主进程业务服务边界，把小说、股票、窗口管理从 `src/main/index.js` 拆开后再改功能。
- 任何涉及 Electron 安全配置或任意 URL 加载的改动，先明确兼容性和信任边界。
- 长期方向应是用 preload + IPC 替代 `nodeIntegration` 和 `@electron/remote`，但这是一次跨窗口改造，不应和构建链升级混在一起。
- 不要重新引入 `node-sass`、Babel 6、Webpack 4 或 Electron 8；Electron 45 已移除 `webFrame.setLayoutZoomLevelLimits`，不要恢复该调用。
- 修改设置项时同时更新 `src/main/utils/db.js` 默认值、`src/renderer/components/setting.vue` 读写逻辑和本文件的配置表。
- 修改窗口行为或 IPC 名称时同步更新本文件的窗口/IPC 表。
- 发布前至少要验证：macOS 菜单栏模式、桌面模式、Touch Bar 模式；Windows 桌面模式；小说/股票/网页/视频/PDF 六类窗口；老板键、全局快捷键、退出快捷键和无网络场景。
