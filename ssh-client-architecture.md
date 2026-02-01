# SSH Client 架构设计文档

## 技术栈

| 层级 | 技术选型 |
|------|----------|
| 框架 | Electron |
| 前端 | React + Vite + Tailwind CSS |
| 终端 | xterm.js + xterm-addon-* |
| SSH | ssh2 |
| 串口 | serialport |
| 数据库 | SQLite (better-sqlite3) |
| 安全存储 | Electron safeStorage API |
| 云同步 | GitHub Gist API |

---

## 1. 系统整体架构

```mermaid
graph TB
    subgraph Electron["Electron 应用"]
        subgraph Main["Main Process (主进程)"]
            IPC[IPC Handler]
            WindowManager[窗口管理器]
            
            subgraph CoreServices["核心服务层"]
                SSHService[SSH 服务]
                SFTPService[SFTP 服务]
                SerialService[串口服务]
                AgentService[SSH Agent 桥接]
            end
            
            subgraph DataLayer["数据层"]
                DBManager[SQLite 管理器]
                KeyStore[密钥存储]
                SafeStorage[SafeStorage 加密]
                LogManager[日志管理器]
            end
            
            subgraph SyncLayer["同步层"]
                GistSync[Gist 同步引擎]
                ConflictResolver[冲突解决器]
                ChangeDetector[变更检测器]
            end
        end
        
        subgraph Renderer["Renderer Process (渲染进程)"]
            subgraph UI["UI 层"]
                MainWindow[主窗口]
                SFTPWindow[SFTP 窗口]
                SnippetWindow[代码片段窗口]
                KeyMgrWindow[密钥管理窗口]
                SettingsWindow[设置窗口]
            end
            
            subgraph State["状态管理"]
                GlobalState[全局状态]
                SessionState[会话状态]
                UIState[UI 状态]
            end
        end
    end
    
    subgraph External["外部依赖"]
        SystemAgent[系统 SSH Agent]
        OnePassAgent[1Password SSH Agent]
        GistAPI[GitHub Gist API]
        FileSystem[文件系统]
    end
    
    Renderer <-->|IPC| Main
    SSHService --> SystemAgent
    SSHService --> OnePassAgent
    GistSync --> GistAPI
    DataLayer --> FileSystem
```

---

## 2. 主进程模块架构

```mermaid
graph LR
    subgraph MainProcess["Main Process"]
        subgraph Window["窗口管理"]
            WM[WindowManager]
            MW[MainWindow]
            IW[IndependentWindows]
            WS[WindowState 持久化]
        end
        
        subgraph Connection["连接管理"]
            CM[ConnectionManager]
            SSH[SSHConnection]
            SFTP[SFTPConnection]
            Serial[SerialConnection]
            Tunnel[TunnelManager]
        end
        
        subgraph Security["安全模块"]
            KM[KeyManager]
            KG[KeyGenerator]
            SS[SafeStorage]
            KH[KnownHosts]
        end
        
        subgraph Data["数据管理"]
            DB[(SQLite)]
            Config[ConfigManager]
            Log[LogManager]
            Theme[ThemeManager]
        end
        
        subgraph Sync["云同步"]
            GS[GistSyncEngine]
            CD[ChangeDetector]
            CR[ConflictResolver]
            DQ[DebounceQueue]
        end
    end
    
    WM --> MW
    WM --> IW
    WM --> WS
    
    CM --> SSH
    CM --> SFTP
    CM --> Serial
    SSH --> Tunnel
    
    KM --> KG
    KM --> SS
    SSH --> KM
    SSH --> KH
    
    Config --> DB
    Log --> DB
    Theme --> DB
    
    GS --> CD
    GS --> CR
    CD --> DQ
```

---

## 3. 渲染进程 UI 架构

```mermaid
graph TB
    subgraph MainWindow["主窗口布局"]
        subgraph TitleBar["自定义标题栏"]
            DragRegion[拖拽区域]
            TabBar[标签栏]
            ToolButtons[工具按钮组]
            WinControls[窗口控制按钮]
        end
        
        subgraph Content["内容区域"]
            subgraph Sidebar["侧边栏"]
                DeviceTree[设备树]
                TreeContextMenu[右键菜单]
                ResizeHandle[宽度调整手柄]
            end
            
            subgraph Main["主区域"]
                TerminalTabs[终端标签页]
                TerminalView[终端视图]
            end
        end
        
        StatusBar[状态栏]
    end
    
    subgraph IndependentWindows["独立窗口"]
        SFTPWindow[SFTP 文件管理器]
        SnippetWindow[代码片段管理]
        KeyWindow[密钥管理]
        SettingsWindow[设置]
    end
    
    subgraph Modals["模态框"]
        HostEditor[主机编辑器]
        GroupEditor[分组编辑器]
        QuickActions[快捷操作面板]
    end
    
    ToolButtons -->|点击| IndependentWindows
    DeviceTree -->|双击| TerminalTabs
    DeviceTree -->|右键| TreeContextMenu
    TreeContextMenu -->|编辑| HostEditor
```

---

## 4. 主窗口详细布局

```
┌─────────────────────────────────────────────────────────────────────────┐
│ [拖拽区域]   │ + │Tab1 │Tab2 │(如果存在,不存在则隐藏tab) │ 📁 📝 🔑 ⚙️│ ― □ ✕ │
├─────────────┼──────────────────────────────────────────────────────────┤
│             │                                                          │
│  ▼ 生产环境  │  $ ssh user@server                                       │
│    ├ Web-01 │  Welcome to Ubuntu 22.04 LTS                             │
│    ├ Web-02 │                                                          │
│    └ DB-01  │  user@server:~$ █                                        │
│             │                                                          │
│  ▼ 测试环境  │                                                          │
│    └ Test-01│                                                          │
│             │                                                          │
│  ▼ 串口设备  │                                                          │
│    └ COM3   │                                                          │
│             │                                                          │
│ [可拖拽边缘] │                                                          │
├─────────────┴──────────────────────────────────────────────────────────┤
│ 🟢 已连接 │ 隧道: L:8080→R:80 │ 延迟: 23ms │ 192.168.1.100           │
└─────────────────────────────────────────────────────────────────────────┘

注：Windows 下工具按钮组左移，为窗口控制按钮腾出空间
    macOS 下窗口控制按钮在左侧，布局无需调整
```

### 同步策略

```mermaid
flowchart TD
    Start[检测到变更] --> Debounce{30秒防抖}
    Debounce -->|等待中有新变更| Debounce
    Debounce -->|30秒无新变更| FetchRemote[获取远程数据]
    
    FetchRemote --> Compare[按记录比较时间戳]
    
    Compare --> LocalNewer{本地更新?}
    LocalNewer -->|是| Upload[上传到 Gist]
    LocalNewer -->|否| RemoteNewer{远程更新?}
    
    RemoteNewer -->|是| Download[下载到本地]
    RemoteNewer -->|否| NoAction[无需操作]
    
    Upload --> UpdateMeta[更新同步元数据]
    Download --> UpdateMeta
    NoAction --> End[完成]
    UpdateMeta --> End
```

---

## 5. 目录结构

```
ssh-client/
├── electron/                    # Electron 主进程
│   ├── main.ts                  # 入口文件
│   ├── preload.ts               # 预加载脚本
│   │
│   ├── windows/                 # 窗口管理
│   │   ├── WindowManager.ts
│   │   ├── MainWindow.ts
│   │   └── IndependentWindow.ts
│   │
│   ├── services/                # 核心服务
│   │   ├── ssh/
│   │   │   ├── SSHService.ts
│   │   │   ├── SFTPService.ts
│   │   │   ├── TunnelManager.ts
│   │   │   └── AgentBridge.ts
│   │   ├── serial/
│   │   │   └── SerialService.ts
│   │   └── terminal/
│   │       ├── ZmodemHandler.ts
│   │       └── BroadcastManager.ts
│   │
│   ├── data/                    # 数据层
│   │   ├── Database.ts          # SQLite 管理
│   │   ├── repositories/        # 数据仓库
│   │   │   ├── HostRepository.ts
│   │   │   ├── KeyRepository.ts
│   │   │   ├── SnippetRepository.ts
│   │   │   └── ...
│   │   └── migrations/          # 数据库迁移
│   │
│   ├── security/                # 安全模块
│   │   ├── KeyManager.ts
│   │   ├── KeyGenerator.ts
│   │   ├── SafeStorageWrapper.ts
│   │   └── KnownHostsManager.ts
│   │
│   ├── sync/                    # 云同步
│   │   ├── GistSyncEngine.ts
│   │   ├── ChangeDetector.ts
│   │   ├── ConflictResolver.ts
│   │   └── SyncScheduler.ts
│   │
│   └── ipc/                     # IPC 处理
│       ├── handlers/
│       └── IPCBridge.ts
│
├── src/                         # React 渲染进程
│   ├── main.tsx                 # React 入口
│   ├── App.tsx
│   │
│   ├── components/              # UI 组件
│   │   ├── layout/
│   │   │   ├── TitleBar.tsx
│   │   │   ├── Sidebar.tsx
│   │   │   ├── StatusBar.tsx
│   │   │   └── TabBar.tsx
│   │   ├── terminal/
│   │   │   ├── TerminalView.tsx
│   │   │   ├── TerminalTabs.tsx
│   │   │   └── SearchBar.tsx
│   │   ├── device-tree/
│   │   │   ├── DeviceTree.tsx
│   │   │   ├── TreeNode.tsx
│   │   │   └── ContextMenu.tsx
│   │   ├── modals/
│   │   │   ├── HostEditor.tsx
│   │   │   ├── GroupEditor.tsx
│   │   │   └── TunnelEditor.tsx
│   │   └── common/
│   │
│   ├── windows/                 # 独立窗口页面
│   │   ├── sftp/
│   │   │   ├── SFTPWindow.tsx
│   │   │   ├── FileList.tsx
│   │   │   └── TransferQueue.tsx
│   │   ├── snippets/
│   │   │   └── SnippetManager.tsx
│   │   ├── keys/
│   │   │   ├── KeyManager.tsx
│   │   │   └── KeyGenerator.tsx
│   │   └── settings/
│   │       ├── SettingsWindow.tsx
│   │       ├── GeneralSettings.tsx
│   │       ├── TerminalSettings.tsx
│   │       ├── ThemeSettings.tsx
│   │       ├── ShortcutSettings.tsx
│   │       └── SyncSettings.tsx
│   │
│   ├── hooks/                   # React Hooks
│   │   ├── useTerminal.ts
│   │   ├── useConnection.ts
│   │   ├── useTheme.ts
│   │   └── useShortcuts.ts
│   │
│   ├── store/                   # 状态管理
│   │   ├── index.ts
│   │   ├── slices/
│   │   │   ├── sessionSlice.ts
│   │   │   ├── uiSlice.ts
│   │   │   └── settingsSlice.ts
│   │   └── selectors/
│   │
│   ├── services/                # 渲染进程服务
│   │   └── ipc.ts               # IPC 客户端
│   │
│   ├── styles/                  # 样式
│   │   ├── index.css
│   │   └── themes/
│   │
│   └── types/                   # TypeScript 类型
│       └── index.ts
│
├── assets/                      # 静态资源
│   └── fonts/
│       └── JetBrainsMono/       # 内置等宽字体
│
├── scripts/                     # 构建脚本
│
├── package.json
├── electron-builder.yml         # 打包配置
├── vite.config.ts
├── tailwind.config.js
└── tsconfig.json
```

---


## 6. 技术要点

### 7. 自定义标题栏
```
- 使用 frameless window
- CSS: -webkit-app-region: drag / no-drag
- Windows: 右侧放置窗口控制按钮
- macOS: 左侧预留系统按钮空间 (traffic lights)
```

### 7.1 xterm.js 插件
```
- @xterm/addon-fit          # 自适应尺寸
- @xterm/addon-search       # 搜索功能
- @xterm/addon-web-links    # 链接识别
- @xterm/addon-unicode11    # Unicode 支持
- @xterm/addon-webgl        # GPU 渲染 (可选)
- zmodem.js                 # Zmodem 支持
```

### 7.2 密钥文件存储
```
位置: {userData}/keys/
文件: {keyId}.pem
元数据: SQLite keys 表
加密: safeStorage 加密 passphrase
```

### 7.3 日志存储
```
位置: {userData}/logs/{hostId}/{date}.log
格式: 纯文本，带时间戳
轮转: 按日期分文件
```

---

## 7.4 安全考虑

| 数据类型 | 存储方式 | 同步方式 |
|----------|----------|----------|
| 主机密码 | safeStorage 加密后存 SQLite | 加密后同步 |
| 密钥文件 | 原文件 + safeStorage 加密 passphrase | 加密后同步 |
| GitHub Token | safeStorage 加密 | 不同步 |
| 其他配置 | SQLite 明文 | 明文同步 |
