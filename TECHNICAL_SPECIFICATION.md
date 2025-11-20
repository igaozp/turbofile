# imFile (Turbofile) 技术规范文档

## 项目概述

### 基本信息
- **项目名称**: imFile
- **版本**: 1.1.2
- **项目类型**: 基于 Electron 的跨平台桌面下载管理器
- **源项目**: Fork 自 Motrix
- **开源协议**: MIT

### 核心功能
- 支持 HTTP/HTTPS 协议下载
- 支持 FTP 协议下载
- 支持 BitTorrent 种子下载
- 支持 Magnet 磁力链接下载
- 支持 Thunder 迅雷链接协议
- 多任务并发下载管理
- 断点续传
- 下载速度限制
- 文件关联和自定义协议处理
- 多语言支持(30+ 语言)
- 深色/浅色主题切换
- 系统托盘模式
- 自动更新

---

## 技术栈

### 核心框架
- **Electron**: 31.6.0 - 跨平台桌面应用框架
- **Node.js**: >=20.0.0 - JavaScript 运行时
- **Vue.js**: 2.7.16 - 前端 UI 框架
- **Vuex**: 3.6.2 - 状态管理
- **Vue Router**: 3.6.5 - 前端路由

### UI 组件库
- **Element UI**: 2.15.14 - 桌面端组件库
- **Tailwind CSS**: 3.4.12 - CSS 工具类框架
- **normalize.css**: 8.0.1 - CSS 重置

### 构建工具
- **Webpack**: 5.102.1 - 模块打包工具
- **Babel**: 7.x - JavaScript 编译器
- **electron-builder**: 25.0.5 - Electron 应用打包工具

### 下载引擎
- **Aria2**: 跨平台命令行下载工具(作为子进程运行)
  - 支持协议: HTTP/HTTPS, FTP, SFTP, BitTorrent, Metalink
  - 通信方式: JSON-RPC 2.0 over WebSocket/HTTP

### 其他关键依赖
- **electron-store**: 10.0.0 - 持久化配置存储
- **electron-log**: 5.2.0 - 日志系统
- **electron-updater**: 6.3.4 - 自动更新
- **i18next**: 23.15.1 - 国际化
- **axios**: 1.12.0 - HTTP 客户端
- **lodash**: 4.17.21 - 工具函数库
- **node-fetch**: 2.7.0 - Fetch API 实现
- **ws**: 8.18.3 - WebSocket 客户端

---

## 项目架构

### 目录结构

```
turbofile/
├── .electron-vue/           # Electron + Vue 构建配置
│   ├── build.js            # 生产构建脚本
│   ├── dev-runner.js       # 开发环境启动脚本
│   ├── webpack.main.config.js      # 主进程 Webpack 配置
│   ├── webpack.renderer.config.js  # 渲染进程 Webpack 配置
│   └── webpack.web.config.js       # Web 版本配置
│
├── build/                   # 构建钩子和辅助脚本
│   ├── afterPackHook.js    # 打包后处理
│   └── afterSignHook.js    # 签名后处理
│
├── extra/                   # 平台特定的额外资源
│   ├── darwin/             # macOS 资源
│   │   ├── x64/engine/    # x64 Aria2 二进制
│   │   └── arm64/engine/  # ARM64 Aria2 二进制
│   ├── linux/              # Linux 资源
│   │   ├── x64/engine/
│   │   ├── arm64/engine/
│   │   └── armv7l/engine/
│   └── win32/              # Windows 资源
│       ├── x64/engine/
│       └── ia32/engine/
│
├── src/                     # 源代码目录
│   ├── main/               # Electron 主进程
│   │   ├── core/          # 核心模块
│   │   ├── ui/            # UI 管理模块
│   │   ├── utils/         # 工具函数
│   │   ├── menus/         # 菜单定义
│   │   ├── configs/       # 配置
│   │   ├── pages/         # 页面 HTML 模板
│   │   ├── Application.js # 应用主类
│   │   ├── Launcher.js    # 启动器
│   │   └── index.js       # 入口文件
│   │
│   ├── renderer/           # Electron 渲染进程
│   │   ├── api/           # API 封装层
│   │   ├── assets/        # 静态资源
│   │   ├── components/    # Vue 组件
│   │   ├── pages/         # 页面
│   │   ├── router/        # 路由配置
│   │   ├── store/         # Vuex 状态管理
│   │   ├── utils/         # 工具函数
│   │   └── workers/       # Web Workers
│   │
│   └── shared/             # 主进程和渲染进程共享代码
│       ├── aria2/         # Aria2 客户端实现
│       ├── locales/       # 国际化语言包
│       ├── utils/         # 共享工具函数
│       └── constants.js   # 常量定义
│
├── static/                  # 静态资源(图标等)
├── electron-builder.json    # 打包配置
├── package.json            # 项目依赖
└── CLAUDE.md              # 项目开发指南
```

### 架构模式

#### 1. Electron 双进程架构

```
┌─────────────────────────────────────────────────────────┐
│                     Electron App                         │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  ┌─────────────────┐         ┌────────────────────┐    │
│  │   主进程 (Main)   │ ◄─IPC──► │  渲染进程 (Renderer) │    │
│  │                  │         │                     │    │
│  │  - Application   │         │  - Vue.js App       │    │
│  │  - Engine        │         │  - Vuex Store       │    │
│  │  - ConfigManager │         │  - Vue Router       │    │
│  │  - WindowManager │         │  - API Client       │    │
│  │  - MenuManager   │         │  - UI Components    │    │
│  │  - TrayManager   │         │                     │    │
│  │  - UpdateManager │         │                     │    │
│  └────────┬─────────┘         └─────────────────────┘    │
│           │                                               │
│           │ spawn                                         │
│           ▼                                               │
│  ┌─────────────────┐                                     │
│  │  Aria2 进程      │                                     │
│  │  (子进程)        │                                     │
│  │                  │                                     │
│  │  JSON-RPC 2.0   │                                     │
│  │  WebSocket/HTTP │                                     │
│  └─────────────────┘                                     │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

#### 2. 主进程架构

主进程采用**管理器模式**，通过 `Application` 类统一管理各个功能模块:

```javascript
Application (EventEmitter)
├── ConfigManager      // 配置管理(系统配置 + 用户配置)
├── Engine             // Aria2 引擎进程管理
├── EngineClient       // Aria2 RPC 客户端
├── WindowManager      // 窗口管理
├── MenuManager        // 菜单管理
├── TrayManager        // 托盘管理
├── DockManager        // Dock 图标管理(macOS)
├── TouchBarManager    // Touch Bar 管理(macOS)
├── ThemeManager       // 主题管理
├── UpdateManager      // 自动更新管理
├── ProtocolManager    // 自定义协议处理
├── UPnPManager        // UPnP 端口映射
├── AutoLaunchManager  // 开机自启动
└── EnergyManager      // 电源管理(防止休眠)
```

#### 3. 渲染进程架构

采用经典的 **Vue.js MVVM 架构** + **Vuex 状态管理**:

```
Vue.js Application
├── Router (Vue Router)
│   └── pages/
│       └── index/          // 主页面
│
├── Store (Vuex)
│   ├── modules/
│   │   ├── app.js         // 应用全局状态
│   │   ├── task.js        // 任务管理状态
│   │   └── preference.js  // 偏好设置状态
│   └── index.js
│
├── Components (Vue Components)
│   ├── Task/              // 任务相关组件
│   ├── Preference/        // 设置相关组件
│   ├── TaskDetail/        // 任务详情组件
│   └── ...
│
└── API Layer
    └── Api.js             // Aria2 API 封装
```

---

## 核心模块详解

### 1. 主进程核心模块

#### 1.1 Application.js - 应用主类

**职责**:
- 应用生命周期管理
- 各管理器的初始化和协调
- IPC 通信处理
- 事件总线(基于 EventEmitter)

**关键方法**:
```javascript
class Application extends EventEmitter {
  init()                    // 初始化所有管理器
  start(page, options)      // 启动应用
  show(page)               // 显示窗口
  hide(page)               // 隐藏窗口
  quit()                   // 退出应用
  relaunch()               // 重启应用
  savePreference(config)   // 保存配置
  handleIpcMessages()      // 处理 IPC 消息
  handleIpcInvokes()       // 处理 IPC 调用
  sendCommandToAll(cmd)    // 向所有窗口发送命令
}
```

**初始化流程**:
```
1. initContext()           // 初始化上下文
2. initConfigManager()     // 初始化配置管理器
3. setupLogger()           // 设置日志系统
4. initLocaleManager()     // 初始化国际化
5. setupApplicationMenu()  // 设置应用菜单
6. initWindowManager()     // 初始化窗口管理器
7. initUPnPManager()       // 初始化 UPnP
8. startEngine()           // 启动 Aria2 引擎
9. initEngineClient()      // 初始化引擎客户端
10. initThemeManager()     // 初始化主题管理器
11. initTrayManager()      // 初始化托盘
12. initTouchBarManager()  // 初始化 Touch Bar
13. initDockManager()      // 初始化 Dock
14. initAutoLaunchManager()// 初始化自启动
15. initEnergyManager()    // 初始化电源管理
16. initProtocolManager()  // 初始化协议处理
17. initUpdaterManager()   // 初始化更新管理器
18. handleCommands()       // 注册命令处理
19. handleEvents()         // 注册事件处理
20. handleIpcMessages()    // 注册 IPC 消息
21. handleIpcInvokes()     // 注册 IPC 调用
```

#### 1.2 Engine.js - Aria2 引擎管理

**职责**:
- 管理 Aria2 子进程的生命周期
- 构建 Aria2 启动参数
- 监控进程状态

**关键方法**:
```javascript
class Engine {
  start()                  // 启动 Aria2 进程
  stop()                   // 停止 Aria2 进程
  restart()                // 重启进程
  getEngineBinPath()       // 获取二进制路径
  getStartArgs()           // 构建启动参数
}
```

**启动参数构建**:
```javascript
// 基础参数
[
  '--conf-path=<aria2.conf路径>',
  '--save-session=<session文件路径>',
  '--input-file=<session文件路径>',  // 如果 session 文件存在
  ...extraConfig  // 额外配置(从 ConfigManager 读取)
]
```

**进程管理**:
- 使用 `child_process.spawn` 启动
- 在开发模式下捕获 stdout/stderr
- 进程 PID 写入文件便于管理
- 进程关闭时自动清理 PID 文件

#### 1.3 EngineClient.js - Aria2 RPC 客户端

**职责**:
- 封装与 Aria2 的 JSON-RPC 通信
- 提供高级 API 方法

**关键方法**:
```javascript
class EngineClient {
  call(method, ...args)           // 调用 RPC 方法
  changeGlobalOption(options)     // 修改全局选项
  shutdown(options)               // 关闭引擎
}
```

**通信协议**:
- 基于 JSON-RPC 2.0
- 支持 WebSocket(优先) 和 HTTP fallback
- 默认端口: 16800
- 支持 secret token 认证

#### 1.4 ConfigManager.js - 配置管理

**职责**:
- 管理系统配置(Aria2 配置)
- 管理用户配置(应用配置)
- 配置持久化(electron-store)
- 配置变更监听

**配置分类**:

**系统配置** (system.json) - Aria2 相关:
```javascript
{
  'all-proxy': '',
  'allow-overwrite': false,
  'auto-file-renaming': true,
  'bt-tracker': '',
  'continue': true,
  'dir': '下载目录',
  'max-concurrent-downloads': 5,
  'max-connection-per-server': 16,
  'max-download-limit': 0,
  'max-overall-download-limit': 0,
  'rpc-listen-port': 16800,
  'rpc-secret': '',
  'seed-ratio': 2,
  'seed-time': 2880,
  // ... 更多 Aria2 选项
}
```

**用户配置** (user.json) - 应用相关:
```javascript
{
  'auto-check-update': true,
  'auto-hide-window': false,
  'auto-sync-tracker': true,
  'enable-upnp': true,
  'keep-seeding': false,
  'locale': 'zh-CN',
  'theme': 'light',
  'run-mode': 1,
  'open-at-login': false,
  'protocols': { magnet: true, thunder: false },
  'proxy': { enable: false, server: '', bypass: '' },
  // ... 更多应用选项
}
```

**配置监听机制**:
```javascript
// 监听配置变更
userConfig.onDidChange(key, (newValue, oldValue) => {
  // 处理配置变更
})
```

#### 1.5 WindowManager.js - 窗口管理

**职责**:
- 创建和管理 BrowserWindow 实例
- 窗口状态持久化
- 窗口事件处理

**关键功能**:
- 多窗口管理(index, about 等)
- 窗口位置和尺寸记忆
- 窗口显示/隐藏
- 全屏状态管理
- 向窗口发送命令

#### 1.6 TrayManager.js - 托盘管理

**职责**:
- 系统托盘图标和菜单
- 托盘速度显示(macOS)
- 拖放文件支持
- 下载状态指示

**特性**:
- 支持亮色/暗色主题图标
- 实时速度显示(Speedometer)
- 右键菜单
- 拖放 .torrent 文件
- 拖放文本(磁力链接)

### 2. 渲染进程核心模块

#### 2.1 API Layer (Api.js)

**职责**:
- 封装 Aria2 JSON-RPC API
- 提供高级业务方法
- 数据格式转换(camelCase ↔ kebab-case)

**关键方法**:

**任务管理**:
```javascript
// 添加任务
addUri(params)           // HTTP/FTP 下载
addTorrent(params)       // BT 下载
addMetalink(params)      // Metalink 下载

// 任务控制
pauseTask(params)        // 暂停
resumeTask(params)       // 恢复
removeTask(params)       // 移除
pauseAllTask()           // 暂停所有
resumeAllTask()          // 恢复所有

// 批量操作
batchPauseTask(params)   // 批量暂停
batchResumeTask(params)  // 批量恢复
batchRemoveTask(params)  // 批量移除

// 任务查询
fetchTaskList(params)    // 获取任务列表
fetchTaskItem(params)    // 获取任务详情
fetchTaskItemWithPeers(params)  // 获取任务及 Peers
```

**配置管理**:
```javascript
getGlobalOption()        // 获取全局配置
changeGlobalOption(options)  // 修改全局配置
getOption(params)        // 获取任务配置
changeOption(params)     // 修改任务配置
savePreference(params)   // 保存偏好设置
```

**统计信息**:
```javascript
getGlobalStat()          // 获取全局统计
getVersion()             // 获取 Aria2 版本
```

#### 2.2 Vuex Store

**状态管理架构**:

**app.js** - 应用全局状态:
```javascript
state: {
  downloadSpeed: 0,        // 下载速度
  uploadSpeed: 0,          // 上传速度
  interval: 1000,          // 更新间隔
  stat: {},               // 全局统计
  about: {},              // 关于信息
  config: {},             // 应用配置
  theme: 'light',         // 主题
  locale: 'zh-CN',        // 语言
  // ...
}

actions: {
  fetchGlobalStat()       // 获取全局统计
  fetchPreference()       // 获取偏好设置
  savePreference()        // 保存偏好设置
  updateTheme()           // 更新主题
  updateLocale()          // 更新语言
}
```

**task.js** - 任务管理状态:
```javascript
state: {
  currentList: 'active',   // 当前列表类型
  taskList: [],            // 任务列表
  selectedGidList: [],     // 选中的任务 GID
  taskDetailVisible: false,// 详情面板可见性
  currentTaskGid: '',      // 当前任务 GID
  currentTaskItem: null,   // 当前任务详情
  currentTaskFiles: [],    // 当前任务文件
  currentTaskPeers: [],    // 当前任务 Peers
  seedingList: [],         // 做种任务列表
  count: {                 // 任务计数
    downloading: 0,
    seeding: 0,
    waiting: 0,
    stoped: 0
  }
}

actions: {
  fetchList()              // 获取任务列表
  addUri()                 // 添加 URI 任务
  addTorrent()             // 添加 BT 任务
  pauseTask()              // 暂停任务
  resumeTask()             // 恢复任务
  removeTask()             // 移除任务
  showTaskDetail()         // 显示任务详情
  selectTasks()            // 选择任务
  batchPauseTask()         // 批量暂停
  batchResumeTask()        // 批量恢复
  // ...
}
```

**preference.js** - 偏好设置状态:
```javascript
state: {
  config: {},              // 配置对象
}

actions: {
  fetchPreference()        // 获取配置
  save()                   // 保存配置
}
```

### 3. 共享模块

#### 3.1 Aria2 客户端实现

**架构**:
```
Aria2.js (extends JSONRPCClient)
└── JSONRPCClient.js (extends EventEmitter)
    ├── WebSocket 通信
    └── HTTP fallback
```

**JSONRPCClient.js** - JSON-RPC 2.0 客户端:
```javascript
class JSONRPCClient extends EventEmitter {
  // 核心方法
  call(method, params)     // RPC 调用
  batch(calls)             // 批量调用
  open()                   // 打开 WebSocket 连接
  close()                  // 关闭连接

  // 底层通信
  websocket(message)       // WebSocket 发送
  http(message)            // HTTP 发送

  // 消息处理
  _onmessage(message)      // 处理响应
  _onresponse(response)    // 处理 RPC 响应
  _onnotification(notif)   // 处理通知
}
```

**Aria2.js** - Aria2 专用封装:
```javascript
class Aria2 extends JSONRPCClient {
  // 方法名自动添加 "aria2." 前缀
  prefix(str)

  // 自动添加 secret token
  addSecret(parameters)

  // 调用 Aria2 方法
  call(method, ...params)

  // 批量调用
  multicall(calls)
  batch(calls)

  // 获取支持的方法和通知
  listMethods()
  listNotifications()
}
```

**通信流程**:
```
1. 优先使用 WebSocket (ws://127.0.0.1:16800/jsonrpc)
2. WebSocket 不可用时降级到 HTTP (http://127.0.0.1:16800/jsonrpc)
3. 所有请求自动添加 secret token
4. 支持事件监听 (下载开始/暂停/完成/错误等)
```

#### 3.2 Constants.js - 常量定义

**关键常量**:

```javascript
// 应用主题
APP_THEME: { AUTO: 'auto', LIGHT: 'light', DARK: 'dark' }

// 运行模式
APP_RUN_MODE: { STANDARD: 1, TRAY: 2, HIDE_TRAY: 3 }

// 任务状态
TASK_STATUS: {
  ACTIVE: 'active',
  WAITING: 'waiting',
  PAUSED: 'paused',
  ERROR: 'error',
  COMPLETE: 'complete',
  REMOVED: 'removed',
  SEEDING: 'seeding'
}

// 引擎配置
ENGINE_RPC_HOST: '127.0.0.1'
ENGINE_RPC_PORT: 16800
ENGINE_MAX_CONCURRENT_DOWNLOADS: 10
ENGINE_MAX_CONNECTION_PER_SERVER: 64

// 时间常量
ONE_SECOND: 1000
ONE_MINUTE: 60000
ONE_HOUR: 3600000
ONE_DAY: 86400000
AUTO_SYNC_TRACKER_INTERVAL: 43200000  // 12 小时
AUTO_CHECK_UPDATE_INTERVAL: 604800000  // 7 天

// Tracker 源
NGOSANG_TRACKERS_BEST_URL_CDN: '...'
XIU2_TRACKERS_BEST_URL_CDN: '...'
```

---

## 数据流和通信机制

### 1. IPC 通信机制

#### 1.1 通信模式

**主进程 → 渲染进程** (单向):
```javascript
// 主进程发送命令
windowManager.sendCommandTo(window, 'application:update-theme', { theme })

// 渲染进程监听
window.addEventListener('application:update-theme', (event) => {
  // 处理命令
})
```

**渲染进程 → 主进程** (单向):
```javascript
// 渲染进程发送命令
ipcRenderer.send('command', 'application:save-preference', config)

// 主进程监听
ipcMain.on('command', (event, command, ...args) => {
  this.emit(command, ...args)
})
```

**渲染进程 → 主进程** (双向 - invoke/handle):
```javascript
// 渲染进程调用
const config = await ipcRenderer.invoke('get-app-config')

// 主进程处理
ipcMain.handle('get-app-config', async () => {
  return {
    ...systemConfig,
    ...userConfig,
    ...context
  }
})
```

#### 1.2 命令系统

**主进程命令** (通过 Application EventEmitter):
```javascript
// 应用控制
'application:save-preference'      // 保存配置
'application:relaunch'             // 重启应用
'application:quit'                 // 退出应用
'application:show'                 // 显示窗口
'application:hide'                 // 隐藏窗口
'application:reset-session'        // 重置会话
'application:factory-reset'        // 恢复出厂设置

// 主题和语言
'application:change-theme'         // 切换主题
'application:change-locale'        // 切换语言

// 窗口和托盘
'application:toggle-dock'          // 切换 Dock 显示
'application:auto-hide-window'     // 自动隐藏窗口
'application:update-tray'          // 更新托盘

// 文件和协议
'application:open-file'            // 打开文件
'application:reveal-in-folder'     // 在文件夹中显示
'application:setup-protocols-client'  // 设置协议处理

// 帮助
'help:official-website'            // 打开官网
```

**渲染进程命令** (广播到所有窗口):
```javascript
'application:update-locale'        // 更新语言
'application:update-theme'         // 更新主题
'application:update-system-theme'  // 更新系统主题
'application:update-tray-focused'  // 更新托盘焦点状态
'application:update-preference-config'  // 更新配置
'application:new-bt-task-with-file'     // 新建 BT 任务
'application:show-task-detail'     // 显示任务详情
```

### 2. 下载任务数据流

```
┌──────────────┐
│  用户操作     │ (添加下载)
└──────┬───────┘
       │
       ▼
┌────────────────────────┐
│  Vue Component         │
│  (AddTask.vue)         │
└───────┬────────────────┘
        │ dispatch('task/addUri')
        ▼
┌────────────────────────┐
│  Vuex Store            │
│  (task.js)             │
└───────┬────────────────┘
        │ api.addUri()
        ▼
┌────────────────────────┐
│  API Layer             │
│  (Api.js)              │
└───────┬────────────────┘
        │ client.call('addUri')
        ▼
┌────────────────────────┐
│  Aria2 Client          │
│  (Aria2.js)            │
└───────┬────────────────┘
        │ WebSocket/HTTP
        │ JSON-RPC 2.0
        ▼
┌────────────────────────┐
│  Aria2 Engine          │
│  (子进程)               │
└───────┬────────────────┘
        │
        ▼
   执行下载任务
```

### 3. 实时状态更新

**定时轮询机制**:
```javascript
// 每秒更新一次全局统计
setInterval(() => {
  store.dispatch('app/fetchGlobalStat')
}, 1000)

// 每 2 秒更新一次任务列表
setInterval(() => {
  store.dispatch('task/fetchList')
}, 2000)
```

**事件驱动更新**:
```javascript
// Aria2 事件通知
client.on('onDownloadStart', (event) => {
  // 下载开始
})

client.on('onDownloadPause', (event) => {
  // 下载暂停
})

client.on('onDownloadComplete', (event) => {
  // 下载完成
})

client.on('onDownloadError', (event) => {
  // 下载错误
})
```

### 4. 配置同步机制

```
┌─────────────────────────────────────────────┐
│  用户修改配置                                  │
└──────────┬──────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────┐
│  Vuex: preference/save()                    │
└──────────┬──────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────┐
│  API: savePreference()                      │
│  - 分离配置: system/user/others              │
└──────────┬──────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────┐
│  IPC: 'command', 'application:save-preference' │
└──────────┬──────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────┐
│  Application: savePreference()              │
│  - ConfigManager.setSystemConfig()          │
│  - ConfigManager.setUserConfig()            │
│  - EngineClient.changeGlobalOption()        │
└──────────┬──────────────────────────────────┘
           │
           ├─► electron-store 持久化
           │
           └─► Aria2 引擎应用配置
```

---

## 构建和部署

### 1. 开发环境

**启动开发服务器**:
```bash
yarn run dev
```

**开发模式特性**:
- Hot Module Replacement (HMR)
- DevTools 自动打开
- Source Maps
- 详细错误输出
- Aria2 stdout/stderr 输出

### 2. 构建流程

#### 2.1 构建脚本

**完整构建**:
```bash
yarn run build
```

**构建流程**:
```
1. 清理旧构建文件
2. 编译主进程代码 (webpack.main.config.js)
   - Babel 转译
   - 代码压缩(生产环境)
   - 输出到 dist/electron/main.js
3. 编译渲染进程代码 (webpack.renderer.config.js)
   - Vue 单文件组件编译
   - Babel 转译
   - CSS 提取和压缩
   - 资源复制
   - 输出到 dist/electron/
4. electron-builder 打包
   - 复制 extra/ 资源
   - 创建安装包
   - 代码签名(如配置)
   - 输出到 release/
```

#### 2.2 Webpack 配置

**主进程配置** (webpack.main.config.js):
```javascript
{
  target: 'electron-main',
  entry: 'src/main/index.js',
  output: 'dist/electron/main.js',
  externals: [...dependencies],  // 外部依赖不打包
  module: {
    rules: [
      { test: /\.js$/, use: 'babel-loader' },
      { test: /\.node$/, use: 'node-loader' }
    ]
  },
  resolve: {
    alias: {
      '@': 'src/main',
      '@shared': 'src/shared'
    }
  }
}
```

**渲染进程配置** (webpack.renderer.config.js):
```javascript
{
  target: 'electron-renderer',
  entry: 'src/renderer/main.js',
  output: 'dist/electron/',
  module: {
    rules: [
      { test: /\.vue$/, use: 'vue-loader' },
      { test: /\.js$/, use: 'babel-loader' },
      { test: /\.css$/, use: ['vue-style-loader', 'css-loader'] },
      { test: /\.scss$/, use: ['vue-style-loader', 'css-loader', 'sass-loader'] },
      { test: /\.(png|jpe?g|gif|svg)$/, use: 'file-loader' }
    ]
  },
  plugins: [
    new VueLoaderPlugin(),
    new HtmlWebpackPlugin(),
    new MiniCssExtractPlugin()
  ]
}
```

#### 2.3 electron-builder 配置

**配置文件**: electron-builder.json

**多平台打包**:

**macOS**:
```javascript
{
  "mac": {
    "target": ["dmg", "zip"],
    "arch": ["x64", "arm64"],
    "category": "public.app-category.utilities",
    "extraResources": {
      "from": "./extra/darwin/${arch}/",
      "to": "./"
    }
  }
}
```

**Windows**:
```javascript
{
  "win": {
    "target": ["nsis", "appx", "zip", "portable"],
    "arch": ["x64"],
    "extraResources": {
      "from": "./extra/win32/${arch}/",
      "to": "./"
    }
  },
  "nsis": {
    "oneClick": false,
    "allowToChangeInstallationDirectory": true
  }
}
```

**Linux**:
```javascript
{
  "linux": {
    "target": ["AppImage", "deb", "rpm", "snap"],
    "arch": ["x64", "arm64", "armv7l"],
    "category": "Network",
    "extraResources": {
      "from": "./extra/linux/${arch}/",
      "to": "./"
    }
  }
}
```

### 3. 平台特定构建

**Windows 7 支持** (使用 Electron 22):
```bash
yarn run build:win7
```

**Apple Silicon 支持**:
```bash
yarn run build:applesilicon
```

### 4. 自动更新

**配置**:
```javascript
// UpdateManager.js
class UpdateManager {
  constructor() {
    autoUpdater.setFeedURL({
      provider: 'github',
      owner: 'imfile-io',
      repo: 'imfile-desktop'
    })
  }

  check() {
    autoUpdater.checkForUpdates()
  }
}
```

**更新流程**:
```
1. 定期检查更新(7天间隔)
2. 发现新版本 → 后台下载
3. 下载完成 → 通知用户
4. 用户确认 → 退出并安装
```

---

## 关键技术点

### 1. Aria2 集成

#### 1.1 进程管理

**启动流程**:
```javascript
// 1. 获取平台和架构对应的二进制文件
const binPath = getAria2BinPath(process.platform, process.arch)
// 例如: extra/darwin/arm64/engine/aria2c

// 2. 构建启动参数
const args = [
  '--conf-path=extra/darwin/arm64/engine/aria2.conf',
  '--save-session=/Users/xxx/.config/imFile/session',
  '--input-file=/Users/xxx/.config/imFile/session',
  '--dir=/Users/xxx/Downloads',
  '--max-concurrent-downloads=5',
  // ... 更多配置
]

// 3. 启动子进程
const instance = spawn(binPath, args, {
  windowsHide: false,
  stdio: 'pipe'
})

// 4. 监听进程事件
instance.on('close', () => { /* 清理 */ })
instance.stdout.on('data', (data) => { /* 日志 */ })
```

**配置文件管理**:
```
aria2.conf (每个平台独立)
├── 基础配置(aria2.conf 中定义)
└── 运行时配置(通过命令行参数覆盖)
    ├── 系统配置(ConfigManager.systemConfig)
    └── 用户配置影响的配置(如 keep-seeding)
```

#### 1.2 RPC 通信

**连接建立**:
```javascript
// 1. 创建 Aria2 客户端
const client = new Aria2({
  host: '127.0.0.1',
  port: 16800,
  secret: '<rpc-secret>'
})

// 2. 打开 WebSocket 连接
await client.open()
// ws://127.0.0.1:16800/jsonrpc

// 3. 监听连接事件
client.on('open', () => { /* 连接成功 */ })
client.on('close', () => { /* 连接关闭 */ })
client.on('error', (err) => { /* 连接错误 */ })
```

**方法调用**:
```javascript
// 添加下载任务
const gid = await client.call('addUri', [
  ['http://example.com/file.zip'],  // uris
  {                                  // options
    dir: '/path/to/download',
    out: 'file.zip',
    'max-connection-per-server': 16
  }
])

// 获取任务状态
const status = await client.call('tellStatus', gid, [
  'gid', 'status', 'totalLength', 'completedLength', 'downloadSpeed'
])
```

**批量调用**:
```javascript
// Multicall (一次 RPC 调用)
const results = await client.multicall([
  ['aria2.tellActive'],
  ['aria2.tellWaiting', 0, 10],
  ['aria2.tellStopped', 0, 10]
])
```

**事件通知**:
```javascript
// Aria2 支持的事件
client.on('onDownloadStart', (params) => {
  const [event] = params
  console.log('Download started:', event.gid)
})

client.on('onDownloadComplete', (params) => {
  const [event] = params
  console.log('Download completed:', event.gid)
})

client.on('onDownloadError', (params) => {
  const [event] = params
  console.log('Download error:', event.gid)
})
```

### 2. 会话持久化

**Session 文件**:
- 路径: `~/.config/imFile/session`
- 格式: Aria2 session 格式
- 内容: 所有未完成的下载任务信息

**保存机制**:
```javascript
// 自动保存: Aria2 配置
{
  'save-session': '/path/to/session',
  'save-session-interval': 60  // 每 60 秒保存一次
}

// 手动保存: API 调用
await client.call('saveSession')
```

**恢复机制**:
```javascript
// 启动时加载 session
const args = [
  '--input-file=/path/to/session'  // 恢复所有任务
]
```

### 3. 多语言支持

**i18next 配置**:
```javascript
// 语言包路径: src/shared/locales/{locale}/
// 支持的语言: 30+ (zh-CN, en-US, ja, ko, ...)

i18next.init({
  lng: 'zh-CN',
  fallbackLng: 'en-US',
  resources: {
    'zh-CN': { translation: require('./locales/zh-CN') },
    'en-US': { translation: require('./locales/en-US') },
    // ...
  }
})
```

**语言切换**:
```javascript
// 主进程
localeManager.changeLanguageByLocale(newLocale)
  .then(() => {
    menuManager.handleLocaleChange(newLocale)
    trayManager.handleLocaleChange(newLocale)
  })

// 渲染进程
this.$i18n.changeLanguage(newLocale)
```

### 4. 主题系统

**主题类型**:
- `auto`: 跟随系统
- `light`: 浅色主题
- `dark`: 深色主题

**实现方式**:
```javascript
// 1. 监听系统主题变化 (nativeTheme)
nativeTheme.on('updated', () => {
  const theme = nativeTheme.shouldUseDarkColors ? 'dark' : 'light'
  this.emit('system-theme-change', theme)
})

// 2. 应用主题到窗口
BrowserWindow.setBackgroundColor(theme === 'dark' ? '#1f1f1f' : '#ffffff')

// 3. 渲染进程应用主题
document.body.setAttribute('data-theme', theme)
```

**CSS 变量**:
```css
/* Light Theme */
:root[data-theme="light"] {
  --color-primary: #409EFF;
  --color-background: #FFFFFF;
  --color-text: #303133;
}

/* Dark Theme */
:root[data-theme="dark"] {
  --color-primary: #409EFF;
  --color-background: #1F1F1F;
  --color-text: #E4E7ED;
}
```

### 5. 协议处理

**支持的协议**:
- `mo://` - imFile 自定义协议
- `imfile://` - imFile 协议
- `magnet:` - 磁力链接
- `thunder:` - 迅雷链接

**注册协议**:
```javascript
// electron-builder.json
{
  "protocols": [
    {
      "name": "imFile Protocol",
      "schemes": ["mo", "imfile"]
    },
    {
      "name": "Magnet Protocol",
      "schemes": ["magnet"]
    },
    {
      "name": "Thunder Protocol",
      "schemes": ["thunder"]
    }
  ]
}
```

**处理协议**:
```javascript
// ProtocolManager.js
class ProtocolManager {
  handle(url) {
    if (url.startsWith('magnet:')) {
      // 解析磁力链接并创建下载任务
      this.handleMagnetLink(url)
    } else if (url.startsWith('thunder:')) {
      // 解析迅雷链接
      this.handleThunderLink(url)
    }
  }
}

// 应用启动时注册
app.on('open-url', (event, url) => {
  event.preventDefault()
  protocolManager.handle(url)
})
```

### 6. UPnP 端口映射

**实现**:
```javascript
// UPnPManager.js
class UPnPManager {
  async map(port) {
    // 使用 @motrix/nat-api 库
    const client = await natAPI.createClient()
    await client.map({
      publicPort: port,
      privatePort: port,
      protocol: 'TCP',
      ttl: 7200  // 2 小时
    })
  }

  async unmap(port) {
    await client.unmap({ publicPort: port })
  }
}
```

**应用场景**:
- BT 监听端口 (默认: 21301)
- DHT 监听端口 (默认: 26701)

### 7. Tracker 自动同步

**数据源**:
```javascript
// ngosang/trackerslist
'https://cdn.jsdelivr.net/gh/ngosang/trackerslist/trackers_best.txt'

// XIU2/TrackersListCollection
'https://cdn.jsdelivr.net/gh/XIU2/TrackersListCollection/best.txt'
```

**同步逻辑**:
```javascript
// 每 12 小时自动同步
setInterval(() => {
  if (shouldSync()) {
    fetchBtTrackerFromSource(sources)
      .then((trackers) => {
        // 更新 bt-tracker 配置
        configManager.setSystemConfig('bt-tracker', trackers.join(','))
      })
  }
}, 12 * 60 * 60 * 1000)
```

### 8. 电源管理

**防止系统休眠**:
```javascript
// EnergyManager.js
class EnergyManager {
  startPowerSaveBlocker() {
    if (!this.id) {
      this.id = powerSaveBlocker.start('prevent-app-suspension')
    }
  }

  stopPowerSaveBlocker() {
    if (this.id) {
      powerSaveBlocker.stop(this.id)
      this.id = null
    }
  }
}

// 有下载任务时阻止休眠
app.on('download-status-change', (downloading) => {
  if (downloading) {
    energyManager.startPowerSaveBlocker()
  } else {
    energyManager.stopPowerSaveBlocker()
  }
})
```

---

## 开发指南

### 1. 环境要求

- Node.js >= 20.0.0
- Yarn 1.x
- Git

### 2. 开发流程

**克隆项目**:
```bash
git clone https://github.com/imfile-io/imfile-desktop.git
cd imfile-desktop
```

**安装依赖**:
```bash
yarn install
```

**启动开发**:
```bash
yarn run dev
```

**代码规范检查**:
```bash
yarn run lint
yarn run lint:fix
```

### 3. 项目结构约定

**命名规范**:
- 文件名: PascalCase (类) 或 kebab-case (组件)
- Vue 组件: PascalCase
- 配置常量: UPPER_SNAKE_CASE
- 函数和变量: camelCase

**代码组织**:
- 主进程代码: `src/main/`
- 渲染进程代码: `src/renderer/`
- 共享代码: `src/shared/`
- 每个管理器一个文件
- 相关组件放在同一目录

### 4. 调试技巧

**主进程调试**:
```bash
# 启用 Node.js 调试
yarn run dev -- --inspect=5858

# Chrome DevTools
chrome://inspect
```

**渲染进程调试**:
- 自动打开 DevTools(开发模式)
- Vue DevTools 扩展

**日志系统**:
```javascript
// 主进程
import logger from './core/Logger'
logger.info('message')
logger.warn('warning')
logger.error('error')

// 渲染进程
console.log('message')
```

**日志文件位置**:
- macOS: `~/Library/Logs/imFile/`
- Windows: `%USERPROFILE%\AppData\Roaming\imFile\logs\`
- Linux: `~/.config/imFile/logs/`

### 5. 常见问题

**Aria2 启动失败**:
- 检查二进制文件权限
- 检查端口是否被占用
- 查看日志文件

**WebSocket 连接失败**:
- 确认 Aria2 进程正在运行
- 检查 RPC 端口配置
- 验证 secret 配置

**构建失败**:
- 清理构建缓存: `yarn run build:clean`
- 重新安装依赖: `rm -rf node_modules && yarn install`
- 检查 Node.js 版本

---

## 扩展开发指南

### 1. 添加新的下载协议

**步骤**:

1. 在 `electron-builder.json` 中注册协议
2. 在 `ProtocolManager` 中添加处理逻辑
3. 解析协议 URL 并转换为 Aria2 支持的格式
4. 调用 API 添加下载任务

### 2. 添加新的配置项

**步骤**:

1. 在 `ConfigManager` 的 defaults 中添加默认值
2. 在 Preference 组件中添加 UI 控件
3. 在 preference.js store 中处理保存逻辑
4. 监听配置变更并应用

### 3. 添加新的任务操作

**步骤**:

1. 在 `Api.js` 中封装 Aria2 API
2. 在 `task.js` store 中添加 action
3. 在 Task 组件中添加 UI 按钮
4. 绑定事件处理

### 4. 添加新的语言

**步骤**:

1. 在 `src/shared/locales/` 下创建语言目录
2. 复制 `en-US/` 的所有文件
3. 翻译所有文本
4. 在 constants.js 中添加语言选项

---

## 性能优化建议

### 1. 渲染进程优化

- 使用虚拟滚动处理大量任务列表
- 避免频繁的 DOM 操作
- 使用 `Object.freeze()` 冻结不可变数据
- 合理使用 computed 和 watch

### 2. 主进程优化

- 避免阻塞主线程
- 使用 Web Workers 处理耗时计算
- 合理设置更新间隔
- 及时清理事件监听器

### 3. 网络优化

- 使用 WebSocket 代替轮询
- 批量 RPC 调用减少网络开销
- 合理设置 Aria2 配置参数

### 4. 内存优化

- 及时释放不再使用的对象
- 避免内存泄漏(事件监听器、定时器)
- 合理设置任务列表显示数量

---

## 安全注意事项

### 1. RPC 安全

- 使用 secret token 认证
- RPC 端口仅监听 localhost
- 不暴露 RPC 端口到外网

### 2. 文件下载安全

- 验证下载链接
- 检查文件类型
- 病毒扫描(可选)
- 沙箱隔离(未实现)

### 3. 配置安全

- 敏感配置加密存储
- 避免在日志中输出敏感信息
- 定期清理临时文件

---

## 测试策略

### 1. 单元测试

**推荐工具**:
- Jest
- @vue/test-utils

**测试重点**:
- 工具函数
- Vuex mutations/actions
- API 层逻辑

### 2. 集成测试

**推荐工具**:
- Spectron (Electron 应用测试)

**测试重点**:
- 窗口管理
- IPC 通信
- 配置持久化

### 3. 端到端测试

**测试场景**:
- 添加下载任务
- 暂停/恢复任务
- 修改配置
- 主题切换

---

## 部署清单

### 1. 构建前检查

- [ ] 更新版本号 (package.json)
- [ ] 更新 CHANGELOG
- [ ] 运行代码检查
- [ ] 测试所有核心功能
- [ ] 检查 Aria2 二进制文件

### 2. 构建

- [ ] 清理旧构建: `yarn run build:clean`
- [ ] 构建应用: `yarn run build`
- [ ] 测试构建后的应用

### 3. 发布

- [ ] 创建 Git tag
- [ ] 推送到 GitHub
- [ ] 上传构建产物到 Release
- [ ] 更新官网下载链接
- [ ] 发布更新公告

---

## 附录

### A. Aria2 常用配置参数

```ini
# 下载目录
dir=/path/to/downloads

# 并发下载数
max-concurrent-downloads=5

# 单服务器最大连接数
max-connection-per-server=16

# 最小分片大小
min-split-size=10M

# 分片数
split=16

# 下载速度限制 (0=无限制)
max-overall-download-limit=0
max-download-limit=0

# 上传速度限制
max-overall-upload-limit=1M
max-upload-limit=0

# 断点续传
continue=true

# 文件预分配
file-allocation=falloc

# BT 设置
bt-max-peers=50
bt-request-peer-speed-limit=50K
bt-seed-unverified=true
bt-save-metadata=true
bt-tracker=<tracker列表>

# DHT
enable-dht=true
enable-dht6=true
dht-listen-port=26701

# RPC
enable-rpc=true
rpc-listen-all=false
rpc-listen-port=16800
rpc-secret=<secret>
```

### B. 常用 Aria2 API

**任务管理**:
- `aria2.addUri(uris, [options])` - 添加 URI 任务
- `aria2.addTorrent(torrent, [uris], [options])` - 添加 BT 任务
- `aria2.remove(gid)` - 移除任务
- `aria2.pause(gid)` - 暂停任务
- `aria2.unpause(gid)` - 恢复任务
- `aria2.tellStatus(gid, [keys])` - 获取任务状态
- `aria2.tellActive([keys])` - 获取活动任务
- `aria2.tellWaiting(offset, num, [keys])` - 获取等待任务
- `aria2.tellStopped(offset, num, [keys])` - 获取已停止任务

**全局操作**:
- `aria2.getGlobalOption()` - 获取全局配置
- `aria2.changeGlobalOption(options)` - 修改全局配置
- `aria2.getGlobalStat()` - 获取全局统计
- `aria2.purgeDownloadResult()` - 清除已完成/错误任务记录
- `aria2.saveSession()` - 保存会话

**系统**:
- `aria2.getVersion()` - 获取版本
- `aria2.shutdown()` - 关闭 Aria2
- `system.multicall(methods)` - 批量调用

### C. 项目资源链接

- **官网**: https://imfile.io
- **GitHub**: https://github.com/imfile-io/imfile-desktop
- **Aria2 文档**: https://aria2.github.io/manual/en/html/
- **Electron 文档**: https://www.electronjs.org/docs
- **Vue.js 文档**: https://vuejs.org/
- **Element UI 文档**: https://element.eleme.io/

---

## 总结

imFile 是一个架构清晰、功能完整的 Electron 桌面下载管理器。项目采用了成熟的技术栈和清晰的模块划分,通过 Aria2 作为下载引擎实现了强大的下载功能。

### 核心技术亮点

1. **双进程架构**: 主进程负责系统交互,渲染进程负责 UI,分工明确
2. **管理器模式**: 主进程通过多个管理器类实现功能解耦
3. **JSON-RPC 通信**: 通过 WebSocket/HTTP 与 Aria2 引擎通信
4. **状态管理**: Vuex 统一管理应用状态
5. **配置持久化**: electron-store 实现配置的持久化存储
6. **多语言支持**: i18next 实现国际化
7. **跨平台**: 支持 macOS, Windows, Linux 多个平台

### 适合复刻的场景

- 需要构建跨平台桌面下载管理器
- 需要集成 Aria2 等命令行工具
- 需要实现复杂的 Electron 应用架构
- 需要参考成熟的 Vue.js + Electron 项目结构

---

**文档版本**: 1.0
**生成日期**: 2025-11-20
**基于项目版本**: 1.1.2
