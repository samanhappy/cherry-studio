# 划词助手功能技术实现文档

## 目录

- [1. 概述](#1-概述)
- [2. 底层原理](#2-底层原理)
- [3. 技术架构](#3-技术架构)
- [4. 核心组件](#4-核心组件)
- [5. 实现细节](#5-实现细节)
- [6. 技术框架](#6-技术框架)
- [7. 数据流](#7-数据流)
- [8. 实现方案对比](#8-实现方案对比)
- [9. 优缺点分析](#9-优缺点分析)
- [10. 平台兼容性](#10-平台兼容性)

---

## 1. 概述

划词助手（Selection Assistant）是 Cherry Studio 的核心功能之一，允许用户在任何应用程序中选择文本后，通过浮动工具栏快速执行翻译、搜索、AI 问答等操作。该功能采用系统级文本选择监听机制，实现了跨应用程序的无缝文本交互体验。

### 1.1 主要功能

- **全局文本选择监听**：监听操作系统级别的文本选择事件
- **浮动工具栏**：在选中文本附近显示操作按钮
- **快速操作**：复制、搜索、翻译、AI 对话等预定义操作
- **自定义操作**：支持用户自定义操作和 AI 助手
- **多种触发模式**：选中即显示、Ctrl 键触发、快捷键触发
- **应用过滤**：支持白名单/黑名单模式，过滤特定应用

### 1.2 关键特性

- **跨应用支持**：适用于几乎所有应用程序（浏览器、编辑器、PDF 阅读器等）
- **智能定位**：工具栏根据选择位置和屏幕边界智能定位
- **非侵入式**：不影响原应用的正常使用
- **高性能**：预加载窗口机制确保快速响应
- **平台适配**：针对 Windows 和 macOS 的特殊优化

---

## 2. 底层原理

### 2.1 文本选择检测机制

划词助手的核心是检测操作系统级别的文本选择事件。实现原理基于以下技术：

#### 2.1.1 系统钩子（System Hook）

使用 `selection-hook` native 模块实现底层文本选择监听：

```typescript
// 从 selection-hook 模块导入
import type {
  SelectionHookConstructor,
  SelectionHookInstance,
  TextSelectionData
} from 'selection-hook'

// 初始化钩子
this.selectionHook = new SelectionHook()
```

**工作原理：**

1. **Windows 平台**：
   - 使用 Windows API 的全局钩子（Global Hooks）
   - 监听 WM_COPY 和 WM_PASTE 消息
   - 通过剪贴板和光标位置检测文本选择
   - 使用 SetWindowsHookEx 注册低级键盘和鼠标钩子

2. **macOS 平台**：
   - 使用 Accessibility API
   - 需要授予辅助功能权限
   - 监听 NSPasteboard 变化
   - 使用 CGEventTap 监听鼠标和键盘事件

#### 2.1.2 剪贴板监听

```typescript
// 检测文本选择的关键步骤：
// 1. 监听鼠标按下和松开事件
// 2. 检测 Ctrl+C（复制）操作
// 3. 读取剪贴板内容
// 4. 获取光标位置和选择矩形
```

#### 2.1.3 位置信息获取

```typescript
interface TextSelectionData {
  text: string                    // 选中的文本
  programName: string             // 程序名称
  posLevel: PositionLevel         // 位置精度级别
  mousePosStart: { x: number; y: number }  // 鼠标起始位置
  mousePosEnd: { x: number; y: number }    // 鼠标结束位置
  startTop: { x: number; y: number }       // 选择矩形左上角
  startBottom: { x: number; y: number }    // 选择矩形左下角
  endTop: { x: number; y: number }         // 选择矩形右上角
  endBottom: { x: number; y: number }      // 选择矩形右下角
  isFullscreen?: boolean          // 是否全屏模式（macOS）
}
```

### 2.2 窗口管理机制

#### 2.2.1 Electron BrowserWindow 特性

使用 Electron 的 BrowserWindow API 创建特殊类型的窗口：

**工具栏窗口特性：**
```typescript
{
  frame: false,              // 无边框
  transparent: true,         // 透明背景
  alwaysOnTop: true,        // 始终置顶
  skipTaskbar: true,        // 不显示在任务栏
  focusable: false,         // 不可获得焦点（Windows）
  type: 'toolbar',          // Windows：工具栏类型
  type: 'panel',            // macOS：面板类型
  hiddenInMissionControl: true  // macOS：隐藏在 Mission Control
}
```

**操作窗口特性：**
```typescript
{
  frame: false,              // 无边框
  transparent: true,         // 透明背景
  hasShadow: false,         // 无阴影
  titleBarStyle: 'hidden',  // 隐藏标题栏（macOS）
  sandbox: true             // 启用沙箱
}
```

#### 2.2.2 窗口层级管理

```typescript
// 工具栏设置最高层级
this.toolbarWindow.setAlwaysOnTop(true, 'screen-saver')

// macOS 全屏应用兼容
this.toolbarWindow.setVisibleOnAllWorkspaces(true, {
  visibleOnFullScreen: true,
  skipTransformProcessType: true
})
```

### 2.3 IPC 通信机制

主进程（Main Process）和渲染进程（Renderer Process）通过 IPC 通道通信：

```typescript
// 主进程注册处理器
ipcMain.handle(IpcChannel.Selection_ProcessAction, (_, actionItem) => {
  selectionService?.processAction(actionItem)
})

// 渲染进程调用
window.api.selection.processAction(actionItem)

// 主进程发送事件
this.toolbarWindow.webContents.send(
  IpcChannel.Selection_TextSelected, 
  selectionData
)

// 渲染进程监听
window.electron.ipcRenderer.on(
  IpcChannel.Selection_TextSelected,
  (_, selectionData) => {
    // 处理选择数据
  }
)
```

---

## 3. 技术架构

### 3.1 整体架构图

```
┌─────────────────────────────────────────────────────────────┐
│                     Operating System                         │
│  ┌──────────────────────────────────────────────────────┐  │
│  │           Native Selection Hook (C++)                 │  │
│  │  - Windows: SetWindowsHookEx, WM_COPY               │  │
│  │  - macOS: Accessibility API, CGEventTap             │  │
│  └──────────────────┬───────────────────────────────────┘  │
└─────────────────────┼───────────────────────────────────────┘
                      │
                      │ Node Native Addon
                      ↓
┌─────────────────────────────────────────────────────────────┐
│              Electron Main Process (Node.js)                 │
│  ┌──────────────────────────────────────────────────────┐  │
│  │            SelectionService (Singleton)              │  │
│  │  - Text selection event handling                     │  │
│  │  - Window lifecycle management                       │  │
│  │  - Position calculation                              │  │
│  │  - Filter mode processing                            │  │
│  │  - IPC handler registration                          │  │
│  └──────────────────┬───────────────────────────────────┘  │
│                     │                                         │
│  ┌──────────────────┴───────────────────────────────────┐  │
│  │              ConfigManager                            │  │
│  │  - Settings persistence                               │  │
│  │  - Config subscriptions                               │  │
│  └──────────────────┬───────────────────────────────────┘  │
└─────────────────────┼───────────────────────────────────────┘
                      │
                      │ IPC Communication
                      ↓
┌─────────────────────────────────────────────────────────────┐
│            Electron Renderer Process (Chromium)              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │         Toolbar Window (React)                       │  │
│  │  - SelectionToolbar component                        │  │
│  │  - Action buttons rendering                          │  │
│  │  - Animation effects                                 │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │      Action Window (React)                           │  │
│  │  - SelectionActionApp component                      │  │
│  │  - ActionTranslate / ActionGeneral                   │  │
│  │  - AI interaction                                    │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │         Redux Store (State Management)                │  │
│  │  - selectionStore: settings and actions              │  │
│  │  - Sync with ConfigManager                           │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 架构层次

#### 3.2.1 Native Layer（Native 层）

- **selection-hook 模块**：C++ 实现的 Node.js native addon
- 职责：系统级事件监听、剪贴板操作、窗口信息获取

#### 3.2.2 Service Layer（服务层）

- **SelectionService**：核心服务类，单例模式
- **ConfigManager**：配置管理，持久化存储
- **ShortcutService**：快捷键集成
- 职责：业务逻辑、状态管理、窗口生命周期

#### 3.2.3 UI Layer（UI 层）

- **SelectionToolbar**：浮动工具栏 UI
- **SelectionActionApp**：操作窗口 UI
- **Redux Store**：状态管理
- 职责：用户交互、数据展示、动画效果

### 3.3 设计模式

#### 3.3.1 Singleton Pattern（单例模式）

```typescript
export class SelectionService {
  private static instance: SelectionService | null = null
  
  public static getInstance(): SelectionService | null {
    if (!SelectionService.instance) {
      SelectionService.instance = new SelectionService()
    }
    return SelectionService.instance
  }
  
  private constructor() {
    // 私有构造函数，确保单例
  }
}
```

**优点**：
- 全局只有一个实例，避免资源冲突
- 统一管理所有选择事件和窗口

#### 3.3.2 Observer Pattern（观察者模式）

```typescript
// 订阅配置变化
configManager.subscribe(ConfigKeys.SelectionAssistantTriggerMode, 
  (triggerMode: TriggerMode) => {
    this.triggerMode = triggerMode
    this.processTriggerMode()
  }
)

// 监听选择事件
this.selectionHook.on('text-selection', this.processTextSelection)
```

#### 3.3.3 Factory Pattern（工厂模式）

```typescript
// 预加载窗口池
private createPreloadedActionWindow(): BrowserWindow {
  // 创建配置好的窗口实例
}

private popActionWindow(): BrowserWindow {
  // 从池中获取或创建新窗口
  const actionWindow = this.preloadedActionWindows.pop() 
    || this.createPreloadedActionWindow()
  return actionWindow
}
```

#### 3.3.4 Strategy Pattern（策略模式）

```typescript
enum TriggerMode {
  Selected = 'selected',    // 选中即显示
  Ctrlkey = 'ctrlkey',     // Ctrl 键触发
  Shortcut = 'shortcut'     // 快捷键触发
}

private processTriggerMode(): void {
  switch (this.triggerMode) {
    case TriggerMode.Selected:
      this.selectionHook.setSelectionPassiveMode(false)
      break
    case TriggerMode.Ctrlkey:
      this.selectionHook.on('key-down', this.handleKeyDownCtrlkeyMode)
      this.selectionHook.setSelectionPassiveMode(true)
      break
    case TriggerMode.Shortcut:
      this.selectionHook.setSelectionPassiveMode(true)
      break
  }
}
```

---

## 4. 核心组件

### 4.1 SelectionService

**位置**：`src/main/services/SelectionService.ts`

**职责**：
- 初始化和管理 selection-hook 实例
- 处理文本选择事件
- 管理工具栏和操作窗口的生命周期
- 处理不同触发模式的逻辑
- 实现应用过滤功能
- 处理 IPC 通信

**关键方法**：

```typescript
class SelectionService {
  // 启动服务
  public start(): boolean
  
  // 停止服务
  public stop(): boolean
  
  // 处理文本选择
  private processTextSelection(selectionData: TextSelectionData): void
  
  // 显示工具栏
  private showToolbarAtPosition(point: Point, orientation: string): void
  
  // 计算工具栏位置
  private calculateToolbarPosition(refPoint: Point, orientation: string): Point
  
  // 处理操作
  public processAction(actionItem: ActionItem, isFullScreen: boolean): void
  
  // 创建操作窗口
  private createPreloadedActionWindow(): BrowserWindow
}
```

**状态管理**：

```typescript
private initStatus: boolean = false      // 初始化状态
private started: boolean = false         // 运行状态
private triggerMode = TriggerMode.Selected  // 触发模式
private isFollowToolbar = true           // 操作窗口跟随工具栏
private filterMode = 'default'           // 过滤模式
private filterList: string[] = []        // 过滤列表
private toolbarWindow: BrowserWindow | null = null     // 工具栏窗口
private actionWindows = new Set<BrowserWindow>()       // 操作窗口集合
private preloadedActionWindows: BrowserWindow[] = []   // 预加载窗口池
```

### 4.2 SelectionToolbar

**位置**：`src/renderer/src/windows/selection/toolbar/SelectionToolbar.tsx`

**职责**：
- 渲染浮动工具栏 UI
- 显示操作按钮
- 处理用户点击事件
- 实现动画效果

**组件结构**：

```tsx
<Container>
  <LogoWrapper>
    <Logo />  {/* 应用图标，可拖拽 */}
  </LogoWrapper>
  <ActionWrapper>
    <ActionIcons>
      <ActionButton>  {/* 复制按钮 */}
        <ActionIcon />
        <ActionTitle />
      </ActionButton>
      <ActionButton>  {/* 搜索按钮 */}
        ...
      </ActionButton>
      {/* 更多操作按钮 */}
    </ActionIcons>
  </ActionWrapper>
</Container>
```

**特性**：
- 使用 styled-components 实现样式
- CSS 变量支持自定义样式
- 动画：旋转、缩放、淡入淡出
- 响应式布局（紧凑模式/标准模式）

### 4.3 SelectionActionApp

**位置**：`src/renderer/src/windows/selection/action/SelectionActionApp.tsx`

**职责**：
- 渲染操作窗口 UI
- 执行具体操作（翻译、AI 对话等）
- 处理窗口控制（最小化、关闭、置顶、透明度）
- 实现自动滚动

**窗口控制功能**：

```tsx
- Pin（置顶）：窗口保持在最前
- Minimize（最小化）：最小化到任务栏
- Close（关闭）：关闭窗口
- Opacity（透明度）：调整窗口透明度 20%-100%
- Auto Close（自动关闭）：失去焦点时自动关闭
- Auto Pin（自动置顶）：打开时自动置顶
```

**内容组件**：
- **ActionTranslate**：翻译功能
- **ActionGeneral**：通用 AI 对话功能

### 4.4 配置管理

**位置**：`src/main/services/ConfigManager.ts`

**相关配置键**：

```typescript
ConfigKeys.SelectionAssistantEnabled        // 是否启用
ConfigKeys.SelectionAssistantTriggerMode    // 触发模式
ConfigKeys.SelectionAssistantFollowToolbar  // 是否跟随工具栏
ConfigKeys.SelectionAssistantRemeberWinSize // 记住窗口大小
ConfigKeys.SelectionAssistantFilterMode     // 过滤模式
ConfigKeys.SelectionAssistantFilterList     // 过滤列表
```

**预定义黑名单**：

**Windows**：
```typescript
[
  'explorer.exe',          // 文件管理器
  'excel.exe',            // Excel
  'powerpnt.exe',         // PowerPoint
  'photoshop.exe',        // Photoshop
  'snipaste.exe',         // 截图工具
  // ... 更多
]
```

**macOS**：
```typescript
[
  'com.apple.finder'      // 访达
]
```

### 4.5 Redux Store

**位置**：`src/renderer/src/store/selectionStore`

**状态定义**：

```typescript
interface SelectionState {
  selectionEnabled: boolean      // 是否启用
  triggerMode: TriggerMode       // 触发模式
  isCompact: boolean            // 紧凑模式
  isAutoClose: boolean          // 自动关闭
  isAutoPin: boolean            // 自动置顶
  isFollowToolbar: boolean      // 跟随工具栏
  isRemeberWinSize: boolean     // 记住窗口大小
  filterMode: FilterMode        // 过滤模式
  filterList: string[]          // 过滤列表
  actionWindowOpacity: number   // 窗口透明度
  actionItems: ActionItem[]     // 操作列表
}
```

---

## 5. 实现细节

### 5.1 文本选择处理流程

```
1. 用户在任意应用选择文本
   ↓
2. selection-hook 监听到选择事件
   ↓
3. 获取选中文本和位置信息
   ↓
4. 触发 'text-selection' 事件
   ↓
5. SelectionService.processTextSelection()
   ↓
6. 应用过滤检查（shouldProcessTextSelection）
   ↓
7. 计算工具栏位置（calculateToolbarPosition）
   ↓
8. 显示工具栏（showToolbarAtPosition）
   ↓
9. 发送 IPC 事件到工具栏窗口
   ↓
10. SelectionToolbar 接收数据并显示
```

### 5.2 位置计算算法

工具栏位置计算考虑以下因素：

**输入**：
- `refPoint`：参考点坐标（鼠标位置或选择矩形）
- `orientation`：期望方向（topLeft, topRight, bottomMiddle 等）

**处理步骤**：

```typescript
// 1. 根据方向计算初始位置
switch (orientation) {
  case 'topLeft':
    posPoint.x = refPoint.x - toolbarWidth
    posPoint.y = refPoint.y - toolbarHeight
    break
  case 'bottomMiddle':
    posPoint.x = refPoint.x - toolbarWidth / 2
    posPoint.y = refPoint.y
    break
  // ... 其他方向
}

// 2. 获取显示器信息
const display = screen.getDisplayNearestPoint(refPoint)

// 3. 边界检查和调整
posPoint.x = Math.max(
  display.workArea.x, 
  Math.min(posPoint.x, display.workArea.x + display.workArea.width - toolbarWidth)
)
posPoint.y = Math.max(
  display.workArea.y,
  Math.min(posPoint.y, display.workArea.y + display.workArea.height - toolbarHeight)
)

// 4. 特殊情况调整（超出屏幕上下边界）
if (exceedsTop) posPoint.y += 32
if (exceedsBottom) posPoint.y -= 32
```

**位置精度级别**：

```typescript
enum PositionLevel {
  NONE,           // 无位置信息，使用光标位置
  MOUSE_SINGLE,   // 单次鼠标点击
  MOUSE_DUAL,     // 鼠标拖拽选择
  SEL_FULL,       // 完整选择矩形
  SEL_DETAILED    // 详细选择矩形（包含多行）
}
```

### 5.3 触发模式实现

#### 5.3.1 选中模式（Selected）

```typescript
// 自动触发，选中文本立即显示工具栏
this.selectionHook.setSelectionPassiveMode(false)
this.selectionHook.on('text-selection', this.processTextSelection)
```

#### 5.3.2 Ctrl 键模式（Ctrlkey）

```typescript
// 按住 Ctrl 键 350ms 后触发
this.selectionHook.on('key-down', this.handleKeyDownCtrlkeyMode)
this.selectionHook.on('key-up', this.handleKeyUpCtrlkeyMode)
this.selectionHook.setSelectionPassiveMode(true)

private handleKeyDownCtrlkeyMode = (data: KeyboardEventData) => {
  if (this.lastCtrlkeyDownTime === 0) {
    this.lastCtrlkeyDownTime = Date.now()
    return
  }
  
  if (Date.now() - this.lastCtrlkeyDownTime >= 350) {
    const selectionData = this.selectionHook!.getCurrentSelection()
    if (selectionData) {
      this.processTextSelection(selectionData)
    }
  }
}
```

#### 5.3.3 快捷键模式（Shortcut）

```typescript
// 通过全局快捷键触发
this.selectionHook.setSelectionPassiveMode(true)

// ShortcutService 调用
public processSelectTextByShortcut(): void {
  const selectionData = this.selectionHook.getCurrentSelection()
  if (selectionData) {
    this.processTextSelection(selectionData)
  }
}
```

### 5.4 窗口预加载机制

为了提高响应速度，实现了窗口预加载池：

```typescript
// 预加载数量
private readonly PRELOAD_ACTION_WINDOW_COUNT = 1

// 初始化时创建预加载窗口
private async initPreloadedActionWindows(): Promise<void> {
  for (let i = 0; i < this.PRELOAD_ACTION_WINDOW_COUNT; i++) {
    await this.pushNewActionWindow()
  }
}

// 获取窗口时从池中取出
private popActionWindow(): BrowserWindow {
  const actionWindow = this.preloadedActionWindows.pop() 
    || this.createPreloadedActionWindow()
  
  // 异步创建新窗口补充池
  this.pushNewActionWindow()
  
  return actionWindow
}
```

**优点**：
- 窗口已加载完成，立即可用
- 避免显示时的加载延迟
- 提升用户体验

### 5.5 平台特殊处理

#### 5.5.1 Windows 特殊处理

```typescript
// 窗口类型
type: 'toolbar'

// 不可获得焦点
focusable: false

// 坐标转换（物理坐标 ↔ 逻辑坐标）
const mousePoint = screen.screenToDipPoint({ x: data.x, y: data.y })
```

#### 5.5.2 macOS 特殊处理

```typescript
// 需要辅助功能权限
systemPreferences.isTrustedAccessibilityClient(false)

// 窗口类型
type: 'panel'

// 全屏应用兼容
this.toolbarWindow.setVisibleOnAllWorkspaces(true, {
  visibleOnFullScreen: true,
  skipTransformProcessType: true
})

// 显示时不激活（避免其他窗口被带到前面）
this.toolbarWindow.showInactive()

// 隐藏时的焦点处理（避免其他窗口被带到前面）
const focusableWindows: BrowserWindow[] = []
for (const window of BrowserWindow.getAllWindows()) {
  if (window.isFocusable()) {
    focusableWindows.push(window)
    window.setFocusable(false)
  }
}
this.toolbarWindow.hide()
setTimeout(() => {
  focusableWindows.forEach(window => window.setFocusable(true))
}, 50)
```

### 5.6 应用过滤实现

```typescript
enum FilterMode {
  DEFAULT = 'default',      // 默认模式（使用预定义黑名单）
  WHITELIST = 'whitelist',  // 白名单模式（仅在列表中的应用启用）
  BLACKLIST = 'blacklist'   // 黑名单模式（排除列表中的应用）
}

private setHookGlobalFilterMode(mode: string, list: string[]): void {
  const predefinedBlacklist = isWin 
    ? SELECTION_PREDEFINED_BLACKLIST.WINDOWS 
    : SELECTION_PREDEFINED_BLACKLIST.MAC
  
  let combinedList: string[] = list
  let combinedMode = mode
  
  // 仅在选中模式下合并预定义黑名单
  if (this.triggerMode === TriggerMode.Selected) {
    switch (mode) {
      case 'blacklist':
        // 合并用户黑名单和预定义黑名单
        combinedList = [...new Set([...list, ...predefinedBlacklist])]
        break
      case 'whitelist':
        combinedList = [...list]
        break
      case 'default':
        // 使用预定义黑名单
        combinedList = [...predefinedBlacklist]
        combinedMode = 'blacklist'
        break
    }
  }
  
  this.selectionHook.setGlobalFilterMode(modeMap[combinedMode], combinedList)
}
```

### 5.7 精细调优列表

针对特定应用的行为优化：

```typescript
// 排除光标检测的应用（避免误触发）
EXCLUDE_CLIPBOARD_CURSOR_DETECT: {
  WINDOWS: ['acrobat.exe', 'wps.exe', 'cajviewer.exe'],
  MAC: []
}

// 延迟读取剪贴板的应用（等待应用完成复制）
INCLUDE_CLIPBOARD_DELAY_READ: {
  WINDOWS: ['acrobat.exe', 'wps.exe', 'cajviewer.exe', 'foxitphantom.exe'],
  MAC: []
}
```

---

## 6. 技术框架

### 6.1 核心技术栈

| 技术 | 版本 | 用途 |
|-----|------|------|
| Electron | Latest | 桌面应用框架 |
| React | 18.x | UI 框架 |
| TypeScript | 5.x | 类型系统 |
| Redux Toolkit | Latest | 状态管理 |
| styled-components | 6.x | CSS-in-JS |
| Ant Design | 5.x | UI 组件库 |
| selection-hook | 1.0.12 | Native 选择监听 |

### 6.2 构建工具

- **electron-vite**：Electron 应用构建
- **Vite**：快速的开发服务器和构建工具
- **rolldown-vite**：实验性 bundler

### 6.3 开发工具

- **Biome**：代码格式化和 linting
- **Vitest**：单元测试框架
- **TypeScript**：类型检查

### 6.4 依赖关系

```
cherry-studio
├── electron (主框架)
├── selection-hook (核心依赖，Native 模块)
│   └── 提供系统级文本选择监听
├── React (UI 层)
│   ├── SelectionToolbar 组件
│   └── SelectionActionApp 组件
├── Redux (状态管理)
│   └── selectionStore
└── Ant Design (UI 组件)
    ├── Button, Switch, Radio
    ├── Slider, Tooltip
    └── ...
```

---

## 7. 数据流

### 7.1 初始化流程

```
1. app.ready 事件触发
   ↓
2. initSelectionService()
   ↓
3. SelectionService.getInstance()
   ↓
4. new SelectionHook() (加载 native 模块)
   ↓
5. 检查操作系统支持和权限
   ↓
6. SelectionService.start()
   ↓
7. 创建工具栏窗口（隐藏）
   ↓
8. 初始化预加载操作窗口池
   ↓
9. 注册事件监听器
   ↓
10. 启动 selection-hook
```

### 7.2 选择到显示流程

```
[用户操作]
用户选择文本
   ↓
[Native Layer]
selection-hook 检测到选择
   ↓
获取文本内容和位置
   ↓
[Service Layer]
触发 'text-selection' 事件
   ↓
SelectionService.processTextSelection()
   ↓
检查应用过滤
   ↓
计算工具栏位置
   ↓
显示工具栏窗口
   ↓
[IPC]
发送 Selection_TextSelected 事件
   ↓
[UI Layer]
SelectionToolbar 接收事件
   ↓
更新状态和 UI
   ↓
显示操作按钮
```

### 7.3 操作执行流程

```
[用户操作]
点击工具栏按钮
   ↓
[UI Layer]
handleAction(actionItem)
   ↓
根据操作类型处理：
├─ copy: 复制到剪贴板
├─ search: 打开搜索引擎
├─ quote: 引用到主窗口
└─ 其他: 打开操作窗口
   ↓
[IPC]
window.api.selection.processAction()
   ↓
[Service Layer]
SelectionService.processAction()
   ↓
从预加载池获取窗口
   ↓
发送操作数据到窗口
   ↓
显示操作窗口
   ↓
[UI Layer]
SelectionActionApp 接收数据
   ↓
根据操作类型渲染：
├─ translate: ActionTranslate
└─ 其他: ActionGeneral
   ↓
执行 AI 调用或其他处理
   ↓
显示结果
```

### 7.4 配置同步流程

```
[UI Layer - Settings]
用户修改设置
   ↓
dispatch(setSelectionEnabled(value))
   ↓
window.api.selection.setEnabled(value)
   ↓
[IPC]
Selection_SetEnabled
   ↓
[Service Layer]
configManager.setSelectionAssistantEnabled(value)
   ↓
保存到配置文件
   ↓
触发订阅回调
   ↓
SelectionService 接收通知
   ↓
更新服务状态
   ↓
[Sync]
storeSyncService.syncToRenderer()
   ↓
[Redux]
同步到所有渲染进程
```

---

## 8. 实现方案对比

### 8.1 文本选择检测方案

#### 方案 A：Native Hook（当前方案）

**实现方式**：
- 使用 C++ 编写的 Node.js native addon
- 直接调用操作系统 API

**优点**：
- ✅ 性能最优
- ✅ 可获取精确的位置信息
- ✅ 支持任何应用程序
- ✅ 响应速度快

**缺点**：
- ❌ 开发和维护成本高
- ❌ 需要处理不同操作系统的差异
- ❌ macOS 需要辅助功能权限
- ❌ 可能存在安全风险

#### 方案 B：Clipboard Polling（轮询剪贴板）

**实现方式**：
```typescript
setInterval(() => {
  const text = clipboard.readText()
  if (text !== lastText) {
    // 检测到新文本
  }
}, 100)
```

**优点**：
- ✅ 实现简单
- ✅ 跨平台兼容性好
- ✅ 不需要特殊权限

**缺点**：
- ❌ 无法获取选择位置
- ❌ 无法区分用户主动选择和其他复制操作
- ❌ 轮询消耗资源
- ❌ 响应延迟

#### 方案 C：Accessibility API（无 Hook）

**实现方式**：
- 使用操作系统的辅助功能 API
- 监听文本选择事件

**优点**：
- ✅ 官方支持的方式
- ✅ 相对安全
- ✅ 可获取部分位置信息

**缺点**：
- ❌ 实现复杂度中等
- ❌ 兼容性问题较多
- ❌ 某些应用不支持
- ❌ macOS 需要辅助功能权限

#### 方案对比表

| 特性 | Native Hook | Clipboard Polling | Accessibility API |
|-----|------------|-------------------|-------------------|
| 性能 | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ |
| 位置精度 | ⭐⭐⭐⭐⭐ | ⭐ | ⭐⭐⭐ |
| 兼容性 | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |
| 开发难度 | ⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |
| 维护成本 | ⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |
| 用户体验 | ⭐⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐ |

**结论**：Cherry Studio 选择了 Native Hook 方案，因为它提供了最佳的性能和用户体验，尽管开发和维护成本较高。

### 8.2 窗口管理方案

#### 方案 A：预加载窗口池（当前方案）

**实现方式**：
```typescript
// 预先创建窗口
private preloadedActionWindows: BrowserWindow[] = []

// 使用时从池中获取
private popActionWindow(): BrowserWindow {
  return this.preloadedActionWindows.pop() || this.createPreloadedActionWindow()
}
```

**优点**：
- ✅ 响应速度极快（窗口已加载）
- ✅ 用户体验好
- ✅ 资源利用率高

**缺点**：
- ❌ 占用额外内存
- ❌ 实现复杂度略高

#### 方案 B：按需创建

**实现方式**：
```typescript
// 每次需要时创建新窗口
private createActionWindow(): BrowserWindow {
  const window = new BrowserWindow({...})
  window.loadFile('...')
  return window
}
```

**优点**：
- ✅ 内存占用小
- ✅ 实现简单

**缺点**：
- ❌ 首次显示有明显延迟
- ❌ 用户体验差

#### 方案 C：单例窗口复用

**实现方式**：
```typescript
// 只创建一个窗口，反复使用
private actionWindow: BrowserWindow | null = null
```

**优点**：
- ✅ 内存占用最小
- ✅ 管理简单

**缺点**：
- ❌ 无法同时打开多个操作窗口
- ❌ 功能受限

**结论**：当前方案平衡了性能和资源占用，适合大多数使用场景。

### 8.3 状态管理方案

#### 方案 A：Redux + IPC（当前方案）

**架构**：
```
Settings UI → Redux → IPC → ConfigManager → File System
              ↓                    ↓
         Other Windows ← IPC ←  Subscribe
```

**优点**：
- ✅ 状态一致性好
- ✅ 可追溯和调试
- ✅ 支持多窗口同步

**缺点**：
- ❌ 架构复杂
- ❌ IPC 通信开销

#### 方案 B：直接 IPC

**架构**：
```
Settings UI → IPC → ConfigManager → File System
```

**优点**：
- ✅ 简单直接
- ✅ 减少中间层

**缺点**：
- ❌ 难以管理复杂状态
- ❌ 多窗口同步困难

**结论**：Redux 方案适合需要多窗口协作的复杂应用。

---

## 9. 优缺点分析

### 9.1 整体优点

#### 9.1.1 用户体验

✅ **全局可用**
- 在任何应用程序中都可以使用
- 无需切换应用或窗口
- 真正的"随选随用"

✅ **快速响应**
- 预加载窗口机制确保瞬时响应
- 工具栏显示延迟 < 100ms
- 操作窗口打开延迟 < 50ms

✅ **智能定位**
- 自动适应屏幕边界
- 跟随选择位置
- 多显示器支持

✅ **非侵入式**
- 不影响原应用功能
- 透明窗口设计
- 自动隐藏机制

#### 9.1.2 功能性

✅ **灵活的触发模式**
- 选中即显示（适合频繁使用）
- Ctrl 键触发（避免误触发）
- 快捷键触发（手动控制）

✅ **应用过滤**
- 预定义黑名单（排除不兼容应用）
- 白名单模式（仅在指定应用启用）
- 黑名单模式（排除指定应用）

✅ **自定义操作**
- 支持内置操作（复制、搜索、翻译）
- 支持自定义 AI 助手
- 支持自定义搜索引擎

✅ **丰富的配置选项**
- 工具栏样式（紧凑/标准）
- 窗口行为（跟随/居中、自动关闭/置顶）
- 透明度调整
- 自定义 CSS

#### 9.1.3 技术实现

✅ **高性能**
- Native 模块实现核心功能
- 最小化 IPC 通信
- 异步处理不阻塞主线程

✅ **可维护性**
- 清晰的架构分层
- 单一职责原则
- 完善的类型定义

✅ **可扩展性**
- 插件化的操作机制
- Redux 状态管理支持复杂场景
- 预留扩展接口

### 9.2 整体缺点

#### 9.2.1 系统兼容性

❌ **平台限制**
- 仅支持 Windows 和 macOS
- Linux 不支持（selection-hook 限制）
- 不同操作系统表现有差异

❌ **权限要求**
- macOS 需要辅助功能权限
- 可能被安全软件拦截
- 企业环境可能受限

#### 9.2.2 应用兼容性

❌ **特定应用问题**
- PDF 阅读器（位置检测不准确）
- Office 应用（可能冲突）
- 远程桌面（无法使用）
- 某些游戏（被反作弊系统阻止）

❌ **精细调优需求**
- 不同应用需要不同的配置
- 维护精细调优列表成本高
- 新应用可能需要添加到黑名单

#### 9.2.3 技术挑战

❌ **Native 模块维护**
- 需要 C++ 开发能力
- 不同平台需要分别编译
- 升级 Electron 可能导致兼容性问题
- 调试困难

❌ **窗口管理复杂**
- macOS 全屏应用兼容性问题
- 窗口层级和焦点管理复杂
- 多显示器场景需要特殊处理

❌ **内存占用**
- 预加载窗口占用内存（~50-100MB）
- 多个操作窗口同时打开时内存增加
- 长时间运行可能有内存泄漏风险

#### 9.2.4 用户体验问题

❌ **学习曲线**
- 多种触发模式可能让用户困惑
- 配置选项较多
- 需要时间适应

❌ **误触发**
- 选中模式下容易误触发
- 在某些应用中可能干扰正常操作

❌ **性能影响**
- 持续的后台监听消耗 CPU
- 在低性能设备上可能有感知延迟

### 9.3 安全性考虑

#### 风险

❌ **隐私风险**
- 可以访问任何应用的选中文本
- 可能泄露敏感信息（密码、私人对话等）
- 需要明确的隐私政策

❌ **安全风险**
- 系统级 Hook 可能被恶意利用
- 需要确保代码安全性
- 定期安全审计

#### 缓解措施

✅ **透明度**
- 开源代码，可审计
- 明确告知用户功能和权限

✅ **用户控制**
- 用户可随时禁用
- 应用过滤机制
- 不上传或记录选中文本（除非用户主动操作）

---

## 10. 平台兼容性

### 10.1 Windows

**支持版本**：Windows 10 及以上

**特性支持**：
- ✅ 全功能支持
- ✅ 工具栏窗口类型：`toolbar`
- ✅ 不需要特殊权限
- ✅ 坐标系统：物理坐标，需要转换

**已知问题**：
- 某些应用（如 Excel）可能与工具栏冲突
- 远程桌面会话中无法使用
- 某些截图工具可能干扰检测

**优化措施**：
- 预定义黑名单排除不兼容应用
- 精细调优列表优化特定应用
- 坐标自动转换处理 DPI 缩放

### 10.2 macOS

**支持版本**：macOS 10.15 及以上

**特性支持**：
- ✅ 全功能支持
- ✅ 工具栏窗口类型：`panel`
- ✅ 全屏应用兼容
- ⚠️ 需要辅助功能权限

**已知问题**：
- 需要在"系统偏好设置 > 安全性与隐私 > 辅助功能"中授权
- 全屏应用中窗口管理复杂（Dock 图标可能消失）
- 焦点管理需要特殊处理（避免其他窗口被带到前面）

**优化措施**：
- 提供权限设置指引
- 特殊的焦点和层级管理逻辑
- `setVisibleOnAllWorkspaces` 支持全屏应用
- `showInactive()` 避免激活窗口

### 10.3 Linux

**支持状态**：❌ 不支持

**原因**：
- selection-hook 库不支持 Linux
- X11/Wayland 的文本选择 API 差异大
- 需要额外的开发和测试工作

**未来计划**：
- 可能在后续版本中添加支持
- 需要社区贡献或单独的 native 模块

### 10.4 平台差异处理

#### 检测平台

```typescript
import { isMac, isWin } from '@main/constant'

const isSupportedOS = isWin || isMac

if (!isSupportedOS) {
  // 不支持的平台
  return false
}
```

#### 平台特定代码

```typescript
// Windows 特定
if (isWin) {
  this.toolbarWindow = new BrowserWindow({
    type: 'toolbar',
    focusable: false
  })
}

// macOS 特定
if (isMac) {
  this.toolbarWindow = new BrowserWindow({
    type: 'panel',
    hiddenInMissionControl: true,
    acceptFirstMouse: true
  })
}
```

#### 配置文件差异

```typescript
// Windows 配置
SELECTION_PREDEFINED_BLACKLIST.WINDOWS: [
  'explorer.exe',
  'excel.exe',
  // ...
]

// macOS 配置
SELECTION_PREDEFINED_BLACKLIST.MAC: [
  'com.apple.finder'
]
```

---

## 11. 总结

### 11.1 核心优势

1. **真正的全局功能**：跨应用程序的文本选择和操作
2. **优秀的性能**：Native 实现 + 预加载机制
3. **灵活的配置**：多种触发模式和过滤选项
4. **良好的用户体验**：智能定位、快速响应、非侵入式

### 11.2 主要挑战

1. **平台兼容性**：不同操作系统需要特殊处理
2. **应用兼容性**：某些应用需要精细调优
3. **技术复杂度**：Native 模块开发和维护成本高
4. **安全和隐私**：需要明确的安全措施

### 11.3 适用场景

✅ **适合**：
- 需要频繁翻译的用户
- 需要快速查找信息的用户
- 需要 AI 辅助阅读的用户
- 跨应用协作的场景

❌ **不适合**：
- Linux 用户
- 不希望授予辅助功能权限的用户
- 低性能设备
- 企业受限环境

### 11.4 未来展望

可能的改进方向：

1. **扩展平台支持**
   - Linux 支持
   - 更好的 Windows 11 集成

2. **增强功能**
   - OCR 集成（图片文字识别）
   - 手写识别
   - 多语言实时翻译

3. **性能优化**
   - 减少内存占用
   - 优化响应速度
   - 降低 CPU 使用率

4. **改进兼容性**
   - 扩展应用支持
   - 更智能的应用检测
   - 自动化的精细调优

5. **用户体验**
   - 简化配置界面
   - 智能推荐操作
   - 更好的视觉反馈

---

## 12. 参考资料

### 12.1 相关文件

- `src/main/services/SelectionService.ts` - 核心服务实现
- `src/main/configs/SelectionConfig.ts` - 配置定义
- `src/renderer/src/windows/selection/toolbar/SelectionToolbar.tsx` - 工具栏 UI
- `src/renderer/src/windows/selection/action/SelectionActionApp.tsx` - 操作窗口 UI
- `src/renderer/src/hooks/useSelectionAssistant.ts` - React Hook
- `src/renderer/src/types/selectionTypes.d.ts` - 类型定义

### 12.2 依赖库

- [selection-hook](https://www.npmjs.com/package/selection-hook) - Native 文本选择监听模块
- [Electron](https://www.electronjs.org/) - 桌面应用框架
- [React](https://react.dev/) - UI 框架

### 12.3 相关 API 文档

- [Electron BrowserWindow](https://www.electronjs.org/docs/latest/api/browser-window)
- [Electron screen](https://www.electronjs.org/docs/latest/api/screen)
- [Windows Hooks API](https://docs.microsoft.com/en-us/windows/win32/winmsg/hooks)
- [macOS Accessibility API](https://developer.apple.com/documentation/accessibility)

---

**文档版本**：1.0  
**创建日期**：2025-11-23  
**作者**：Cherry Studio Development Team  
**维护状态**：Active
